# GLPI Plugin — Changelog

**Date:** 2026-05-18  
**Scope:** Full resource expansion · deep detail · sub-items · search · stats · entities · bulk · upload · validations · timeline · actors · profile switching

---

## Summary

| File | Changes |
|------|---------|
| `vector_plugin_glpi/client/glpi_api.py` | RESOURCE_MAP: 10 → 34 entries; `_WITH_PARAMS`; 15 new client methods |
| `vector_plugin_glpi/plugin.py` | plugin_capabilities: 10 → 34 entries |
| `vector_plugin_glpi/fetcher.py` | 20 new fetcher methods added across all feature areas |
| `vector_plugin_glpi/transform.py` | CHANGE_STATUS + PROBLEM_STATUS maps; labels for changes and problems |
| `vector_plugin_glpi/router.py` | Endpoints: 9 → 76 |

---

## Commit history (newest first)

### feat: profile switching, timeline, actor management
**Router: 61 → 76 routes**

#### Profile switching
- `GET /profiles` — list all GLPI profiles the current user can access
- `POST /profiles/active` — switch active profile by ID

```python
# POST /profiles/active
{"profile_id": 2}
```

#### Timeline — tickets, changes, problems, projects
Single call replaces 3 separate sub-item fetches. Items are merged and sorted chronologically. Each entry is tagged with a `type` field.

```
GET /tickets/{id}/timeline
GET /changes/{id}/timeline
GET /problems/{id}/timeline
GET /projects/{id}/timeline
```

Response shape:
```json
{
  "success": true,
  "data": [
    {"type": "followup", "id": 1, "content": "...", "date": "2026-05-01 10:00:00"},
    {"type": "task",     "id": 3, "content": "...", "date": "2026-05-02 09:30:00"},
    {"type": "solution", "id": 1, "content": "...", "date": "2026-05-03 14:00:00"}
  ],
  "total": 3
}
```

Implementation: `asyncio.gather` fetches followups + tasks + solutions in parallel, then sorts by `date_creation` or `date` field.

#### Actor management — tickets, changes, problems

| Endpoint | Description |
|---|---|
| `GET /{resource}/{id}/actors` | All users + groups with role labels |
| `POST /{resource}/{id}/actors` | Add a user or group |
| `DELETE /{resource}/{id}/actors/user/{record_id}` | Remove a user actor |
| `DELETE /{resource}/{id}/actors/group/{record_id}` | Remove a group actor |

Role values: `1=requester`, `2=assigned`, `3=observer`

```python
# POST /tickets/42/actors
{"actor_type": "user", "actor_id": 5, "role": 2}

# GET /tickets/42/actors response
{
  "users":  [{"id": 1, "users_id": 5, "type": 2, "role": "assigned", ...}],
  "groups": [{"id": 1, "groups_id": 3, "type": 2, "role": "assigned", ...}]
}
```

GLPI sub-itemtypes used:
- Tickets: `Ticket_User` / `Group_Ticket`
- Changes: `Change_User` / `Group_Change`
- Problems: `Problem_User` / `Group_Problem`

---

### feat: edit/delete sub-items, upload, who-am-I, validations, bulk, reservations
**Router: 33 → 61 routes**

#### Edit + Delete sub-items
Full PUT/DELETE added for every sub-item type:

| Resource | Followups | Tasks | Solutions |
|---|---|---|---|
| tickets | PUT + DELETE | PUT + DELETE | PUT + DELETE |
| changes | PUT + DELETE | PUT + DELETE | PUT + DELETE |
| problems | PUT + DELETE | PUT + DELETE | PUT + DELETE |
| projects | — | PUT + DELETE | — |

```
PUT  /tickets/{id}/followups/{followup_id}
DELETE /tickets/{id}/followups/{followup_id}
```

#### File / Document upload
```
POST /upload   (multipart/form-data)
```
Fields: `file` (required), `name`, `linked_itemtype`, `linked_items_id`

Example — attach a file to ticket 42:
```
POST /upload
file=@report.pdf
linked_itemtype=Ticket
linked_items_id=42
```

Implementation bypasses `_request()` (which sends `Content-Type: application/json`) and builds a raw httpx multipart call so the boundary header is set correctly.

#### Who am I — `GET /me`
```json
{
  "id": 2,
  "name": "admin",
  "realname": "Administrator",
  "email": "admin@example.com",
  "active_entity": 0,
  "active_entity_name": "Root entity",
  "active_profile": "Super-Admin",
  "profiles": [...]
}
```

Uses `asyncio.gather` on `getFullSession` + `getMyProfiles`.

#### Ticket validation workflow

| Endpoint | Action |
|---|---|
| `GET /tickets/{id}/validations` | List all validation requests |
| `POST /tickets/{id}/validations` | Request approval |
| `PUT /tickets/{id}/validations/{vid}` | Approve (`status: 3`) or reject (`status: 4`) |
| `DELETE /tickets/{id}/validations/{vid}` | Remove request |

```python
# POST /tickets/42/validations
{"users_id_validate": 5, "comment_submission": "Please approve"}

# PUT /tickets/42/validations/7  — approve
{"status": 3, "comment_validation": "Looks good"}
```

#### Bulk actions

```
POST /{resource}/bulk-update   body: {"items": [{"id": 1, "status": 5}, {"id": 2, "status": 5}]}
POST /{resource}/bulk-delete   body: {"ids": [1, 2, 3], "force_purge": false}
```

Both map to single GLPI API calls (`PUT /Itemtype` and `DELETE /Itemtype` with array input).

#### Reservations
`reservations` and `reservationitems` added to `RESOURCE_MAP` and `plugin_capabilities`. Full CRUD available automatically through the generic router.

---

### feat: stats, search-options, entities, POST sub-items, fix ITIL labels
**Router: 19 → 33 routes**

#### Fix status/priority labels for changes and problems
`transform.py` previously only applied labels to tickets. Changes and problems were returning raw integers.

Added:
```python
CHANGE_STATUS  = {1: "New", 2: "Assigned", 3: "Planned", ..., 13: "Cancelled"}
PROBLEM_STATUS = {1: "New", 2: "Assigned", ..., 8: "Observed"}
```
`map_items()` and `map_item()` now call the correct mapper for each resource type.

#### `GET /stats`
Returns total counts for 10 key resources + ticket breakdown by status and priority. All counts fetched in parallel via `asyncio.gather`.

```json
{
  "counts": {"tickets": 150, "changes": 23, "computers": 45, ...},
  "tickets": {
    "total": 150,
    "open": 87,
    "by_status":   {"New": 30, "Processing (assigned)": 45, "Solved": 20, ...},
    "by_priority": {"Medium": 60, "High": 40, ...}
  }
}
```

Ticket status counts use GLPI search engine with field `12` (Status). Priority uses field `52`.

#### `GET /{resource}/search-options`
Proxies `listSearchOptions/{Itemtype}` — returns every searchable field with its ID and name. Use these IDs in the `criteria` array of `POST /{resource}/search`.

#### `GET /entities` + `POST /entities/active`
```python
# POST /entities/active
{"entity_id": 3, "is_recursive": true}
```

#### POST sub-items (create from UI)
All 10 sub-item create endpoints — followups, tasks, solutions for tickets/changes/problems/projects:
```
POST /tickets/{id}/followups    body: {"content": "..."}
POST /tickets/{id}/tasks        body: {"content": "..."}
POST /tickets/{id}/solutions    body: {"content": "..."}
# same for /changes, /problems, /projects (tasks only)
```

Parent IDs are injected server-side from the URL — callers only need to send `content` and optional fields.

---

### feat: embed full sub-data on all single-item fetches

Added `_WITH_PARAMS` dict — maps every GLPI itemtype to its applicable `with_*` params. `get_item` now passes all of them in a single request.

| `with_*` param | Data embedded |
|---|---|
| `with_logs` | Full change history |
| `with_documents` | Attached files |
| `with_users` | All involved users (requester, assigned, observer) |
| `with_groups` | Assigned groups |
| `with_contracts` | Linked contracts |
| `with_infocoms` | Financial info (purchase date, warranty, price) |
| `with_devices` | Hardware components (Computer) |
| `with_softwares` | Installed software (Computer) |
| `with_networkports` | Network interfaces |
| `with_tickets` | Associated tickets (assets) |

List view intentionally unchanged — `_WITH_PARAMS` only applies to `get_item`.

---

### feat: RESOURCE_MAP 10 → 32 + sub-item GET endpoints + search endpoint

#### RESOURCE_MAP expansion (10 → 32)
22 new resources across 5 categories. Every addition gets full CRUD for free.

#### New GET sub-item endpoints
Changes, Problems and Projects previously had no sub-item endpoints. Added:
- `/changes/{id}/followups|tasks|solutions`
- `/problems/{id}/followups|tasks|solutions`
- `/projects/{id}/tasks`

Sub-itemtype accuracy: `ChangeTask` not `ITILTask`, `ProblemTask` not `ITILTask`, `KnowbaseItem` not `KnowledgeBase`, `ConsumableItem` not `Consumable`.

#### `POST /{resource}/search` exposed
`GLPIClient.search_items()` and `DataFetcher.search_items()` existed but had no HTTP endpoint. Now exposed.

```bash
POST /tickets/search
{
  "criteria": [{"field": 12, "searchtype": "equals", "value": "1"}],
  "forcedisplay": [1, 2, 12],
  "limit": 25
}
```

---

## Final endpoint inventory (76 total)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check |
| GET | `/stats` | Counts + ticket breakdowns |
| GET | `/entities` | Active + accessible entities |
| POST | `/entities/active` | Switch entity |
| GET | `/profiles` | Accessible profiles |
| POST | `/profiles/active` | Switch profile |
| GET | `/me` | Current user identity |
| POST | `/upload` | File/document upload |
| GET | `/{resource}` | List items |
| GET | `/{resource}/search-options` | GLPI field IDs |
| GET | `/{resource}/{id}` | Get single item (full detail) |
| POST | `/{resource}` | Create item |
| POST | `/{resource}/bulk-update` | Bulk update |
| POST | `/{resource}/bulk-delete` | Bulk delete |
| PUT | `/{resource}/{id}` | Update item |
| DELETE | `/{resource}/{id}` | Delete item |
| POST | `/{resource}/search` | Advanced search |
| GET | `/tickets/{id}/timeline` | Chronological feed |
| GET | `/tickets/{id}/actors` | All actors + roles |
| POST | `/tickets/{id}/actors` | Add actor |
| DELETE | `/tickets/{id}/actors/{type}/{rid}` | Remove actor |
| GET | `/tickets/{id}/validations` | Validation requests |
| POST | `/tickets/{id}/validations` | Request approval |
| PUT | `/tickets/{id}/validations/{vid}` | Approve/reject |
| DELETE | `/tickets/{id}/validations/{vid}` | Remove validation |
| GET/POST | `/tickets/{id}/followups` | List / create |
| PUT/DELETE | `/tickets/{id}/followups/{fid}` | Edit / remove |
| GET/POST | `/tickets/{id}/tasks` | List / create |
| PUT/DELETE | `/tickets/{id}/tasks/{tid}` | Edit / remove |
| GET/POST | `/tickets/{id}/solutions` | List / create |
| PUT/DELETE | `/tickets/{id}/solutions/{sid}` | Edit / remove |
| GET/POST/PUT/DELETE | `/changes/{id}/timeline\|actors\|followups\|tasks\|solutions` | Same as tickets |
| GET/POST/PUT/DELETE | `/problems/{id}/timeline\|actors\|followups\|tasks\|solutions` | Same as tickets |
| GET/POST/PUT/DELETE | `/projects/{id}/timeline\|tasks` | Timeline + task CRUD |
| GET | `/manifest` | Plugin metadata |
| GET | `/config-schema` | Config schema |

---

## 1. `vector_plugin_glpi/client/glpi_api.py`

### What changed
`RESOURCE_MAP` — the central dict that maps lowercase resource names to GLPI CamelCase itemtypes — was expanded from **10 to 32 entries**. Every entry added here automatically gets full CRUD (list / get / create / update / delete) through the generic router with zero additional code.

### Before (10 entries)
```python
RESOURCE_MAP: dict[str, str] = {
    "tickets":           "Ticket",
    "changes":           "Change",
    "problems":          "Problem",
    "computers":         "Computer",
    "printers":          "Printer",
    "networkequipments": "NetworkEquipment",
    "phones":            "Phone",
    "software":          "Software",
    "users":             "User",
    "groups":            "Group",
}
```

### After (32 entries)
```python
RESOURCE_MAP: dict[str, str] = {
    # ITSM
    "tickets":           "Ticket",
    "changes":           "Change",
    "problems":          "Problem",
    # Assets
    "computers":         "Computer",
    "monitors":          "Monitor",
    "printers":          "Printer",
    "networkequipments": "NetworkEquipment",
    "phones":            "Phone",
    "software":          "Software",
    "racks":             "Rack",
    "consumables":       "ConsumableItem",
    "cartridges":        "CartridgeItem",
    # Business
    "licenses":          "SoftwareLicense",
    "suppliers":         "Supplier",
    "contracts":         "Contract",
    "documents":         "Document",
    "budgets":           "Budget",
    # Knowledge & Projects
    "knowledgebase":     "KnowbaseItem",
    "projects":          "Project",
    "domains":           "Domain",
    "certificates":      "Certificate",
    "clusters":          "Cluster",
    # Directory
    "users":             "User",
    "groups":            "Group",
    "locations":         "Location",
    "manufacturers":     "Manufacturer",
    # Reference / lookup tables
    "itilcategories":    "ITILCategory",
    "slas":              "SLA",
    "olas":              "OLA",
    "requesttypes":      "RequestType",
    "solutiontypes":     "SolutionType",
    "taskcategories":    "TaskCategory",
}
```

### New resources added (22)

| Resource key | GLPI itemtype | Category |
|---|---|---|
| `monitors` | `Monitor` | Assets |
| `racks` | `Rack` | Assets |
| `consumables` | `ConsumableItem` | Assets |
| `cartridges` | `CartridgeItem` | Assets |
| `licenses` | `SoftwareLicense` | Business |
| `suppliers` | `Supplier` | Business |
| `contracts` | `Contract` | Business |
| `documents` | `Document` | Business |
| `budgets` | `Budget` | Business |
| `knowledgebase` | `KnowbaseItem` | Knowledge |
| `projects` | `Project` | Knowledge |
| `domains` | `Domain` | Knowledge |
| `certificates` | `Certificate` | Knowledge |
| `clusters` | `Cluster` | Knowledge |
| `locations` | `Location` | Directory |
| `manufacturers` | `Manufacturer` | Directory |
| `itilcategories` | `ITILCategory` | Reference |
| `slas` | `SLA` | Reference |
| `olas` | `OLA` | Reference |
| `requesttypes` | `RequestType` | Reference |
| `solutiontypes` | `SolutionType` | Reference |
| `taskcategories` | `TaskCategory` | Reference |

> **GLPI itemtype accuracy notes:**
> - `ConsumableItem` / `CartridgeItem` — using `Consumable` / `Cartridge` returns empty results from GLPI
> - `KnowbaseItem` — correct GLPI spelling (not `KnowledgeBase`)
> - `SoftwareLicense` — correct itemtype for licenses (not `License`)

---

### 1b. `_WITH_PARAMS` map added + `get_item` updated

**Problem before this change:** `get_item` only sent `expand_dropdowns=true`. GLPI omits all sub-objects (history, users, hardware, financials, etc.) unless explicitly requested via `with_*` query params. Every single-item fetch was returning a shallow record.

**Fix:** Added `_WITH_PARAMS` — a dict mapping every GLPI itemtype to the list of `with_*` params applicable to it. `get_item` now iterates this map and adds each param before making the request.

```python
_WITH_PARAMS: dict[str, list[str]] = {
    # ITSM
    "Ticket":           ["with_logs", "with_documents", "with_users", "with_groups", "with_contracts"],
    "Change":           ["with_logs", "with_documents", "with_users", "with_groups", "with_contracts"],
    "Problem":          ["with_logs", "with_documents", "with_users", "with_groups", "with_contracts"],
    # Assets
    "Computer":         ["with_logs", "with_infocoms", "with_contracts", "with_documents",
                         "with_tickets", "with_networkports", "with_devices", "with_softwares"],
    "Monitor":          ["with_logs", "with_infocoms", "with_contracts", "with_documents", "with_tickets"],
    "Printer":          ["with_logs", "with_infocoms", "with_contracts", "with_documents",
                         "with_tickets", "with_networkports"],
    "NetworkEquipment": ["with_logs", "with_infocoms", "with_contracts", "with_documents",
                         "with_tickets", "with_networkports"],
    "Phone":            ["with_logs", "with_infocoms", "with_contracts", "with_documents",
                         "with_tickets", "with_networkports"],
    "Software":         ["with_logs", "with_infocoms", "with_contracts", "with_documents", "with_tickets"],
    "Rack":             ["with_logs", "with_infocoms", "with_contracts", "with_documents", "with_tickets"],
    "ConsumableItem":   ["with_logs", "with_infocoms", "with_contracts", "with_documents"],
    "CartridgeItem":    ["with_logs", "with_infocoms", "with_contracts", "with_documents"],
    # Business
    "SoftwareLicense":  ["with_logs", "with_infocoms", "with_contracts", "with_documents"],
    "Supplier":         ["with_logs", "with_contracts", "with_documents"],
    "Contract":         ["with_logs", "with_documents"],
    "Document":         ["with_logs"],
    "Budget":           ["with_logs"],
    # Knowledge & Projects
    "KnowbaseItem":     ["with_logs", "with_documents"],
    "Project":          ["with_logs", "with_documents"],
    "Domain":           ["with_logs"],
    "Certificate":      ["with_logs"],
    "Cluster":          ["with_logs", "with_tickets"],
    # Directory
    "User":             ["with_logs", "with_groups"],
    "Group":            ["with_logs"],
    "Location":         ["with_logs"],
    "Manufacturer":     ["with_logs"],
}
```

**`get_item` before:**
```python
params = {
    "expand_dropdowns": int(expand_dropdowns),
    "get_hateoas": 0,
}
```

**`get_item` after:**
```python
params: dict[str, Any] = {
    "expand_dropdowns": int(expand_dropdowns),
    "get_hateoas": 0,
}
for key in _WITH_PARAMS.get(itemtype, []):
    params[key] = 1
```

**What each `with_*` param adds:**

| Param | Data embedded |
|---|---|
| `with_logs` | Full change history — every field edit, by whom, timestamp |
| `with_documents` | Attached files and documents |
| `with_users` | All involved users (requester, assigned tech, observer, validator) |
| `with_groups` | Assigned groups |
| `with_contracts` | Linked contracts |
| `with_infocoms` | Financial info — purchase date, warranty end, price, supplier, order number |
| `with_devices` | Hardware components (CPU, RAM, drives, GPU) — Computer only |
| `with_softwares` | Installed software list — Computer only |
| `with_networkports` | Network interfaces and IPs |
| `with_tickets` | Associated tickets — for assets |

> **List view is intentionally unchanged.** `list_items` does not use `_WITH_PARAMS` — loading all sub-objects for 50 rows at once would be too slow. Deep data is only fetched on single-item (`GET /{resource}/{id}`) calls.

---

## 2. `vector_plugin_glpi/plugin.py`

### What changed
`plugin_capabilities` updated to match `RESOURCE_MAP` exactly — **10 → 32 entries**. This list is what the Vector SDK advertises to the backend on plugin registration.

### Before
```python
plugin_capabilities = [
    "tickets", "changes", "problems",
    "computers", "printers", "networkequipments",
    "phones", "software",
    "users", "groups",
]
```

### After
```python
plugin_capabilities = [
    # ITSM
    "tickets", "changes", "problems",
    # Assets
    "computers", "monitors", "printers", "networkequipments",
    "phones", "software", "racks", "consumables", "cartridges",
    # Business
    "licenses", "suppliers", "contracts", "documents", "budgets",
    # Knowledge & Projects
    "knowledgebase", "projects", "domains", "certificates", "clusters",
    # Directory
    "users", "groups", "locations", "manufacturers",
    # Reference / lookup tables
    "itilcategories", "slas", "olas",
    "requesttypes", "solutiontypes", "taskcategories",
]
```

---

## 3. `vector_plugin_glpi/router.py`

### What changed
**10 new endpoints added** — the `POST /{resource}/search` endpoint (previously built in client/fetcher but never exposed via HTTP) plus 9 sub-item endpoints for Changes, Problems, and Projects. Tickets already had sub-item endpoints; the others had none.

All new sub-item endpoints reuse the existing `fetcher.fetch_sub_items()` method — no new fetcher or client code required.

---

### 3a. Search endpoint

**Before:** `GLPIClient.search_items()` and `DataFetcher.search_items()` were fully implemented but unreachable — no HTTP endpoint exposed them.

**After:**
```python
@router.post("/{resource}/search", response_model=GLPIListResponse)
async def search_items(
    resource: str,
    payload: dict[str, Any],
    fetcher: DataFetcher = Depends(get_fetcher),
) -> GLPIListResponse:
    resource = _validate_resource(resource)
    criteria     = payload.get("criteria")
    forcedisplay = payload.get("forcedisplay")
    limit        = int(payload.get("limit", 50))
    offset       = int(payload.get("offset", 0))
    try:
        items, total = await fetcher.search_items(
            resource,
            criteria=criteria,
            forcedisplay=forcedisplay,
            limit=limit,
            offset=offset,
        )
        return GLPIListResponse(success=True, data=items, total=total, limit=limit, offset=offset)
    except Exception as exc:
        _handle_glpi_exc(exc)
```

**Example request:**
```bash
POST /api/v1/plugins/glpi/tickets/search
{
  "criteria": [
    {"field": 12, "searchtype": "equals", "value": "1"},
    {"link": "AND", "field": 10, "searchtype": "contains", "value": "network"}
  ],
  "forcedisplay": [1, 2, 12, 10],
  "limit": 25,
  "offset": 0
}
```

---

### 3b. Change sub-item endpoints (3 new)

**Before:** No sub-item endpoints for Changes.

```python
@router.get("/changes/{change_id}/followups")
async def get_change_followups(change_id: int, limit: int = Query(50, ge=1, le=200), offset: int = Query(0, ge=0), fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("changes", change_id, "ITILFollowup", limit=limit, offset=offset)
    return {"success": True, "data": items, "total": len(items)}

@router.get("/changes/{change_id}/tasks")
async def get_change_tasks(change_id: int, limit: int = Query(50, ge=1, le=200), offset: int = Query(0, ge=0), fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("changes", change_id, "ChangeTask", limit=limit, offset=offset)
    return {"success": True, "data": items, "total": len(items)}

@router.get("/changes/{change_id}/solutions")
async def get_change_solutions(change_id: int, fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("changes", change_id, "ITILSolution")
    return {"success": True, "data": items, "total": len(items)}
```

> Change tasks use `ChangeTask` — not `ITILTask`. Wrong name returns empty results silently.

---

### 3c. Problem sub-item endpoints (3 new)

**Before:** No sub-item endpoints for Problems.

```python
@router.get("/problems/{problem_id}/followups")
async def get_problem_followups(problem_id: int, limit: int = Query(50, ge=1, le=200), offset: int = Query(0, ge=0), fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("problems", problem_id, "ITILFollowup", limit=limit, offset=offset)
    return {"success": True, "data": items, "total": len(items)}

@router.get("/problems/{problem_id}/tasks")
async def get_problem_tasks(problem_id: int, limit: int = Query(50, ge=1, le=200), offset: int = Query(0, ge=0), fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("problems", problem_id, "ProblemTask", limit=limit, offset=offset)
    return {"success": True, "data": items, "total": len(items)}

@router.get("/problems/{problem_id}/solutions")
async def get_problem_solutions(problem_id: int, fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("problems", problem_id, "ITILSolution")
    return {"success": True, "data": items, "total": len(items)}
```

> Problem tasks use `ProblemTask` — not `ITILTask`.

---

### 3d. Project sub-item endpoints (1 new)

**Before:** No sub-item endpoints for Projects.

```python
@router.get("/projects/{project_id}/tasks")
async def get_project_tasks(project_id: int, limit: int = Query(50, ge=1, le=200), offset: int = Query(0, ge=0), fetcher=Depends(get_fetcher)):
    items = await fetcher.fetch_sub_items("projects", project_id, "ProjectTask", limit=limit, offset=offset)
    return {"success": True, "data": items, "total": len(items)}
```

---

### Complete endpoint inventory

| Method | Path | Description |
|--------|------|-------------|
| GET | `/plugins/glpi/health` | Health check |
| GET | `/plugins/glpi/{resource}` | List items (paginated) |
| GET | `/plugins/glpi/{resource}/{id}` | Get single item |
| POST | `/plugins/glpi/{resource}` | Create item |
| PUT | `/plugins/glpi/{resource}/{id}` | Update item |
| DELETE | `/plugins/glpi/{resource}/{id}` | Delete item |
| **POST** | **`/plugins/glpi/{resource}/search`** | **Advanced search (NEW)** |
| GET | `/plugins/glpi/tickets/{id}/followups` | Ticket followups |
| GET | `/plugins/glpi/tickets/{id}/tasks` | Ticket tasks |
| GET | `/plugins/glpi/tickets/{id}/solutions` | Ticket solutions |
| **GET** | **`/plugins/glpi/changes/{id}/followups`** | **Change followups (NEW)** |
| **GET** | **`/plugins/glpi/changes/{id}/tasks`** | **Change tasks (NEW)** |
| **GET** | **`/plugins/glpi/changes/{id}/solutions`** | **Change solutions (NEW)** |
| **GET** | **`/plugins/glpi/problems/{id}/followups`** | **Problem followups (NEW)** |
| **GET** | **`/plugins/glpi/problems/{id}/tasks`** | **Problem tasks (NEW)** |
| **GET** | **`/plugins/glpi/problems/{id}/solutions`** | **Problem solutions (NEW)** |
| **GET** | **`/plugins/glpi/projects/{id}/tasks`** | **Project tasks (NEW)** |
| GET | `/plugins/glpi/manifest` | Plugin metadata |
| GET | `/plugins/glpi/config-schema` | Config schema |
