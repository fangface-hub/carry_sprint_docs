# CarrySprint Software Method Design

## 1. Purpose

This document defines software methods that realize the requirements in `docs/requirements/software_requirements_specification.md`.
This document defines inter-process interfaces.
This document defines process input/output.
This document defines process conditions, process behaviors.

## 2. Scope

Target runtime structure:

- Client: Web browser
- Server Process A: Web Gateway Process
- Server Process B: Application Process
- Data Store: SQLite per project file

Target functional scope:

- Top page retrieval
- Project selection
- Project registration
- Sprint workspace retrieval
- Task update
- Resource settings retrieval/update
- Working-day calendar retrieval/update
- Carry-over review apply
- User menu visibility retrieval/update
- User registration/modification/deletion
- Initial admin user bootstrap
- Project role assignment
- Default locale resolution by explicit user setting
- Default locale resolution by client language/region
- User locale setting retrieval/update

## 3. Process Model

### 3.1 Process List

| Process ID | Process Name | Responsibility |
| --- | --- | --- |
| P1 | Web Gateway Process | HTTP endpoint, request validation, ZeroMQ relay, HTTP response mapping |
| P2 | Application Process | Domain logic execution, persistence access, response payload composition |

### 3.2 Data Store Access Rule

- P1 never accesses SQLite directly.
- P2 handles all SQLite read/write operations.
- One project uses one SQLite file with format `project_<id>.sqlite`.

### 3.3 Data Flow Diagram

The following DFD defines process-level data flow among browser, P1, P2, SQLite.

```plantuml
!include ./diagrams/dfd/dfd_level1.puml
```

### 3.4 MVC Responsibility Mapping

This section defines MVC boundaries across P1 and P2 in method design.

| MVC Role | Process/Interface Element | Responsibility |
| --- | --- | --- |
| Controller | P2 command dispatcher and use case handlers (Section 5.2, Section 6) | Execute command-level control flow and domain decision order |
| Model | P2 persistence access rules plus IF-DB-01 (Section 3.2, Section 4.3) | Manage domain state and SQLite read/write operations |
| View | P1 HTTP response mapping plus IF-HTTP-01 response schema (Section 5.1, Section 4.1) | Build external response representation for browser clients |

Boundary rules:

- P1 does not execute domain decision logic.
- P2 does not expose SQLite internals to P1.
- P2 returns domain result payload. P1 maps domain result payload to HTTP response output.

## 4. Inter-Process Interfaces

### 4.1 Interface IF-HTTP-01 (Browser -> P1)

Protocol:

- HTTP/1.1 over HTTPS
- HTTP/2 over HTTPS

Content type and response format rules:

- Browser UI routes (`/`, `/projects`, `/users`, project screen routes) return `text/html` browser UI shell.
- API routes (`/api/...`) return `application/json` with the common response schema.

Request header rules:

- Browser UI routes do not require `X-Request-Id`.
- API routes require `X-Request-Id`.
- API write requests require `Content-Type: application/json`.

Response body common schema:

```json
{
  "request_id": "string",
  "result": "ok|error",
  "data": {},
  "error": {
    "code": "string",
    "message": "string"
  }
}
```

Browser UI route contracts:

| UI | Route | Route State | Browser Behavior |
| --- | --- | --- | --- |
| Top Page | / | none | Open top page menu |
| Project Select Screen | /projects | none | Open project selection screen |
| Project Register Screen | /projects/new | none | Open project registration screen |
| Sprint Workspace Screen | /projects/{project_id}/sprints/{sprint_id}/workspace | required path params: `project_id`, `sprint_id` | Open sprint workspace for one project plus one sprint |
| Resource Settings Screen | /projects/{project_id}/resources | required path param: `project_id` | Open resource settings for one project |
| Working-Day Calendar Screen | /projects/{project_id}/calendar | required path param: `project_id` | Open working-day calendar for one project |
| User Management Screen | /users | none | Open user management screen |
| Carry-Over Review Dialog | /projects/{project_id}/sprints/{sprint_id}/workspace?dialog=carryover | required path params: `project_id`, `sprint_id`; required query param: `dialog=carryover` | Open sprint workspace plus carry-over review dialog |

Browser UI route rules:

- P1 shall accept stable browser UI routes for direct access.
- P1 shall resolve project context from `project_id` path parameter.
- P1 shall resolve sprint context from `sprint_id` path parameter.
- P1 shall open the carry-over review dialog on the sprint workspace route.
- P1 shall resolve the carry-over review dialog state from query parameter `dialog=carryover`.

Browser UI API integration acceptance criteria:

- Top Page (`/`) shall call API-10 to resolve selectable users.
- Top Page (`/`) shall call API-16 to resolve effective locale.
- Top Page (`/`) shall call API-17 to resolve enabled menu buttons.
- Top Page (`/`) shall call API-18/API-19 to read/write user menu visibility.
- Top Page (`/`) shall call API-21/API-22 to read/write user locale settings.
- Project Select Screen (`/projects`) shall call API-01 and may call API-02 for selected project detail.
- Project Register Screen (`/projects/new`) shall call API-10 for administrator candidate list and shall call API-20 for project registration.
- Sprint Workspace Screen (`/projects/{project_id}/sprints/{sprint_id}/workspace`) shall call API-03 and may call API-04 for task update actions.
- Carry-Over Review Dialog route shall call API-03 and API-09.
- Resource Settings Screen shall call API-05 and API-06.
- Working-Day Calendar Screen shall call API-07 and API-08.
- User Management Screen (`/users`) shall call API-10/API-11/API-12/API-13 and API-14/API-15.
- A screen is considered integrated when it renders data from its required APIs and applies user write actions through the corresponding write APIs.

Endpoint contracts:

| API ID | Method | Path | Input Body | Output Data |
| --- | --- | --- | --- | --- |
| API-01 | GET | /api/projects | none | project list |
| API-02 | GET | /api/projects/{project_id}/summary | none | project summary |
| API-03 | GET | /api/projects/{project_id}/sprints/{sprint_id}/workspace | none | budget-in tasks, budget-out tasks, totals |
| API-04 | PATCH | /api/projects/{project_id}/tasks/{task_id} | estimate, impact, status fields | updated task |
| API-05 | GET | /api/projects/{project_id}/resources | none | resource list |
| API-06 | PUT | /api/projects/{project_id}/resources | full resource list | saved resource list |
| API-07 | GET | /api/projects/{project_id}/calendar | optional month range | calendar days |
| API-08 | PUT | /api/projects/{project_id}/calendar | calendar day list | saved calendar days |
| API-09 | POST | /api/projects/{project_id}/sprints/{sprint_id}/carryover/apply | carry-over decision list | applied task list |
| API-10 | GET | /api/users | none | user list |
| API-11 | POST | /api/users | user fields | registered user |
| API-12 | PATCH | /api/users/{user_id} | user fields | updated user |
| API-13 | DELETE | /api/users/{user_id} | none | none |
| API-14 | GET | /api/projects/{project_id}/roles | none | project role list |
| API-15 | PUT | /api/projects/{project_id}/roles | role assignment list | saved role assignment list |
| API-16 | GET | /api/locales/default | none | resolved default locale |
| API-17 | GET | /api/top/menu | none | enabled menu button list for signed-in user |
| API-18 | GET | /api/users/{user_id}/menu-visibility | none | user menu visibility settings |
| API-19 | PUT | /api/users/{user_id}/menu-visibility | menu visibility setting list | saved user menu visibility settings |
| API-20 | POST | /api/projects | project registration fields | registered project plus initial sprint |
| API-21 | GET | /api/users/{user_id}/locale | none | user locale setting |
| API-22 | PUT | /api/users/{user_id}/locale | locale setting | saved user locale setting |

### 4.2 Interface IF-ZMQ-01 (P1 -> P2)

Protocol:

- ZeroMQ REQ/REP
- UTF-8 JSON payload

Message request schema:

```json
{
  "request_id": "string",
  "command": "string",
  "project_id": "string",
  "path_params": {},
  "query_params": {},
  "payload": {}
}
```

Message response schema:

```json
{
  "request_id": "string",
  "status": "ok|error",
  "data": {},
  "error": {
    "code": "string",
    "message": "string"
  }
}
```

Command mapping:

| Command | Source API |
| --- | --- |
| `list_projects` | API-01 |
| `get_project_summary` | API-02 |
| `get_sprint_workspace` | API-03 |
| `update_task` | API-04 |
| `list_resources` | API-05 |
| `save_resources` | API-06 |
| `get_calendar` | API-07 |
| `save_calendar` | API-08 |
| `apply_carryover` | API-09 |
| `list_users` | API-10 |
| `register_user` | API-11 |
| `update_user` | API-12 |
| `delete_user` | API-13 |
| `get_project_roles` | API-14 |
| `save_project_roles` | API-15 |
| `resolve_default_locale` | API-16 |
| `get_top_menu` | API-17 |
| `get_user_menu_visibility` | API-18 |
| `save_user_menu_visibility` | API-19 |
| `create_project` | API-20 |
| `get_user_locale_setting` | API-21 |
| `save_user_locale_setting` | API-22 |

### 4.3 Interface IF-DB-01 (P2 -> SQLite)

Access mode:

- SQL read for retrieval APIs
- SQL transaction for update APIs

Primary tables:

- `projects`
- `sprints`
- `tasks`
- `resources`
- `working_day_calendar`
- `task_resource_allocations`
- `users`
- `project_roles`
- `user_menu_visibility`
- `user_locale_settings`

## 5. Process Input/Output

### 5.1 P1 Web Gateway Process

Inputs:

- HTTP request line
- HTTP headers
- JSON request body

Outputs:

- ZeroMQ request message to P2
- HTTP response to browser

P1 input validation rules:

- Condition: required path parameter is missing. Behavior: return `400` with `INVALID_PATH_PARAM`.
- Condition: JSON body parsing fails. Behavior: return `400` with `INVALID_JSON`.
- Condition: unknown route arrives. Behavior: return `404` with `ROUTE_NOT_FOUND`.

P1 response mapping rules:

- Condition: P2 status is `ok`. Behavior: return `200` with data payload.
- Condition: P2 returns domain validation error. Behavior: return `422` with domain error code.
- Condition: P2 timeout happens. Behavior: return `504` with `UPSTREAM_TIMEOUT`.
- Condition: P2 transport error happens. Behavior: return `502` with `UPSTREAM_UNAVAILABLE`.

### 5.2 P2 Application Process

Inputs:

- ZeroMQ command envelope
- Path parameter values
- Query parameter values
- JSON payload values

Outputs:

- SQL read queries
- SQL write transactions
- ZeroMQ response envelope

P2 command dispatch rules:

- Condition: command value matches defined command. Behavior: execute mapped use case handler.
- Condition: command value does not match defined command. Behavior: return error `UNKNOWN_COMMAND`.

P2 persistence rules:

- Condition: write command starts. Behavior: open transaction.
- Condition: all SQL statements finish successfully. Behavior: commit transaction.
- Condition: any SQL statement fails. Behavior: rollback transaction, return `PERSISTENCE_ERROR`.

### 5.3 P2 Startup Initialization Process

Inputs:

- Application start event

Outputs:

- SQL read query for bootstrap check
- SQL write transaction for bootstrap insert

Startup initialization rules:

- Condition: user row with user_id `admin` does not exist. Behavior: insert user row with user_id `admin`, initial password value `admin`, set created_at.
- Condition: user row with user_id `admin` exists. Behavior: skip bootstrap insert.
- Condition: bootstrap SQL operation fails. Behavior: stop P2 startup, return `INITIAL_USER_BOOTSTRAP_FAILED`.

## 6. Use Case Conditions/Behaviors

### 6.1 UC-01 List Projects

- Condition: request has valid header `X-Request-Id`. Behavior: P1 sends `list_projects` to P2.
- Condition: P2 finds project rows. Behavior: return sorted project list.
- Condition: P2 finds no rows. Behavior: return empty list.

### 6.2 UC-02 Get Sprint Workspace

- Condition: project exists, sprint exists. Behavior: read tasks in sprint scope, classify budget-in tasks, budget-out tasks.
- Condition: task estimated hours is null. Behavior: classify task as budget-out with flag `needs_input=true`.
- Condition: task impact value is null. Behavior: classify task as budget-out with flag `needs_input=true`.
- Condition: sprint record does not exist. Behavior: return `SPRINT_NOT_FOUND`.

Classification behavior:

- Condition: cumulative forecast hours <= available hours. Behavior: place task in budget-in list.
- Condition: cumulative forecast hours > available hours. Behavior: place task in budget-out list.
- Condition: task status is in progress. Behavior: keep status flag in output item.

### 6.3 UC-03 Update Task

- Condition: target task exists. Behavior: update mutable fields, set `updated_at`.
- Condition: estimate value < 0. Behavior: return `INVALID_ESTIMATE`.
- Condition: impact value not in allowed set. Behavior: return `INVALID_IMPACT`.

### 6.4 UC-04 Save Resources

- Condition: payload passes schema check. Behavior: replace resource set in one transaction.
- Condition: duplicated resource_id appears. Behavior: return `DUPLICATE_RESOURCE_ID`.
- Condition: capacity_hours_per_day <= 0. Behavior: return `INVALID_RESOURCE_CAPACITY`.

### 6.5 UC-05 Save Calendar

- Condition: date format is valid ISO date. Behavior: upsert calendar row for each date.
- Condition: duplicated date appears in payload. Behavior: return `DUPLICATE_CALENDAR_DATE`.
- Condition: payload month range is missing for query. Behavior: use current month.

### 6.6 UC-06 Apply Carry-Over

- Condition: task belongs to current sprint, decision is carry-over, input target exists. Behavior: move sprint_id to next sprint.
- Condition: task belongs to current sprint, decision is carry-over, input target does not exist. Behavior: clear sprint_id.
- Condition: task is marked as keep. Behavior: keep sprint_id unchanged.
- Condition: target next sprint does not exist. Behavior: return `TARGET_SPRINT_NOT_FOUND`.

### 6.7 UC-07 List Users

- Condition: request has valid header `X-Request-Id`. Behavior: P1 sends `list_users` to P2.
- Condition: P2 finds user rows. Behavior: return sorted user list.
- Condition: P2 finds no rows. Behavior: return empty list.

### 6.8 UC-08 Register User

- Condition: user_id does not exist in `users` table. Behavior: insert user row, set `created_at`.
- Condition: user_id already exists. Behavior: return `DUPLICATE_USER_ID`.

### 6.9 UC-09 Update User

- Condition: target user exists. Behavior: update mutable user fields, set `updated_at`.
- Condition: target user does not exist. Behavior: return `USER_NOT_FOUND`.

### 6.10 UC-10 Delete User

- Condition: target user exists. Behavior: delete user row plus all associated project role rows in one transaction.
- Condition: target user does not exist. Behavior: return `USER_NOT_FOUND`.

### 6.11 UC-11 Get Project Roles

- Condition: project exists. Behavior: read all role assignment rows for the project. Return role list.
- Condition: project does not exist. Behavior: return `PROJECT_NOT_FOUND`.

### 6.12 UC-12 Save Project Roles

- Condition: payload passes schema check. Behavior: replace project role set in one transaction.
- Condition: role value is not in allowed set (`administrator`, `assignee`). Behavior: return `INVALID_ROLE`.
- Condition: user_id in payload does not exist in `users` table. Behavior: return `USER_NOT_FOUND`.
- Condition: project does not exist. Behavior: return `PROJECT_NOT_FOUND`.

### 6.13 UC-13 Resolve Default Locale

- Condition: explicit user locale setting exists. Behavior: return explicit locale.
- Condition: locale configuration contains an entry matching client language plus client region. Behavior: return matched locale.
- Condition: locale configuration contains no language plus region entry. Behavior: evaluate client region only.
- Condition: locale configuration contains an entry matching client region. Behavior: return region-matched locale.
- Condition: locale configuration contains no region entry. Behavior: evaluate client language only.
- Condition: locale configuration contains an entry matching client language. Behavior: return language-matched locale.
- Condition: no locale rule matches request context. Behavior: return fallback locale `en`.

### 6.17 UC-17 Get User Locale Setting

- Condition: target user exists. Behavior: return explicit locale setting when it exists.
- Condition: target user has no explicit locale setting. Behavior: return empty locale value.
- Condition: target user does not exist. Behavior: return `USER_NOT_FOUND`.

### 6.18 UC-18 Save User Locale Setting

- Condition: locale value is empty string. Behavior: clear explicit locale setting.
- Condition: locale value is in allowed locale set. Behavior: upsert explicit locale setting.
- Condition: locale value is not in allowed locale set. Behavior: return `INVALID_LOCALE`.
- Condition: target user does not exist. Behavior: return `USER_NOT_FOUND`.

### 6.14 UC-14 Get Top Menu

- Condition: signed-in user is resolved from request context. Behavior: return enabled menu button list from user menu visibility setting.
- Condition: user menu visibility setting row does not exist. Behavior: return default enabled menu list.

Menu key to browser UI route mapping:

| Menu Key | Browser UI Route |
| --- | --- |
| project_select | /projects |
| sprint_workspace | /projects/{project_id}/sprints/{sprint_id}/workspace |
| resource_settings | /projects/{project_id}/resources |
| calendar_settings | /projects/{project_id}/calendar |
| user_management | /users |

Default enabled menu list:

- Project Select
- Sprint Workspace
- Resource Settings
- Working-Day Calendar

### 6.15 UC-15 Get User Menu Visibility

- Condition: target user exists. Behavior: return target user menu visibility settings.
- Condition: target user does not exist. Behavior: return `USER_NOT_FOUND`.

### 6.16 UC-16 Save User Menu Visibility

- Condition: target user exists, payload menu keys are valid. Behavior: replace target user menu visibility settings in one transaction.
- Condition: menu key is not in allowed set. Behavior: return `INVALID_MENU_KEY`.
- Condition: duplicated menu key appears in payload. Behavior: return `DUPLICATE_MENU_KEY`.
- Condition: target user does not exist. Behavior: return `USER_NOT_FOUND`.

Allowed menu key set:

- project_select
- sprint_workspace
- resource_settings
- calendar_settings
- user_management

### 6.17 UC-17 Register Project

- Condition: project_id does not exist in `projects` table. Behavior: insert project row in `system.sqlite`, create `project_{id}.sqlite`, insert initial sprint row, insert initial administrator role row.
- Condition: project_id already exists in `projects` table. Behavior: return `DUPLICATE_PROJECT_ID`.
- Condition: initial sprint start_date is after end_date. Behavior: return `INVALID_SPRINT_DATE_RANGE`.
- Condition: administrator user_id does not exist in `users` table. Behavior: return `USER_NOT_FOUND`.

### 6.18 Activity Diagrams

The following activity diagram defines UC-02 control flow with condition branches.

```plantuml
!include ./diagrams/activity/activity_uc02_get_sprint_workspace.puml
```

The following activity diagram defines UC-06 control flow with transaction/rollback branches.

```plantuml
!include ./diagrams/activity/activity_uc06_apply_carryover.puml
```

The following activity diagram defines UC-14 control flow for signed-in user top menu resolution.

```plantuml
!include ./diagrams/activity/activity_uc14_get_top_menu.puml
```

The following activity diagram defines UC-16 control flow for user menu visibility update.

```plantuml
!include ./diagrams/activity/activity_uc16_save_user_menu_visibility.puml
```

The following activity diagram defines UC-17 control flow for project registration.

```plantuml
!include ./diagrams/activity/activity_uc17_register_project.puml
```

The following activity diagram defines P2 startup initialization flow for initial admin user bootstrap.

```plantuml
!include ./diagrams/activity/activity_startup_initialize_admin_user.puml
```

## 7. Error Model

Error response rules:

- Error payload always includes `request_id`, `error.code`, `error.message`.
- Error code naming uses uppercase snake case.
- Error message uses deterministic wording.

Error code table:

| Code | Meaning | HTTP Status |
| --- | --- | --- |
| INVALID_PATH_PARAM | Required path parameter missing or malformed | 400 |
| INVALID_JSON | JSON parse failed | 400 |
| ROUTE_NOT_FOUND | Undefined API route | 404 |
| UNKNOWN_COMMAND | Undefined ZeroMQ command | 400 |
| SPRINT_NOT_FOUND | Sprint row not found | 404 |
| INVALID_ESTIMATE | Estimate value invalid | 422 |
| INVALID_IMPACT | Impact value invalid | 422 |
| DUPLICATE_RESOURCE_ID | Resource identifier duplicated | 422 |
| INVALID_RESOURCE_CAPACITY | Resource capacity value invalid | 422 |
| DUPLICATE_CALENDAR_DATE | Calendar date duplicated in input | 422 |
| TARGET_SPRINT_NOT_FOUND | Carry-over target sprint missing | 404 |
| PROJECT_NOT_FOUND | Project row not found | 404 |
| USER_NOT_FOUND | User row not found | 404 |
| DUPLICATE_USER_ID | User identifier duplicated | 422 |
| INVALID_ROLE | Role value not in allowed set | 422 |
| INVALID_MENU_KEY | Menu key value not in allowed set | 422 |
| DUPLICATE_MENU_KEY | Menu key duplicated in input | 422 |
| DUPLICATE_PROJECT_ID | Project identifier duplicated | 422 |
| INVALID_SPRINT_DATE_RANGE | Initial sprint period is invalid | 422 |
| PERSISTENCE_ERROR | SQLite read failure, SQLite write failure | 500 |
| INITIAL_USER_BOOTSTRAP_FAILED | Initial admin user bootstrap failed at startup | 500 |
| UPSTREAM_UNAVAILABLE | ZeroMQ transport failure | 502 |
| UPSTREAM_TIMEOUT | ZeroMQ timeout | 504 |

## 8. Timeouts/Retries

- P1 timeout for IF-ZMQ-01 uses 3000 ms.
- P1 performs no automatic retry for write commands.
- P1 performs one retry for read commands after timeout.
- P2 SQL busy timeout uses 2000 ms.

## 9. Requirements Traceability Matrix

This table traces each requirement in `docs/requirements/software_requirements_specification.md` to the design element in this document that realizes it.

### 9.1 System/Software Architecture

| Requirement ID | SRS Section | Requirement Summary | Method Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-SYS-01 | §1 System Architecture | ZeroMQ-based inter-process communication between client and server | §4.2 IF-ZMQ-01 | ZeroMQ REQ/REP protocol, JSON request/response schema, command mapping |
| SRS-SYS-02 | §1 System Architecture | SQLite-based per-project storage | §3.2 Data Store Access Rule, §4.3 IF-DB-01 | P2-only SQLite access rule, SQL read/write access mode |
| SRS-SYS-03 | §2 Software Architecture | API gateway as HTTP entry point with request routing boundary | §4.1 IF-HTTP-01, §5.1 P1 | HTTPS endpoint contracts, P1 input validation rules, P1 response mapping rules |
| SRS-SYS-04 | §2 Software Architecture | Application component for MVC-based domain logic | §3.4 MVC Responsibility Mapping, §5.2 P2, §6.1 to 6.17 | MVC role boundaries, P2 command dispatch rules, use case handlers UC-01 through UC-17 |

### 9.2 Database

| Requirement ID | SRS Section | Requirement Summary | Method Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-DB-01 | §3.1 Database Policy | Use SQLite as the database engine | §4.3 IF-DB-01 | SQL read/transaction access modes |
| SRS-DB-02 | §3.1 Database Policy | Isolate data by project with one SQLite file per project | §3.2 Data Store Access Rule | One project uses one SQLite file rule |
| SRS-DB-03 | §3.1 Database Policy | Use `project_{id}.sqlite` as the standard file naming format | §3.2 Data Store Access Rule | File format `project_<id>.sqlite` |
| SRS-DB-04 | §3.1 Database Policy | Manage resource information plus working-day calendar in SQLite | §4.3 IF-DB-01 | Primary tables `resources`, `working_day_calendar` |
| SRS-DB-05 | §3.1 Database Policy | Manage user information plus project role assignments in SQLite | §4.3 IF-DB-01 | Primary tables `users`, `project_roles` |

### 9.3 Screen Design/UI Policy

| Requirement ID | SRS Section | Requirement Summary | Method Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-UI-01 | §4.1 Screen Design Policy | Run the client in a web browser | §4.1 IF-HTTP-01 | HTTP/1.1 over HTTPS browser interface, HTTP/2 over HTTPS browser interface |
| SRS-UI-02 | §4.1 Screen Design Policy | Require no additional software installation on the client side | §4.1 IF-HTTP-01 | Browser-only HTTPS interface; no client-side process required |
| SRS-UI-03 | §4.1 Screen Design Policy | Exclude progress-monitoring UI | §4.1 IF-HTTP-01 | No monitoring API endpoint defined |
| SRS-UI-04 | §4.1 Screen Design Policy | Center screen layout on budget-in/budget-out visualization | §6.2 UC-02 | Task classification logic by cumulative forecast hours vs. available hours |
| SRS-UI-05 | §4.2 Common UI Requirements | Implement UI structure plus elements defined in this specification | §4.1 IF-HTTP-01 | Endpoint contracts cover all defined screens |
| SRS-UI-06 | §4.2 Common UI Requirements | Display budget-in, budget-out, impact, estimated hours with consistent visual rules | §4.1 API-03, §6.2 UC-02 | Sprint workspace response payload includes budget-in list, budget-out list, totals |
| SRS-UI-07 | §4.2 Common UI Requirements | Do not display monitoring metrics | §4.1 IF-HTTP-01 | No monitoring data field included in any response payload |
| SRS-UI-08 | §4.2 Common UI Requirements | Make primary operations executable within three clicks | §4.1 API-01, API-02, API-03 | Project list, project summary, sprint workspace accessible via single API call each |
| SRS-UI-09 | §4.2 Common UI Requirements | Represent states with labels, icons, not color alone | - | Client-side rendering concern; no server-side method design element |
| SRS-UI-10 | §4.2 Common UI Requirements | Provide a top page as a major screen | §4.1 API-17, §6.14 UC-14 | `get_top_menu` command provides top page menu model |
| SRS-UI-11 | §4.2 Common UI Requirements | Display top-page menu entries as buttons | §4.1 API-17, §6.14 UC-14 | Enabled menu item list for button rendering |
| SRS-UI-12 | §4.2 Common UI Requirements | Support menu visibility settings per user | §4.1 API-18, API-19, §6.15, §6.16 | User menu visibility retrieval and update commands |
| SRS-UI-13 | §4.2 Common UI Requirements | Display only enabled menu buttons for the signed-in user | §4.1 API-17, §6.14 UC-14 | Signed-in user scoped menu filtering behavior |
| SRS-UI-14 | §4.2.1 UI Placement URL | Place each major UI at the defined stable URL | §4.1 IF-HTTP-01, §6.14 UC-14 | Browser UI route contracts, menu key to browser UI route mapping |

### 9.4 Screen-specific Requirements

| Requirement ID | SRS Section | Requirement Summary | Method Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-SC-01 | §4.3.1 Project Select Screen | Display project search plus project list plus project summary | §6.1 UC-01, §4.1 API-01, API-02 | `list_projects` command, `get_project_summary` command |
| SRS-SC-02 | §4.3.2 Top Page Screen | Display menu area plus menu visibility settings area | §6.14, §6.15, §6.16, §4.1 API-17 to API-19 | `get_top_menu`, `get_user_menu_visibility`, `save_user_menu_visibility` commands |
| SRS-SC-03 | §4.3.3 Sprint Workspace Screen | Display budget-in tasks plus budget-out tasks plus task editing area | §6.2 UC-02, §6.3 UC-03, §4.1 API-03, API-04 | `get_sprint_workspace` command with classification behavior, `update_task` command |
| SRS-SC-04 | §4.3.4 Resource Settings Screen | Display resource table plus resource edit form | §6.4 UC-04, §4.1 API-05, API-06 | `list_resources` command, `save_resources` command |
| SRS-SC-05 | §4.3.5 Working-Day Calendar Screen | Display calendar grid plus calendar control area | §6.5 UC-05, §4.1 API-07, API-08 | `get_calendar` command, `save_calendar` command |
| SRS-SC-06 | §4.3.6 Carry-Over Review Dialog | Display deferred task review plus carry-over decision input | §6.6 UC-06, §4.1 API-09 | `apply_carryover` command with keep/move decision handling |
| SRS-SC-07 | §4.3.7 User Management Screen | Display user list plus user edit form plus project role assignment area | §6.7 to 6.12, §4.1 API-10 to API-15 | User CRUD commands, `get_project_roles` command, `save_project_roles` command |
| SRS-SC-08 | §4.3.8 Project Register Screen | Display project registration inputs plus initial sprint inputs plus administrator assignment input | §6.17 UC-17, §4.1 API-20 | `create_project` command with initial sprint and administrator setup |

### 9.5 User Management Requirements

| Requirement ID | SRS Section | Requirement Summary | Method Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-UM-00 | §5.0 Initial User Requirements | Prepare initial user `admin` with initial password `admin` | §5.3 P2 Startup Initialization Process | Startup bootstrap rule inserts initial user when missing |
| SRS-UM-01 | §5.1 User Operation Requirements | Register a user from the user management screen | §6.8 UC-08, §4.1 API-11 | `register_user` command with duplicate check |
| SRS-UM-02 | §5.1 User Operation Requirements | Modify registered user information from the user management screen | §6.9 UC-09, §4.1 API-12 | `update_user` command with existence check |
| SRS-UM-03 | §5.1 User Operation Requirements | Delete a registered user from the user management screen | §6.10 UC-10, §4.1 API-13 | `delete_user` command with cascade role deletion in one transaction |
| SRS-UM-04 | §5.2 Project Role Requirements | Assign an administrator role to a user for a specified project | §6.12 UC-12, §4.1 API-15 | `save_project_roles` command with role value `administrator` |
| SRS-UM-05 | §5.2 Project Role Requirements | Assign an assignee role to a user for a specified project | §6.12 UC-12, §4.1 API-15 | `save_project_roles` command with role value `assignee` |

### 9.6 Internationalization Requirements

| Requirement ID | SRS Section | Requirement Summary | Method Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-I18N-01 | §6.1 Default Locale Resolution | Change default locale according to client language/region | §6.13 UC-13, §4.1 API-16 | `resolve_default_locale` command with locale matching behavior |
| SRS-I18N-02 | §6.1 Default Locale Resolution | Use `en` when locale configuration is not prepared | §6.13 UC-13 | fallback locale `en` |

## 9. Traceability to Requirements

| Requirement Area | Method Design Coverage |
| --- | --- |
| System architecture with browser, server, IPC, SQLite | Section 2, Section 4 |
| Process-level data flow definition | Section 3.3 |
| ZeroMQ request/response path | Section 4.2, Section 8 |
| Per-project SQLite policy | Section 3.2, Section 4.3 |
| Resource information handling | API-05, API-06, UC-04 |
| Working-day calendar handling | API-07, API-08, UC-05 |
| Budget-in/budget-out UI support | API-03, UC-02, Section 6.13 |
| Carry-over decision support | API-09, UC-06, Section 6.13 |
| Top page menu button support | API-17, UC-14 |
| User-specific menu visibility control | API-18, API-19, UC-15, UC-16 |
| User registration/modification/deletion | API-10, API-11, API-12, API-13, UC-07 through UC-10 |
| Project registration | API-20, UC-17 |
| Initial admin user bootstrap | Section 5.3 |
| Project role assignment | API-14, API-15, UC-11, UC-12 |
| Default locale resolution | API-16, UC-13 |

## 10. Implementation Constraints

- P1 implementation language uses Go.
- P2 implementation language uses Go.
- P1 keeps route handlers stateless.
- P2 keeps domain handlers deterministic for same input.
- Domain handlers avoid hidden side effects outside SQLite transaction boundary.
