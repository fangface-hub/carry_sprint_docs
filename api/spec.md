# API Specification

## Base URL

```
http://127.0.0.1:7654
```

## Protocol

All requests use HTTP/1.1.  
All request and response bodies use JSON.  
All responses include the header `Content-Type: application/json`.  
Successful responses use HTTP status code `200` unless otherwise specified.  
Created resources use HTTP status code `201`.  
Not-found errors use HTTP status code `404`.  
Validation errors use HTTP status code `422`.  
Server errors use HTTP status code `500`.

## Error Response Format

All error responses share the same JSON structure.

```json
{
  "error": "<human-readable message>"
}
```

---

## Health

### GET /health

Returns the server health status.

**Response**

```json
{
  "status": "ok",
  "version": "1.0.0"
}
```

---

## Sprints

A sprint is a fixed time-box with a name, start date, and end date.

### Sprint Object

| Field | Type | Description |
|---|---|---|
| `id` | integer | Unique identifier, auto-incremented |
| `name` | string | Sprint name, non-empty |
| `start_date` | string | ISO 8601 date (`YYYY-MM-DD`) |
| `end_date` | string | ISO 8601 date (`YYYY-MM-DD`) |
| `status` | string | One of: `planned`, `active`, `completed` |
| `created_at` | string | ISO 8601 datetime |
| `updated_at` | string | ISO 8601 datetime |

### GET /sprints

Returns all sprints ordered by `start_date` ascending.

**Response**

```json
[
  {
    "id": 1,
    "name": "Sprint 1",
    "start_date": "2026-01-01",
    "end_date": "2026-01-14",
    "status": "completed",
    "created_at": "2026-01-01T00:00:00Z",
    "updated_at": "2026-01-14T00:00:00Z"
  }
]
```

### GET /sprints/:id

Returns a single sprint by ID.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Sprint ID |

**Response** — Sprint object.

### POST /sprints

Creates a new sprint.

**Request Body**

| Field | Required | Description |
|---|---|---|
| `name` | yes | Sprint name |
| `start_date` | yes | ISO 8601 date |
| `end_date` | yes | ISO 8601 date |

**Response** — Created sprint object. HTTP status `201`.

### PUT /sprints/:id

Updates an existing sprint.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Sprint ID |

**Request Body** — Same fields as POST, all optional.

**Response** — Updated sprint object.

### DELETE /sprints/:id

Deletes a sprint and all its tasks.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Sprint ID |

**Response** — HTTP status `204`, no body.

---

## Tasks

A task belongs to a sprint.

### Task Object

| Field | Type | Description |
|---|---|---|
| `id` | integer | Unique identifier, auto-incremented |
| `sprint_id` | integer | ID of the owning sprint |
| `title` | string | Task title, non-empty |
| `description` | string | Optional free-text description |
| `status` | string | One of: `todo`, `in_progress`, `done` |
| `priority` | string | One of: `low`, `medium`, `high` |
| `estimate` | integer | Estimated effort in story points, nullable |
| `created_at` | string | ISO 8601 datetime |
| `updated_at` | string | ISO 8601 datetime |

### GET /sprints/:sprint_id/tasks

Returns all tasks for a sprint ordered by `priority` descending then `id` ascending.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `sprint_id` | integer | Sprint ID |

**Response**

```json
[
  {
    "id": 1,
    "sprint_id": 1,
    "title": "Set up project",
    "description": "",
    "status": "done",
    "priority": "high",
    "estimate": 2,
    "created_at": "2026-01-01T00:00:00Z",
    "updated_at": "2026-01-02T00:00:00Z"
  }
]
```

### GET /tasks/:id

Returns a single task by ID.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Task ID |

**Response** — Task object.

### POST /sprints/:sprint_id/tasks

Creates a new task within a sprint.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `sprint_id` | integer | Sprint ID |

**Request Body**

| Field | Required | Description |
|---|---|---|
| `title` | yes | Task title |
| `description` | no | Free-text description |
| `priority` | no | `low`, `medium`, or `high`; defaults to `medium` |
| `estimate` | no | Story points |

**Response** — Created task object. HTTP status `201`.

### PUT /tasks/:id

Updates an existing task.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Task ID |

**Request Body** — Same fields as POST, all optional. `status` is also updatable.

**Response** — Updated task object.

### DELETE /tasks/:id

Deletes a task.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Task ID |

**Response** — HTTP status `204`, no body.

---

## Carry

Carry moves incomplete tasks from one sprint to the next.

### POST /sprints/:id/carry

Copies all tasks with `status != "done"` from the given sprint into the target sprint.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Source sprint ID |

**Request Body**

| Field | Required | Description |
|---|---|---|
| `target_sprint_id` | yes | ID of the destination sprint |

**Response**

```json
{
  "carried": 3
}
```

The `carried` field contains the number of tasks moved.

---

## Summary

### GET /sprints/:id/summary

Returns a summary of task counts for a sprint.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | integer | Sprint ID |

**Response**

```json
{
  "sprint_id": 1,
  "total": 10,
  "todo": 2,
  "in_progress": 1,
  "done": 7
}
```
