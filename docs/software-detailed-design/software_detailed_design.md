# CarrySprint Software Detailed Design

## 1. Purpose

This document defines the internal structure of P1 (Web Gateway Process), P2 (Application Process) based on `docs/software-method-design/software_method_design.md`.
This document defines package structure.
This document defines Go struct definitions.
This document defines database DDL.
This document defines P1 handler processing steps.
This document defines P2 use case handler processing steps.
This document defines SQL query design.

## 2. Package Structure

### 2.1 P1 Package Structure

```text
p1/
  main.go                   Entry point. Wires HTTP transport and ZMQ gateway.
  transport/
    http/
      router.go             Registers browser UI routes. Registers API routes. Resolves screen shell metadata.
      ui_shell.tmpl         Shared HTML shell template for browser UI routes.
      ui_app.js             Browser UI renderer. Switches route to screen-specific rendering logic.
      middleware/
        request_id.go       Validates X-Request-Id.
      handler/
        projects.go         API-01, API-02, API-20 handlers
        workspace.go        API-03 handler
        tasks.go            API-04 handler
        resources.go        API-05, API-06 handlers
        calendar.go         API-07, API-08 handlers
        carryover.go        API-09 handler
        users.go            API-10, API-11, API-12, API-13 handlers
        roles.go            API-14, API-15 handlers
        locale.go           API-16, API-21, API-22 handlers
        top.go              API-17 handler
        menu_visibility.go  API-18, API-19 handlers
      presenter/
        response.go         Maps domain result from P2 into HTTP response schema.
  gateway/
    zmq/
      client.go             ZMQ REQ wrapper.
      codec.go              JSON encode/decode for IF-ZMQ-01.
  shared/
    model/
      zmq.go                ZMQRequest, ZMQResponse, ZMQError.
      http.go               HTTPResponse schema.
    apperror/
      code.go               P1-level transport and mapping error codes.
```

P1 module intent:

- `transport/http/handler` handles protocol input validation only.
- `transport/http/presenter` handles protocol output mapping only.
- `gateway/zmq` handles inter-process communication only.
- `shared` stores contracts shared by P1 internal modules.

### 2.2 P2 Package Structure

```text
p2/
  main.go                   Entry point. Wires ZMQ adapter, application use cases, repositories.
  adapter/
    zmq/
      server.go             ZMQ REP listener.
      dispatcher.go         Routes command to use case executor.
  application/
    usecase/
      projects.go           ListProjects, GetProjectSummary, CreateProject
      workspace.go          GetSprintWorkspace
      tasks.go              UpdateTask
      resources.go          ListResources, SaveResources
      calendar.go           GetCalendar, SaveCalendar
      carryover.go          ApplyCarryover
      users.go              ListUsers, RegisterUser, UpdateUser, DeleteUser
      roles.go              GetProjectRoles, SaveProjectRoles
      locale.go             ResolveDefaultLocale, GetUserLocaleSetting, SaveUserLocaleSetting
      top_menu.go           GetTopMenu
      menu_visibility.go    GetUserMenuVisibility, SaveUserMenuVisibility
      bootstrap.go          InitializeAdminUser
    validator/
      projects.go           Input rule checks for UC-17
      tasks.go              Input rule checks for UC-03
      resources.go          Input rule checks for UC-04
      calendar.go           Input rule checks for UC-05
      carryover.go          Input rule checks for UC-06
      users.go              Input rule checks for UC-08 to UC-10
      roles.go              Input rule checks for UC-11, UC-12
      locale.go             Input rule checks for UC-13, UC-18
      menu_visibility.go    Input rule checks for UC-16
    presenter/
      response.go           Builds command response payload.
  domain/
    model/
      entity.go             Domain entities.
      command.go            Command parameter models.
    service/
      workspace_classifier.go  Budget-in and budget-out classification policy.
      carryover_policy.go      Carry-over decision policy.
    repository/
      interface.go          Repository interfaces used by use cases.
  infrastructure/
    sqlite/
      db.go                 OpenSystemDB and OpenProjectDB.
      query/
        projects.sql.go     Query set for projects.
        sprints.sql.go      Query set for sprints.
        tasks.sql.go        Query set for tasks.
        resources.sql.go    Query set for resources.
        calendar.sql.go     Query set for calendar.
        carryover.sql.go    Query set for carry-over.
        users.sql.go        Query set for users.
        roles.sql.go        Query set for project roles.
        locale.sql.go       Query set for locale config.
        menu_visibility.sql.go Query set for user menu visibility.
        bootstrap.sql.go    Query set for initial admin bootstrap.
    repository/
      projects_repo.go      Implementation of projects repository interface.
      tasks_repo.go         Implementation of tasks repository interface.
      resources_repo.go     Implementation of resources repository interface.
      calendar_repo.go      Implementation of calendar repository interface.
      carryover_repo.go     Implementation of carry-over repository interface.
      users_repo.go         Implementation of users repository interface.
      roles_repo.go         Implementation of roles repository interface.
      locale_repo.go        Implementation of locale repository interface.
      menu_visibility_repo.go Implementation of menu visibility repository interface.
      bootstrap_repo.go     Implementation of startup bootstrap repository interface.
  shared/
    apperror/
      code.go               Domain error code constants.
```

P2 module intent:

- `adapter/zmq` isolates transport protocol concerns.
- `application/usecase` orchestrates use case flow.
- `application/validator` isolates validation rules from orchestration.
- `domain/service` centralizes business policy logic.
- `domain/repository/interface.go` defines persistence ports.
- `infrastructure/repository` plus `infrastructure/sqlite` provide persistence adapters.

### 2.3 MVC Responsibility Mapping

This section defines the MVC role boundaries in the detailed design.

| MVC Role | Package | Responsibility |
| --- | --- | --- |
| Controller | `p2/adapter/zmq/dispatcher.go`, `p2/application/usecase/*` | Accept command input from P1, execute use case flow, orchestrate domain processing order |
| Model | `p2/domain/*`, `p2/infrastructure/repository/*`, `p2/infrastructure/sqlite/*` | Define domain model and business policy, execute persistence read/write against SQLite |
| View | `p1/transport/http/presenter/response.go`, `p1/shared/model/http.go` | Define HTTP response schema, map domain result to HTTP response body |

Rules:

- P1 does not execute domain decision logic. P1 validates transport-level input and maps protocol responses.
- P2 is the application component that performs MVC-based domain logic.
- Domain layer depends on no transport module.
- Application layer depends on domain interfaces and domain services.
- Infrastructure layer depends on domain repository interfaces.
- P2 does not generate HTTP response format directly. P1 constructs HTTP response output.

## 3. Domain Entity Struct Definitions

### 3.1 Project

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| ProjectID | string | project_id | No | Primary key |
| Name | string | name | No | Project name |
| Description | string | description | No | Project description |
| CreatedAt | time.Time | created_at | No | Creation timestamp (RFC3339) |
| UpdatedAt | time.Time | updated_at | No | Last update timestamp (RFC3339) |

### 3.2 Sprint

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| SprintID | string | sprint_id | No | Primary key |
| ProjectID | string | project_id | No | Foreign key to projects |
| Name | string | name | No | Sprint name |
| StartDate | string | start_date | No | ISO 8601 date (e.g. "2026-04-01") |
| EndDate | string | end_date | No | ISO 8601 date |
| AvailableHours | float64 | available_hours | No | Total available work hours in sprint |
| CreatedAt | time.Time | created_at | No | Creation timestamp (RFC3339) |
| UpdatedAt | time.Time | updated_at | No | Last update timestamp (RFC3339) |

### 3.3 Task

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| TaskID | string | task_id | No | Primary key |
| ProjectID | string | project_id | No | Foreign key to projects |
| SprintID | \*string | sprint_id | Yes | Foreign key to sprints; nil if unassigned |
| Title | string | title | No | Task title |
| EstimateHours | \*float64 | estimate_hours | Yes | Estimated work hours; nil if not yet set |
| Impact | \*string | impact | Yes | Impact level: "high", "medium", "low"; nil if not yet set |
| Status | string | status | No | Task status: "todo", "in_progress", "done" |
| CreatedAt | time.Time | created_at | No | Creation timestamp (RFC3339) |
| UpdatedAt | time.Time | updated_at | No | Last update timestamp (RFC3339) |

### 3.4 Resource

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| ResourceID | string | resource_id | No | Primary key |
| Name | string | name | No | Resource display name |
| CapacityHoursPerDay | float64 | capacity_hours_per_day | No | Available work hours per day (> 0) |

### 3.5 WorkingDayCalendar

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| Date | string | date | No | ISO 8601 date (e.g. "2026-04-01") |
| IsWorking | bool | is_working | No | true if the date is a working day |

### 3.6 TaskResourceAllocation

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| TaskID | string | task_id | No | Foreign key to tasks |
| ResourceID | string | resource_id | No | Foreign key to resources |
| Hours | float64 | hours | No | Allocated hours |

### 3.7 User

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| UserID | string | user_id | No | Primary key |
| Name | string | name | No | User display name |
| Email | string | email | No | User email address; unique across all users |
| CreatedAt | time.Time | created_at | No | Creation timestamp (RFC3339) |
| UpdatedAt | time.Time | updated_at | No | Last update timestamp (RFC3339) |

### 3.8 ProjectRole

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| ProjectID | string | project_id | No | Foreign key to projects |
| UserID | string | user_id | No | Reference to users.user_id |
| Role | string | role | No | Allowed values: "administrator", "assignee" |

### 3.9 UserMenuVisibility

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| UserID | string | user_id | No | Reference to users.user_id |
| MenuKey | string | menu_key | No | Menu identifier key |
| IsEnabled | bool | is_enabled | No | true if menu button is enabled for the user |

### 3.10 UserLocaleSetting

| Field | Go Type | JSON Key | Nullable | Description |
| --- | --- | --- | --- | --- |
| UserID | string | user_id | No | Reference to users.user_id |
| Locale | string | locale | No | Explicit locale setting. Empty string means automatic selection. |

## 4. ZMQ Message Struct Definitions

### 4.1 ZMQRequest

| Field | Go Type | JSON Key | Description |
| --- | --- | --- | --- |
| RequestID | string | request_id | Forwarded from X-Request-Id header |
| Command | string | command | Command name matching IF-ZMQ-01 command mapping |
| ProjectID | string | project_id | Project identifier; empty string if not applicable |
| PathParams | map[string]string | path_params | URL path parameter key-value pairs |
| QueryParams | map[string]string | query_params | URL query parameter key-value pairs |
| Payload | json.RawMessage | payload | Request body JSON bytes; null if no body |

### 4.2 ZMQResponse

| Field | Go Type | JSON Key | Description |
| --- | --- | --- | --- |
| RequestID | string | request_id | Echoed from ZMQRequest.RequestID |
| Status | string | status | "ok", "error" |
| Data | json.RawMessage | data | Response payload bytes; null on error |
| Error | \*ZMQError | error | Error detail object; null on success |

### 4.3 ZMQError

| Field | Go Type | JSON Key | Description |
| --- | --- | --- | --- |
| Code | string | code | Error code in uppercase snake case |
| Message | string | message | Deterministic human-readable message |

## 5. API Request/Response Payload Schemas

### 5.0 Browser UI Routes

P1 browser UI route registration schema:

| UI | Browser Route | Required Route State | P1 Route Action | Upstream API Dependency |
| --- | --- | --- | --- | --- |
| Top Page | / | none | Serve top page shell | API-10, API-16, API-17, API-18, API-19, API-21, API-22 |
| Project Select Screen | /projects | none | Serve project select shell | API-01, API-02 |
| Project Register Screen | /projects/new | none | Serve project register shell | API-10, API-20 |
| Sprint Workspace Screen | /projects/{project_id}/sprints/{sprint_id}/workspace | path params: `project_id`, `sprint_id` | Serve sprint workspace shell | API-03, API-04 |
| Resource Settings Screen | /projects/{project_id}/resources | path param: `project_id` | Serve resource settings shell | API-05, API-06 |
| Working-Day Calendar Screen | /projects/{project_id}/calendar | path param: `project_id` | Serve calendar settings shell | API-07, API-08 |
| User Management Screen | /users | none | Serve user management shell | API-10, API-11, API-12, API-13, API-14, API-15 |
| Carry-Over Review Dialog | /projects/{project_id}/sprints/{sprint_id}/workspace?dialog=carryover | path params: `project_id`, `sprint_id`; query param: `dialog=carryover` | Serve sprint workspace shell. Open carry-over dialog state. | API-03, API-09 |

P1 browser UI route rules:

- `router.go` shall register stable browser UI routes.
- `router.go` shall register API routes under `/api`.
- `router.go` shall resolve `project_id` from browser route path parameters.
- `router.go` shall resolve `sprint_id` from browser route path parameters.
- `router.go` shall detect carry-over dialog state from query parameter `dialog=carryover`.
- Browser UI routes shall return `200 OK` with `Content-Type: text/html` and browser UI shell content.

Browser UI API integration acceptance criteria:

- Top Page (`/`) shall integrate API-10 for user selection.
- Top Page (`/`) shall integrate API-16 for effective locale resolution.
- Top Page (`/`) shall integrate API-17 for menu rendering.
- Top Page (`/`) shall integrate API-18 and API-19 for menu visibility operations.
- Top Page (`/`) shall integrate API-21 and API-22 for locale setting operations.
- Project Select Screen (`/projects`) shall integrate API-01 and API-02.
- Project Register Screen (`/projects/new`) shall integrate API-10 and API-20.
- Sprint Workspace Screen (`/projects/{project_id}/sprints/{sprint_id}/workspace`) shall integrate API-03 and API-04.
- Carry-Over Review Dialog route shall integrate API-03 and API-09.
- Resource Settings Screen shall integrate API-05 and API-06.
- Working-Day Calendar Screen shall integrate API-07 and API-08.
- User Management Screen (`/users`) shall integrate API-10, API-11, API-12, API-13, API-14, API-15.
- Integration completion for each screen requires both data rendering from read APIs and write-path execution through corresponding write APIs.

### 5.1 API-01 GET /api/projects

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| projects | array | List of project objects sorted by name ascending |
| projects[].project_id | string | Project identifier |
| projects[].name | string | Project name |
| projects[].description | string | Project description |
| projects[].updated_at | string | RFC3339 timestamp |

### 5.2 API-02 GET /api/projects/{project_id}/summary

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| project_id | string | Project identifier |
| name | string | Project name |
| description | string | Project description |
| sprint_count | integer | Total number of sprints in the project |
| task_count | integer | Total number of tasks in the project |
| updated_at | string | RFC3339 timestamp |

### 5.3 API-03 GET /api/projects/{project_id}/sprints/{sprint_id}/workspace

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| sprint_id | string | Sprint identifier |
| sprint_name | string | Sprint name |
| available_hours | number | Total available hours in the sprint |
| budget_in | array | Tasks classified within budget |
| budget_out | array | Tasks classified outside budget |
| totals.budget_in_hours | number | Sum of estimate_hours for all budget-in tasks |
| totals.budget_out_hours | number | Sum of estimate_hours for all budget-out tasks |

Task item fields (common to budget_in/budget_out):

| Field | Type | Description |
| --- | --- | --- |
| task_id | string | Task identifier |
| title | string | Task title |
| estimate_hours | number/null | Estimated hours; null if not set |
| impact | string/null | "high", "medium", "low"; null if not set |
| status | string | "todo", "in_progress", "done" |
| needs_input | boolean | true when estimate_hours is null. true when impact is null. |

### 5.4 API-04 PATCH /api/projects/{project_id}/tasks/{task_id}

Request body fields (all optional; at least one field required):

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| estimate_hours | number | >= 0 | Updated estimated hours |
| impact | string | "high", "medium", "low" | Updated impact level |
| status | string | "todo", "in_progress", "done" | Updated task status |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| task_id | string | Task identifier |
| estimate_hours | number/null | Updated estimated hours |
| impact | string/null | Updated impact level |
| status | string | Updated status |
| updated_at | string | RFC3339 timestamp |

### 5.5 API-05 GET /api/projects/{project_id}/resources

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| resources | array | List of resource objects |
| resources[].resource_id | string | Resource identifier |
| resources[].name | string | Resource name |
| resources[].capacity_hours_per_day | number | Daily capacity |

### 5.6 API-06 PUT /api/projects/{project_id}/resources

Request body fields:

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| resources | array | Required | Full resource list to replace existing set |
| resources[].resource_id | string | Required | Resource identifier; must be unique in the list |
| resources[].name | string | Required | Resource name |
| resources[].capacity_hours_per_day | number | > 0 | Daily capacity |

Response `data` field: same structure as request body.

### 5.7 API-07 GET /api/projects/{project_id}/calendar

Query parameters:

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| month | string | No | ISO 8601 year-month (e.g. "2026-04"); defaults to current month if omitted |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| month | string | Year-month of the returned data (e.g. "2026-04") |
| days | array | List of calendar day objects |
| days[].date | string | ISO 8601 date |
| days[].is_working | boolean | true if working day |

### 5.8 API-08 PUT /api/projects/{project_id}/calendar

Request body fields:

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| days | array | Required | List of calendar day objects to upsert |
| days[].date | string | Valid ISO 8601 date; unique in list | Calendar date |
| days[].is_working | boolean | Required | true if working day |

Response `data` field: same structure as request body.

### 5.9 API-09 POST /api/projects/{project_id}/sprints/{sprint_id}/carryover/apply

Request body fields:

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| decisions | array | Required | List of carry-over decision objects |
| decisions[].task_id | string | Required | Task identifier |
| decisions[].action | string | "carryover", "keep" | Decision for the task |
| decisions[].target_sprint_id | string/null | Required when action is "carryover" | Target sprint to move the task into |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| applied | array | List of result objects for each processed task |
| applied[].task_id | string | Task identifier |
| applied[].action | string | Applied action ("carryover", "keep") |
| applied[].sprint_id | string/null | Resulting sprint_id after the operation |

### 5.10 API-10 GET /api/users

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| users | array | List of user objects sorted by user_id ascending |
| users[].user_id | string | User identifier |
| users[].name | string | User display name |
| users[].email | string | User email address |

### 5.11 API-11 POST /api/users

Request body fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| user_id | string | Yes | User identifier; must be unique |
| name | string | Yes | User display name |
| email | string | Yes | User email address; must be unique |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| user_id | string | Registered user identifier |
| name | string | User display name |
| email | string | User email address |
| created_at | string | RFC3339 timestamp |

### 5.12 API-12 PATCH /api/users/{user_id}

Request body fields (all optional; at least one field required):

| Field | Type | Description |
| --- | --- | --- |
| name | string | Updated user display name |
| email | string | Updated user email address |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| user_id | string | User identifier |
| name | string | Updated display name |
| email | string | Updated email address |
| updated_at | string | RFC3339 timestamp |

### 5.13 API-13 DELETE /api/users/{user_id}

Request: No body.

Response: `result: "ok"`, no `data` field.

### 5.14 API-14 GET /api/projects/{project_id}/roles

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| roles | array | List of role assignment objects sorted by user_id ascending |
| roles[].user_id | string | User identifier |
| roles[].role | string | "administrator", "assignee" |

### 5.15 API-15 PUT /api/projects/{project_id}/roles

Request body fields:

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| roles | array | Required | Full role assignment list to replace existing set |
| roles[].user_id | string | Must exist in users table | User identifier |
| roles[].role | string | "administrator", "assignee" | Assigned role |

Response `data` field: same structure as request body.

### 5.16 API-16 GET /api/locales/default

Request: No body.

Request header:

| Header | Required | Description |
| --- | --- | --- |
| Accept-Language | No | Client language/region preference. Example: `ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7` |

Accept-Language parsing rules:

- The header value is split by `,` into language tags.
- Each tag is optionally followed by `;q=<value>`. The `q` value is a float in range [0.0, 1.0]. Default `q` is 1.0 when omitted.
- Tags are sorted in descending order of `q` value before evaluation.
- P2 shall return explicit user locale when `X-User-Id` resolves to a saved locale setting.
- P2 shall evaluate language plus region match first.
- P2 shall evaluate region-only match after language plus region lookup fails.
- P2 shall evaluate language-only match after region-only lookup fails.
- If no tag matches any `locale_config` entry, P2 returns `fallback`.
- If the header is absent or empty, P2 returns `fallback`.

Supported locale values in `locale_config`: `de`, `fr`, `it`, `ja`, `zh`.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| locale | string | Resolved default locale |
| source | string | `user_setting`, `matched`, `region_matched`, `language_matched`, `fallback` |

### 5.17 API-17 GET /api/top/menu

Request: No body.

Request header:

| Header | Required | Description |
| --- | --- | --- |
| X-User-Id | Yes | Signed-in user identifier |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| user_id | string | Signed-in user identifier |
| menu_buttons | array | Enabled menu list for the signed-in user |
| menu_buttons[].menu_key | string | `project_select`, `sprint_workspace`, `resource_settings`, `calendar_settings`, `user_management` |
| menu_buttons[].label | string | UI display label |

### 5.18 API-18 GET /api/users/{user_id}/menu-visibility

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| user_id | string | Target user identifier |
| menu_visibility | array | User menu visibility settings |
| menu_visibility[].menu_key | string | `project_select`, `sprint_workspace`, `resource_settings`, `calendar_settings`, `user_management` |
| menu_visibility[].is_enabled | boolean | true if enabled |

### 5.19 API-19 PUT /api/users/{user_id}/menu-visibility

Request body fields:

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| menu_visibility | array | Required | Full menu visibility setting list to replace existing set |
| menu_visibility[].menu_key | string | Must be in allowed key set | Menu identifier key |
| menu_visibility[].is_enabled | boolean | Required | Enable flag for the menu key |

Response `data` field: same structure as request body plus `user_id`.

### 5.20 API-20 POST /api/projects

Request body fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| project_id | string | Yes | Project identifier; must be unique |
| name | string | Yes | Project display name |
| description | string | Yes | Project description |
| initial_sprint | object | Yes | Initial sprint definition |
| initial_sprint.sprint_id | string | Yes | Initial sprint identifier |
| initial_sprint.name | string | Yes | Initial sprint name |
| initial_sprint.start_date | string | Yes | ISO 8601 date |
| initial_sprint.end_date | string | Yes | ISO 8601 date |
| initial_admin_user_id | string | Yes | Existing user identifier for administrator assignment |

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| project_id | string | Registered project identifier |
| name | string | Registered project name |
| description | string | Registered project description |
| initial_sprint.sprint_id | string | Registered initial sprint identifier |
| initial_sprint.name | string | Registered initial sprint name |
| initial_admin_user_id | string | Assigned administrator user identifier |
| created_at | string | RFC3339 timestamp |

### 5.21 API-21 GET /api/users/{user_id}/locale

Request: No body.

Response `data` field:

| Field | Type | Description |
| --- | --- | --- |
| user_id | string | Target user identifier |
| locale | string | Explicit locale setting. Empty string means automatic selection. |
| locale_options | array | Allowed locale values for explicit selection: `en`, `de`, `fr`, `it`, `ja`, `zh` |

### 5.22 API-22 PUT /api/users/{user_id}/locale

Request body fields:

| Field | Type | Constraint | Description |
| --- | --- | --- | --- |
| locale | string | Empty string or one of `en`, `de`, `fr`, `it`, `ja`, `zh` | Explicit locale setting |

Response `data` field: same structure as API-21.

## 6. Database Schema

### 6.1 Storage Topology

| File | Scope | Tables |
| --- | --- | --- |
| `system.sqlite` | System-level; one file per deployment | `users`, `projects`, `user_credentials`, `user_menu_visibility`, `user_locale_settings`, `locale_config` |
| `project_{id}.sqlite` | Project-level; one file per project | `sprints`, `tasks`, `resources`, `working_day_calendar`, `task_resource_allocations`, `project_roles` |

- P2 opens `system.sqlite` at startup. P2 holds a single long-lived connection.
- P2 opens `project_{id}.sqlite` on the first access for a given `project_id`. P2 caches the connection by project identifier.
- `project_roles.user_id` references `users.user_id` in `system.sqlite`. No SQLite foreign key constraint crosses database files. P2 validates `user_id` existence in `system.sqlite` before inserting into `project_roles`.

### 6.2 Table: users (system.sqlite)

```sql
CREATE TABLE IF NOT EXISTS users (
    user_id    TEXT NOT NULL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL UNIQUE,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

### 6.2.1 Table: locale_config (system.sqlite)

```sql
CREATE TABLE IF NOT EXISTS locale_config (
    language TEXT NOT NULL,
    region   TEXT NOT NULL,
    locale   TEXT NOT NULL,
    PRIMARY KEY (language, region)
);
```

Column notes:

- `language`: ISO 639-1 lowercase language code (e.g. `ja`, `en`).
- `region`: ISO 3166-1 alpha-2 uppercase region code (e.g. `JP`, `US`).
- `locale`: Resolved locale string returned to the client (e.g. `ja`, `en`).

Seed data inserted at startup:

| language | region | locale |
| --- | --- | --- |
| `ja` | `JP` | `ja` |
| `de` | `DE` | `de` |
| `zh` | `CN` | `zh` |
| `it` | `IT` | `it` |
| `fr` | `FR` | `fr` |

### 6.2.2 Table: user_locale_settings (system.sqlite)

```sql
CREATE TABLE IF NOT EXISTS user_locale_settings (
  user_id TEXT NOT NULL PRIMARY KEY,
  locale  TEXT NOT NULL
);
```

Column notes:

- `user_id` references `users.user_id` at the application validation layer.
- `locale` stores one explicit locale value from `en`, `de`, `fr`, `it`, `ja`, `zh`.

### 6.2.3 Table: user_credentials (system.sqlite)

```sql
CREATE TABLE IF NOT EXISTS user_credentials (
  user_id       TEXT NOT NULL PRIMARY KEY,
  password_hash TEXT NOT NULL,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);
```

Column notes:

- `password_hash` stores one-way hashed password bytes encoded as text.
- Initial admin bootstrap computes hash from initial password input value `admin`.

### 6.3 Table: projects (system.sqlite)

```sql
CREATE TABLE IF NOT EXISTS projects (
    project_id  TEXT NOT NULL PRIMARY KEY,
    name        TEXT NOT NULL,
    description TEXT NOT NULL DEFAULT '',
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL
);
```

### 6.4 Table: sprints (project_{id}.sqlite)

```sql
CREATE TABLE IF NOT EXISTS sprints (
    sprint_id       TEXT NOT NULL PRIMARY KEY,
    project_id      TEXT NOT NULL REFERENCES projects(project_id),
    name            TEXT NOT NULL,
    start_date      TEXT NOT NULL,
    end_date        TEXT NOT NULL,
    available_hours REAL NOT NULL DEFAULT 0,
    created_at      TEXT NOT NULL,
    updated_at      TEXT NOT NULL
);
```

### 6.5 Table: tasks (project_{id}.sqlite)

```sql
CREATE TABLE IF NOT EXISTS tasks (
    task_id        TEXT NOT NULL PRIMARY KEY,
    project_id     TEXT NOT NULL REFERENCES projects(project_id),
    sprint_id      TEXT REFERENCES sprints(sprint_id),
    title          TEXT NOT NULL,
    estimate_hours REAL,
    impact         TEXT,
    status         TEXT NOT NULL DEFAULT 'todo',
    created_at     TEXT NOT NULL,
    updated_at     TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_tasks_sprint_id ON tasks(sprint_id);
```

Column notes:

- `estimate_hours`: NULL if not yet set.
- `impact`: NULL if not yet set. Stored values: "high", "medium", "low".
- `status`: Stored values: "todo", "in_progress", "done".

### 6.6 Table: resources (project_{id}.sqlite)

```sql
CREATE TABLE IF NOT EXISTS resources (
    resource_id            TEXT NOT NULL PRIMARY KEY,
    name                   TEXT NOT NULL,
    capacity_hours_per_day REAL NOT NULL
);
```

### 6.7 Table: working_day_calendar (project_{id}.sqlite)

```sql
CREATE TABLE IF NOT EXISTS working_day_calendar (
    date       TEXT    NOT NULL PRIMARY KEY,
    is_working INTEGER NOT NULL DEFAULT 1
);
```

Column notes:

- `date`: ISO 8601 date string (e.g. "2026-04-01").
- `is_working`: 1 for working day, 0 for non-working day.

### 6.8 Table: task_resource_allocations (project_{id}.sqlite)

```sql
CREATE TABLE IF NOT EXISTS task_resource_allocations (
    task_id     TEXT NOT NULL REFERENCES tasks(task_id),
    resource_id TEXT NOT NULL REFERENCES resources(resource_id),
    hours       REAL NOT NULL,
    PRIMARY KEY (task_id, resource_id)
);
```

### 6.9 Table: project_roles (project_{id}.sqlite)

```sql
CREATE TABLE IF NOT EXISTS project_roles (
    project_id TEXT NOT NULL,
    user_id    TEXT NOT NULL,
    role       TEXT NOT NULL,
    PRIMARY KEY (project_id, user_id)
);
```

Column notes:

- `role`: Stored values: "administrator", "assignee".
- `user_id` is not enforced by SQLite FK across database files. P2 validates existence in `system.sqlite` before insert.

### 6.10 Table: user_menu_visibility (system.sqlite)

```sql
CREATE TABLE IF NOT EXISTS user_menu_visibility (
  user_id    TEXT NOT NULL,
  menu_key   TEXT NOT NULL,
  is_enabled INTEGER NOT NULL,
  PRIMARY KEY (user_id, menu_key)
);
```

Column notes:

- `menu_key`: Stored values: `project_select`, `sprint_workspace`, `resource_settings`, `calendar_settings`, `user_management`.
- `is_enabled`: 1 for enabled, 0 for disabled.

## 7. P1 Handler Processing Design

P1 router registration rules:

- `router.go` shall match browser UI routes before fallback 404 handling.
- `router.go` shall dispatch `/api/...` requests to API handlers.
- `router.go` shall dispatch non-API browser UI routes to browser UI shell rendering.
- `router.go` shall return `404 ROUTE_NOT_FOUND` for undefined browser UI routes.
- `router.go` shall return `404 ROUTE_NOT_FOUND` for undefined API routes.

### 7.1 Common Steps for All API Handlers

P1 executes these steps before API-specific logic for every API handler:

1. Read `X-Request-Id` from the HTTP request header. If absent, return `400` with `INVALID_PATH_PARAM`.
2. Extract path parameters from the URL.
3. Build `ZMQRequest` with `request_id`, `command`, resolved params.
4. Call `zmq.client.Send(req)`. If transport error, return `502 UPSTREAM_UNAVAILABLE`. If timeout, return `504 UPSTREAM_TIMEOUT`.
5. Map `ZMQResponse.Status` to HTTP status: "ok" to 200; "error" with domain validation code to 422; "error" with not-found code to 404.
6. Write HTTP response body with the common response schema.

### 7.2 HandleListProjects (API-01)

1. Build `ZMQRequest`: `command = "list_projects"`, empty `path_params`, empty `payload`.
2. Send to P2, return HTTP response.

### 7.3 HandleGetProjectSummary (API-02)

1. Extract `project_id` from path.
2. Build `ZMQRequest`: `command = "get_project_summary"`, `path_params = {"project_id": project_id}`.
3. Send to P2, return HTTP response.

### 7.4 HandleGetSprintWorkspace (API-03)

1. Extract `project_id`, `sprint_id` from path.
2. Build `ZMQRequest`: `command = "get_sprint_workspace"`, `path_params = {"project_id": ..., "sprint_id": ...}`.
3. Send to P2, return HTTP response.

### 7.5 HandleUpdateTask (API-04)

1. Extract `project_id`, `task_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "update_task"`, path params, serialized payload.
4. Send to P2, return HTTP response.

### 7.6 HandleListResources (API-05)

1. Extract `project_id` from path.
2. Build `ZMQRequest`: `command = "list_resources"`, `path_params = {"project_id": project_id}`.
3. Send to P2, return HTTP response.

### 7.7 HandleSaveResources (API-06)

1. Extract `project_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "save_resources"`, path params, serialized payload.
4. Send to P2, return HTTP response.

### 7.8 HandleGetCalendar (API-07)

1. Extract `project_id` from path.
2. Read optional query parameter `month` from URL.
3. Build `ZMQRequest`: `command = "get_calendar"`, `path_params = {"project_id": ...}`, `query_params = {"month": month}`.
4. Send to P2, return HTTP response.

### 7.9 HandleSaveCalendar (API-08)

1. Extract `project_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "save_calendar"`, path params, serialized payload.
4. Send to P2, return HTTP response.

### 7.10 HandleApplyCarryover (API-09)

1. Extract `project_id`, `sprint_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "apply_carryover"`, path params, serialized payload.
4. Send to P2, return HTTP response.

### 7.11 HandleListUsers (API-10)

1. Build `ZMQRequest`: `command = "list_users"`, empty `path_params`, empty `payload`.
2. Send to P2, return HTTP response.

### 7.12 HandleRegisterUser (API-11)

1. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
2. Build `ZMQRequest`: `command = "register_user"`, empty `path_params`, serialized payload.
3. Send to P2, return HTTP response.

### 7.13 HandleUpdateUser (API-12)

1. Extract `user_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "update_user"`, `path_params = {"user_id": user_id}`, serialized payload.
4. Send to P2, return HTTP response.

### 7.14 HandleDeleteUser (API-13)

1. Extract `user_id` from path.
2. Build `ZMQRequest`: `command = "delete_user"`, `path_params = {"user_id": user_id}`, empty `payload`.
3. Send to P2, return HTTP response with no `data` field.

### 7.15 HandleGetProjectRoles (API-14)

1. Extract `project_id` from path.
2. Build `ZMQRequest`: `command = "get_project_roles"`, `path_params = {"project_id": project_id}`.
3. Send to P2, return HTTP response.

### 7.16 HandleSaveProjectRoles (API-15)

1. Extract `project_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "save_project_roles"`, path params, serialized payload.
4. Send to P2, return HTTP response.

### 7.17 HandleGetDefaultLocale (API-16)

1. Read optional `Accept-Language` from request header.
2. Build `ZMQRequest`: `command = "resolve_default_locale"`, `query_params = {"accept_language": header_value}`.
3. Send to P2, return HTTP response.

### 7.18 HandleGetTopMenu (API-17)

1. Read `X-User-Id` from request header. If absent, return `400 INVALID_PATH_PARAM`.
2. Build `ZMQRequest`: `command = "get_top_menu"`, `query_params = {"user_id": header_value}`.
3. Send to P2, return HTTP response.

### 7.19 HandleGetUserMenuVisibility (API-18)

1. Extract `user_id` from path.
2. Build `ZMQRequest`: `command = "get_user_menu_visibility"`, `path_params = {"user_id": user_id}`.
3. Send to P2, return HTTP response.

### 7.20 HandleSaveUserMenuVisibility (API-19)

1. Extract `user_id` from path.
2. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
3. Build `ZMQRequest`: `command = "save_user_menu_visibility"`, `path_params = {"user_id": user_id}`, serialized payload.
4. Send to P2, return HTTP response.

### 7.21 HandleCreateProject (API-20)

1. Parse JSON body. If parsing fails, return `400 INVALID_JSON`.
2. Build `ZMQRequest`: `command = "create_project"`, empty `path_params`, serialized payload.
3. Send to P2, return HTTP response.

## 8. P2 Use Case Handler Design

SQL queries are shown after each processing steps list. Steps reference query labels (Q1, Q2, ...) defined in the Queries section of each handler.

### 8.1 UC-01 ListProjects

Function: `ListProjects(env ZMQRequest) ZMQResponse`

Processing steps:

1. Execute Q1 on `system.sqlite`.
2. Build response list from result rows sorted by name ascending.
3. Return `ZMQResponse` with `status = "ok"`, projects list as `data`.

**Q1  ESelect all projects (system.sqlite):**

```sql
SELECT project_id, name, description, updated_at
FROM projects
ORDER BY name ASC;
```

### 8.2 UC-01b GetProjectSummary

Function: `GetProjectSummary(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id` from `path_params`.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `PROJECT_NOT_FOUND`.
3. Open `project_{id}.sqlite`. Execute Q2, Q3.
4. Build response with project fields, aggregated counts.
5. Return `ZMQResponse` with `status = "ok"`.

**Q1  ESelect project by ID (system.sqlite):**

```sql
SELECT project_id, name, description, updated_at
FROM projects
WHERE project_id = ?;
```

**Q2  ECount sprints (project_{id}.sqlite):**

```sql
SELECT COUNT(*) AS sprint_count FROM sprints;
```

**Q3  ECount tasks (project_{id}.sqlite):**

```sql
SELECT COUNT(*) AS task_count FROM tasks;
```

### 8.3 UC-02 GetSprintWorkspace

Function: `GetSprintWorkspace(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id`, `sprint_id` from `path_params`.
2. Open `project_{id}.sqlite`. Execute Q1. If no row returned, return error `SPRINT_NOT_FOUND`.
3. Execute Q2 to fetch tasks for the sprint.
4. Initialize `cumulative = 0.0`.
5. For each task in order, set `needs_input = true` when `estimate_hours` is NULL.
6. For each task in order, set `needs_input = true` when `impact` is NULL.
7. If `needs_input` is true, append the task to `budget_out`.
8. If `needs_input` is false, `cumulative + estimate_hours <= available_hours`, append the task to `budget_in`, add `estimate_hours` to `cumulative`.
9. If `needs_input` is false, `cumulative + estimate_hours > available_hours`, append the task to `budget_out`.
10. Compute `totals.budget_in_hours` as sum of `estimate_hours` for budget-in tasks (treat NULL as 0).
11. Compute `totals.budget_out_hours` as sum of `estimate_hours` for budget-out tasks (treat NULL as 0).
12. Return `ZMQResponse` with `status = "ok"`.

**Q1 - Select sprint (project_{id}.sqlite):**

```sql
SELECT sprint_id, name, available_hours
FROM sprints
WHERE sprint_id = ? AND project_id = ?;
```

**Q2 - Select tasks in sprint (project_{id}.sqlite):**

```sql
SELECT task_id, title, estimate_hours, impact, status
FROM tasks
WHERE sprint_id = ?
ORDER BY task_id ASC;
```

### 8.4 UC-03 UpdateTask

Function: `UpdateTask(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id`, `task_id` from `path_params`. Decode payload into `UpdateTaskPayload`.
2. Open `project_{id}.sqlite`. Execute Q1. If no row returned, return error `SPRINT_NOT_FOUND`.
3. If `estimate_hours` is present, `estimate_hours < 0`, return error `INVALID_ESTIMATE`.
4. If `impact` is present, value not in `{"high", "medium", "low"}`, return error `INVALID_IMPACT`.
5. Build SET clause dynamically from non-nil payload fields plus `updated_at`. Execute Q2.
6. Re-read the updated row. Return `ZMQResponse` with `status = "ok"`. Return updated task as `data`.

**Q1 - Check task existence (project_{id}.sqlite):**

```sql
SELECT task_id FROM tasks WHERE task_id = ? AND project_id = ?;
```

**Q2  EUpdate task fields (project_{id}.sqlite):**

```sql
UPDATE tasks SET <field> = ?, updated_at = ? WHERE task_id = ?;
```

Note: SET clause is built dynamically from non-nil payload fields. `<field>` is replaced with the actual column name at runtime.

### 8.5 UC-04 SaveResources

Function: `SaveResources(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id` from `path_params`. Decode payload into resource list.
2. Validate: if any `resource_id` appears more than once in the list, return error `DUPLICATE_RESOURCE_ID`.
3. Validate: if any `capacity_hours_per_day <= 0`, return error `INVALID_RESOURCE_CAPACITY`.
4. Open `project_{id}.sqlite`. Begin transaction. Execute Q1.
5. For each resource item, execute Q2.
6. Commit transaction.
7. Return `ZMQResponse` with `status = "ok"`. Return saved resource list as `data`.

**Q1  EDelete all resources (project_{id}.sqlite):**

```sql
DELETE FROM resources;
```

**Q2  EInsert resource (project_{id}.sqlite):**

```sql
INSERT INTO resources (resource_id, name, capacity_hours_per_day)
VALUES (?, ?, ?);
```

### 8.6 UC-05 GetCalendar

Function: `GetCalendar(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id` from `path_params`. Read `month` from `query_params`.
2. If `month` is empty, set `month` to the current year-month in format "YYYY-MM".
3. Compute date range: `start = month + "-01"`, `end = last day of the month`.
4. Open `project_{id}.sqlite`. Execute Q1.
5. Build response with `month`. Build response with `days` list.
6. Return `ZMQResponse` with `status = "ok"`.

**Q1  ESelect calendar days for month (project_{id}.sqlite):**

```sql
SELECT date, is_working
FROM working_day_calendar
WHERE date >= ? AND date <= ?
ORDER BY date ASC;
```

### 8.7 UC-05b SaveCalendar

Function: `SaveCalendar(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id` from `path_params`. Decode payload into calendar day list.
2. Validate: if any `date` value is not a valid ISO 8601 date, return error `INVALID_PATH_PARAM`.
3. Validate: if any `date` value appears more than once in the list, return error `DUPLICATE_CALENDAR_DATE`.
4. Open `project_{id}.sqlite`. Begin transaction. For each day item, execute Q1.
5. Commit transaction.
6. Return `ZMQResponse` with `status = "ok"`. Return saved days list as `data`.

**Q1  EUpsert calendar day (project_{id}.sqlite):**

```sql
INSERT INTO working_day_calendar (date, is_working) VALUES (?, ?)
ON CONFLICT(date) DO UPDATE SET is_working = excluded.is_working;
```

### 8.8 UC-06 ApplyCarryover

Function: `ApplyCarryover(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id`, `sprint_id` from `path_params`. Decode payload into decisions list.
2. Open `project_{id}.sqlite`.
3. For each decision with `action = "carryover"`, with non-null `target_sprint_id`, execute Q1. If no row returned, return error `TARGET_SPRINT_NOT_FOUND`.
4. Begin transaction. For each decision with `action = "carryover"`, execute Q2 (set `sprint_id` to `target_sprint_id`; may be NULL).
5. For decisions with `action = "keep"`, perform no SQL update; include in result with unchanged `sprint_id`.
6. Commit transaction.
7. Build `applied` list from each decision result. Return `ZMQResponse` with `status = "ok"`.

**Q1  ECheck target sprint existence (project_{id}.sqlite):**

```sql
SELECT sprint_id FROM sprints WHERE sprint_id = ? AND project_id = ?;
```

**Q2  EUpdate task sprint assignment (project_{id}.sqlite):**

```sql
UPDATE tasks SET sprint_id = ?, updated_at = ? WHERE task_id = ?;
```

### 8.9 UC-07 ListUsers

Function: `ListUsers(env ZMQRequest) ZMQResponse`

Processing steps:

1. Execute Q1 on `system.sqlite`.
2. Build response list. Return `ZMQResponse` with `status = "ok"`. Return users list as `data`.

**Q1  ESelect all users (system.sqlite):**

```sql
SELECT user_id, name, email
FROM users
ORDER BY user_id ASC;
```

### 8.10 UC-08 RegisterUser

Function: `RegisterUser(env ZMQRequest) ZMQResponse`

Processing steps:

1. Decode payload into `RegisterUserPayload` with fields `user_id`, `name`, `email`.
2. Execute Q1 on `system.sqlite`. If row exists, return error `DUPLICATE_USER_ID`.
3. Execute Q2 on `system.sqlite`. Set `created_at`. Set `updated_at` to current UTC time in RFC3339.
4. Return `ZMQResponse` with `status = "ok"`. Return registered user as `data`.

**Q1  ECheck user_id uniqueness (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q2  EInsert user (system.sqlite):**

```sql
INSERT INTO users (user_id, name, email, created_at, updated_at)
VALUES (?, ?, ?, ?, ?);
```

### 8.11 UC-09 UpdateUser

Function: `UpdateUser(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `user_id` from `path_params`. Decode payload into `UpdateUserPayload`.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `USER_NOT_FOUND`.
3. Build SET clause from non-nil payload fields plus `updated_at`. Execute Q2 on `system.sqlite`.
4. Re-read the updated row. Return `ZMQResponse` with `status = "ok"`. Return updated user as `data`.

**Q1  ECheck user existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q2  EUpdate user fields (system.sqlite):**

```sql
UPDATE users SET <field> = ?, updated_at = ? WHERE user_id = ?;
```

Note: SET clause is built dynamically from non-nil payload fields.

### 8.12 UC-10 DeleteUser

Function: `DeleteUser(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `user_id` from `path_params`.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `USER_NOT_FOUND`.
3. Execute Q2 on `system.sqlite` to delete the user row.
4. Execute Q3 on `system.sqlite` to enumerate all project identifiers.
5. For each `project_id`, open `project_{id}.sqlite`. Execute Q4 to remove role rows for the deleted user.
6. Return `ZMQResponse` with `status = "ok"`. Return no `data` field.

Note: Steps 3 plus 5 run in separate SQLite transactions across different database files. Atomicity is not guaranteed across files. Step 3 completes first. If step 5 fails for a project, an orphaned role row may remain. P2 tolerates orphaned rows on read by skipping role entries whose `user_id` is absent from `system.sqlite`.

**Q1  ECheck user existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q2  EDelete user (system.sqlite):**

```sql
DELETE FROM users WHERE user_id = ?;
```

**Q3  EList all project IDs (system.sqlite):**

```sql
SELECT project_id FROM projects;
```

**Q4  EDelete role rows for user (project_{id}.sqlite):**

```sql
DELETE FROM project_roles WHERE user_id = ?;
```

### 8.13 UC-11 GetProjectRoles

Function: `GetProjectRoles(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id` from `path_params`.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `PROJECT_NOT_FOUND`.
3. Open `project_{id}.sqlite`. Execute Q2.
4. Return `ZMQResponse` with `status = "ok"`. Return roles list as `data`.

**Q1  ECheck project existence (system.sqlite):**

```sql
SELECT project_id FROM projects WHERE project_id = ?;
```

**Q2  ESelect project roles (project_{id}.sqlite):**

```sql
SELECT user_id, role
FROM project_roles
WHERE project_id = ?
ORDER BY user_id ASC;
```

### 8.14 UC-12 SaveProjectRoles

Function: `SaveProjectRoles(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `project_id` from `path_params`. Decode payload into role assignment list.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `PROJECT_NOT_FOUND`.
3. Validate: if any `role` value is not in `{"administrator", "assignee"}`, return error `INVALID_ROLE`.
4. For each `user_id` in the payload, execute Q2 on `system.sqlite`. If no row returned for any `user_id`, return error `USER_NOT_FOUND`.
5. Open `project_{id}.sqlite`. Begin transaction. Execute Q3.
6. For each role item, execute Q4.
7. Commit transaction.
8. Return `ZMQResponse` with `status = "ok"`. Return saved roles list as `data`.

**Q1  ECheck project existence (system.sqlite):**

```sql
SELECT project_id FROM projects WHERE project_id = ?;
```

**Q2  ECheck user existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q3  EDelete existing project roles (project_{id}.sqlite):**

```sql
DELETE FROM project_roles WHERE project_id = ?;
```

**Q4  EInsert project role (project_{id}.sqlite):**

```sql
INSERT INTO project_roles (project_id, user_id, role) VALUES (?, ?, ?);
```

### 8.15 UC-13 ResolveDefaultLocale

Function: `ResolveDefaultLocale(env ZMQRequest) ZMQResponse`

Processing steps:

1. Read `accept_language` from `query_params`. If empty, return `ZMQResponse` with `data = {"locale": "en", "source": "fallback"}`.
2. Split `accept_language` by `,` into tag entries.
3. For each tag entry, strip `;q=<value>` suffix. Parse `q` as float64. Default `q = 1.0` when `;q=` is absent.
4. Sort tag entries by `q` value descending.
5. For each sorted tag (highest `q` first):
   a. Normalize: replace `_` with `-`.
   b. Split by `-`. If fewer than 2 segments, skip (region-less tag).
   c. Set `language = lowercase(segments[0])`, `region = uppercase(segments[1])`.
   d. Execute Q1 with `language` and `region`.
   e. If Q1 returns a row, return `ZMQResponse` with `data = {"locale": matched_locale, "source": "matched"}`.
6. If no tag matched, return `ZMQResponse` with `data = {"locale": "en", "source": "fallback"}`.

**Q1 - Select locale configuration by language-region (system.sqlite):**

```sql
SELECT locale
FROM locale_config
WHERE language = ? AND region = ?
LIMIT 1;
```

Note: If `locale_config` is not seeded, Q1 returns no row for all tags and fallback `en` is returned.

### 8.16 UC-14 GetTopMenu

Function: `GetTopMenu(env ZMQRequest) ZMQResponse`

Processing steps:

1. Read `user_id` from `query_params`.
2. Execute Q1 on `system.sqlite` to verify the user exists. If no row returned, return error `USER_NOT_FOUND`.
3. Execute Q2 on `system.sqlite` to read user-specific menu visibility rows.
4. If Q2 returns no row, build default enabled menu list.
5. If Q2 returns rows, filter only `is_enabled = 1` rows.
6. Build response menu button list with `{menu_key, label}`.
7. Return `ZMQResponse` with `status = "ok"`.

**Q1 - Check user existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q2 - Select user menu visibility (system.sqlite):**

```sql
SELECT menu_key, is_enabled
FROM user_menu_visibility
WHERE user_id = ?
ORDER BY menu_key ASC;
```

### 8.17 UC-15 GetUserMenuVisibility

Function: `GetUserMenuVisibility(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `user_id` from `path_params`.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `USER_NOT_FOUND`.
3. Execute Q2 on `system.sqlite`.
4. If Q2 returns no row, build default visibility setting list with all allowed keys enabled.
5. Return `ZMQResponse` with `status = "ok"`.

**Q1 - Check user existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q2 - Select menu visibility rows (system.sqlite):**

```sql
SELECT menu_key, is_enabled
FROM user_menu_visibility
WHERE user_id = ?
ORDER BY menu_key ASC;
```

### 8.18 UC-16 SaveUserMenuVisibility

Function: `SaveUserMenuVisibility(env ZMQRequest) ZMQResponse`

Processing steps:

1. Extract `user_id` from `path_params`. Decode payload into menu visibility list.
2. Execute Q1 on `system.sqlite`. If no row returned, return error `USER_NOT_FOUND`.
3. Validate: if any `menu_key` is not in allowed set, return error `INVALID_MENU_KEY`.
4. Validate: if any `menu_key` appears more than once in the list, return error `DUPLICATE_MENU_KEY`.
5. Begin transaction on `system.sqlite`. Execute Q2.
6. For each item, execute Q3.
7. Commit transaction.
8. Return `ZMQResponse` with `status = "ok"`. Return saved visibility list as `data`.

**Q1 - Check user existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q2 - Delete current menu visibility rows (system.sqlite):**

```sql
DELETE FROM user_menu_visibility WHERE user_id = ?;
```

**Q3 - Insert menu visibility row (system.sqlite):**

```sql
INSERT INTO user_menu_visibility (user_id, menu_key, is_enabled)
VALUES (?, ?, ?);
```

### 8.19 Startup InitializeAdminUser

Function: `InitializeAdminUser() error`

Processing steps:

1. Execute Q1 on `system.sqlite` to check whether user_id `admin` exists.
2. If Q1 returns a row, finish with success.
3. Compute `password_hash` from initial password input value `admin`.
4. Begin transaction on `system.sqlite`.
5. Execute Q2 to insert `users` row for `admin`.
6. Execute Q3 to insert `user_credentials` row for `admin`.
7. Commit transaction.
8. If any SQL step fails, rollback transaction. Return `INITIAL_USER_BOOTSTRAP_FAILED` mapped error.

**Q1 - Check initial admin existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = 'admin';
```

**Q2 - Insert initial admin user profile (system.sqlite):**

```sql
INSERT INTO users (user_id, name, email, created_at, updated_at)
VALUES ('admin', 'Administrator', 'admin@local', ?, ?);
```

**Q3 - Insert initial admin credential (system.sqlite):**

```sql
INSERT INTO user_credentials (user_id, password_hash, created_at, updated_at)
VALUES ('admin', ?, ?, ?);
```

### 8.20 UC-17 CreateProject

Function: `CreateProject(env ZMQRequest) ZMQResponse`

Processing steps:

1. Decode payload into `CreateProjectPayload`.
2. Execute Q1 on `system.sqlite`. If row exists, return error `DUPLICATE_PROJECT_ID`.
3. Execute Q2 on `system.sqlite` to verify `initial_admin_user_id` exists. If no row returned, return error `USER_NOT_FOUND`.
4. Validate: if `initial_sprint.start_date` is after `initial_sprint.end_date`, return error `INVALID_SPRINT_DATE_RANGE`.
5. Begin transaction on `system.sqlite`. Execute Q3 to insert project row. Commit transaction.
6. Open `project_{id}.sqlite`. Begin transaction. Execute Q4 to insert initial sprint row.
7. Execute Q5 to insert initial project administrator role row.
8. Commit transaction.
9. Return `ZMQResponse` with `status = "ok"`. Return registered project and initial sprint summary as `data`.

**Q1 - Check project_id uniqueness (system.sqlite):**

```sql
SELECT project_id FROM projects WHERE project_id = ?;
```

**Q2 - Check initial administrator existence (system.sqlite):**

```sql
SELECT user_id FROM users WHERE user_id = ?;
```

**Q3 - Insert project (system.sqlite):**

```sql
INSERT INTO projects (project_id, name, description, created_at, updated_at)
VALUES (?, ?, ?, ?, ?);
```

**Q4 - Insert initial sprint (project_{id}.sqlite):**

```sql
INSERT INTO sprints (sprint_id, project_id, name, start_date, end_date, available_hours, created_at, updated_at)
VALUES (?, ?, ?, ?, ?, 0, ?, ?);
```

**Q5 - Insert initial administrator role (project_{id}.sqlite):**

```sql
INSERT INTO project_roles (project_id, user_id, role)
VALUES (?, ?, 'administrator');
```

## 9. Error Code Reference

| Code | Origin | HTTP Status | Trigger Condition |
| --- | --- | --- | --- |
| INVALID_PATH_PARAM | P1 | 400 | Required path parameter missing or malformed |
| INVALID_JSON | P1 | 400 | JSON body parse failure |
| ROUTE_NOT_FOUND | P1 | 404 | No route matches the request path |
| UNKNOWN_COMMAND | P2 | 400 | Command value not in dispatcher registry |
| PROJECT_NOT_FOUND | P2 | 404 | Project row absent in system.sqlite |
| SPRINT_NOT_FOUND | P2 | 404 | Sprint row absent in project_{id}.sqlite |
| TARGET_SPRINT_NOT_FOUND | P2 | 404 | Target sprint for carry-over absent |
| USER_NOT_FOUND | P2 | 404 | User row absent in system.sqlite |
| INVALID_ESTIMATE | P2 | 422 | estimate_hours < 0 |
| INVALID_IMPACT | P2 | 422 | impact value not in allowed set |
| DUPLICATE_RESOURCE_ID | P2 | 422 | Same resource_id appears more than once in input |
| INVALID_RESOURCE_CAPACITY | P2 | 422 | capacity_hours_per_day <= 0 |
| DUPLICATE_CALENDAR_DATE | P2 | 422 | Same date appears more than once in input |
| DUPLICATE_USER_ID | P2 | 422 | user_id already exists in users table |
| INVALID_ROLE | P2 | 422 | role value not in "administrator", "assignee" |
| INVALID_MENU_KEY | P2 | 422 | menu_key value not in allowed set |
| DUPLICATE_MENU_KEY | P2 | 422 | Same menu_key appears more than once in input |
| DUPLICATE_PROJECT_ID | P2 | 422 | project_id already exists in projects table |
| INVALID_SPRINT_DATE_RANGE | P2 | 422 | initial_sprint.start_date is after initial_sprint.end_date |
| INITIAL_USER_BOOTSTRAP_FAILED | P2 | 500 | Startup bootstrap for initial admin user fails |
| PERSISTENCE_ERROR | P2 | 500 | SQLite read operation failure, SQLite write operation failure |
| UPSTREAM_UNAVAILABLE | P1 | 502 | ZMQ transport failure |
| UPSTREAM_TIMEOUT | P1 | 504 | ZMQ response not received within 3000 ms |

## 10. Traceability to Method Design

| Method Design Section | Detailed Design Section |
| --- | --- |
| §3.1 Process List (P1, P2) | §2 Package Structure |
| §3.2 Data Store Access Rule | §6.1 Storage Topology |
| §4.1 IF-HTTP-01 endpoint contracts | §5 API Request/Response Payload Schemas |
| §4.2 IF-ZMQ-01 message schema | §4 ZMQ Message Struct Definitions |
| §4.3 IF-DB-01 primary tables | §6 Database Schema |
| §5.1 P1 validation/response mapping rules | §7 P1 Handler Processing Design |
| §5.2 P2 dispatch/persistence rules | §8 P2 Use Case Handler Design |
| §5.3 P2 startup initialization process | §8.19 Startup InitializeAdminUser |
| §6.1 to 6.17 Use case conditions/behaviors | §8.1 to 8.20 Use case handler processing steps |
| §7 Error model | §9 Error Code Reference |
| §8 Timeouts/retries | §7.1 Common Steps (transport error, timeout handling) |

## 11. Traceability to Requirements

| Requirement ID | SRS Section | Requirement Summary | Detailed Design Section | Design Element |
| --- | --- | --- | --- | --- |
| SRS-SYS-04 | §2 Software Architecture | Application component for MVC-based domain logic | §2.3 MVC Responsibility Mapping, §8 P2 Use Case Handler Design, §7.1 Common Steps | MVC role boundaries, P2 use case orchestration, P1 response construction boundary |
| SRS-UI-10 | §4.2 Common UI Requirements | Provide a top page as a major screen | §5.17 API-17, §7.18, §8.16 | Top menu endpoint, top menu handler, top menu use case |
| SRS-UI-11 | §4.2 Common UI Requirements | Display top-page menu entries as buttons | §5.17 API-17, §8.16 | Response payload `menu_buttons[{menu_key,label}]` |
| SRS-UI-12 | §4.2 Common UI Requirements | Support menu visibility settings per user | §5.18 API-18, §5.19 API-19, §7.19, §7.20, §8.17, §8.18, §6.10 | User menu visibility read/write endpoints, handlers, use cases, table |
| SRS-UI-13 | §4.2 Common UI Requirements | Display only enabled menu buttons for the signed-in user | §8.16 UC-14 | Filter rule `is_enabled = 1` for signed-in user |
| SRS-UI-14 | §4.2.1 UI Placement URL | Place each major UI at the defined stable URL | §5.0 Browser UI Routes, §7 P1 Handler Processing Design | Browser UI route registration schema, router registration rules |
| SRS-SC-02 | §4.3.2 Top Page Screen | Display menu area plus menu visibility settings area | §5.17 to §5.19, §8.16 to §8.18 | Top menu retrieval plus per-user menu visibility retrieval/update |
| SRS-SC-08 | §4.3.8 Project Register Screen | Display project registration inputs plus initial sprint inputs plus administrator assignment input | §5.20, §7.21, §8.20 | Project registration payload schema, P1 registration handler, P2 project creation use case |
| SRS-UM-00 | §5.0 Initial User Requirements | Prepare initial user `admin` with initial password `admin` | §6.2.1, §8.19 | Startup bootstrap inserts `admin` user plus credential hash from initial password value |
