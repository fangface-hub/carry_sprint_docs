# CarrySprint Software Method Design

## 1. Purpose

This document defines software methods that realize the requirements in `docs/requirements/software_requirements_specification.md`.
This document defines inter-process interfaces, process inputs and outputs, process conditions, process behaviors.

## 2. Scope

Target runtime structure:

- Client: Web browser
- Server Process A: Web Gateway Process
- Server Process B: Application Process
- Data Store: SQLite per project file

Target functional scope:

- Project selection
- Sprint workspace retrieval
- Task update
- Resource settings retrieval and update
- Working-day calendar retrieval and update
- Carry-over review apply

## 3. Process Model

### 3.1 Process List

| Process ID | Process Name | Responsibility |
| --- | --- | --- |
| P1 | Web Gateway Process | HTTP endpoint, request validation, ZeroMQ relay, HTTP response mapping |
| P2 | Application Process | Domain logic execution, persistence access, response payload composition |

### 3.2 Data Store Access Rule

- P1 never accesses SQLite directly.
- P2 handles all SQLite read and write operations.
- One project uses one SQLite file with format `project_<id>.sqlite`.

### 3.3 Data Flow Diagram

The following DFD defines process-level data flow among browser, P1, P2, SQLite.

```plantuml
!include ./diagrams/dfd/dfd_level1.puml
```

## 4. Inter-Process Interfaces

### 4.1 Interface IF-HTTP-01 (Browser -> P1)

Protocol:

- HTTP/1.1 or HTTP/2 over HTTPS
- Content-Type: `application/json`

Request header rules:

- `X-Request-Id` is required.
- `Content-Type` is required for write requests.

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

## 5. Process Input and Output

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

## 6. Use Case Conditions and Behaviors

### 6.1 UC-01 List Projects

- Condition: request has valid header `X-Request-Id`. Behavior: P1 sends `list_projects` to P2.
- Condition: P2 finds project rows. Behavior: return sorted project list.
- Condition: P2 finds no rows. Behavior: return empty list.

### 6.2 UC-02 Get Sprint Workspace

- Condition: project exists, sprint exists. Behavior: read tasks in sprint scope, classify budget-in and budget-out.
- Condition: task estimated hours or impact value is null. Behavior: classify task as budget-out with flag `needs_input=true`.
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

- Condition: task belongs to current sprint and decision is carry-over. Behavior: clear sprint_id or move to next sprint based on input target.
- Condition: task is marked as keep. Behavior: keep sprint_id unchanged.
- Condition: target next sprint does not exist. Behavior: return `TARGET_SPRINT_NOT_FOUND`.

### 6.7 Activity Diagrams

The following activity diagram defines UC-02 control flow with condition branches.

```plantuml
!include ./diagrams/activity/activity_uc02_get_sprint_workspace.puml
```

The following activity diagram defines UC-06 control flow with transaction and rollback branches.

```plantuml
!include ./diagrams/activity/activity_uc06_apply_carryover.puml
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
| PERSISTENCE_ERROR | SQLite read or write failed | 500 |
| UPSTREAM_UNAVAILABLE | ZeroMQ transport failure | 502 |
| UPSTREAM_TIMEOUT | ZeroMQ timeout | 504 |

## 8. Timeouts and Retries

- P1 timeout for IF-ZMQ-01 uses 3000 ms.
- P1 performs no automatic retry for write commands.
- P1 performs one retry for read commands after timeout.
- P2 SQL busy timeout uses 2000 ms.

## 9. Traceability to Requirements

| Requirement Area | Method Design Coverage |
| --- | --- |
| System architecture with browser, server, IPC, SQLite | Section 2, Section 4 |
| Process-level data flow definition | Section 3.3 |
| ZeroMQ request and response path | Section 4.2, Section 8 |
| Per-project SQLite policy | Section 3.2, Section 4.3 |
| Resource information handling | API-05, API-06, UC-04 |
| Working-day calendar handling | API-07, API-08, UC-05 |
| Budget-in and budget-out UI support | API-03, UC-02, Section 6.7 |
| Carry-over decision support | API-09, UC-06, Section 6.7 |

## 10. Implementation Constraints

- P1 implementation language uses Go.
- P2 implementation language uses Go.
- P1 keeps route handlers stateless.
- P2 keeps domain handlers deterministic for same input.
- Domain handlers avoid hidden side effects outside SQLite transaction boundary.
