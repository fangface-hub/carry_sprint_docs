# Constraints and Rules

## General

Every rule in this document is enforced at the application layer unless stated otherwise.  
Rules marked **DB** are also enforced at the database layer.

---

## Sprint Rules

### SR-01 — Name is required

A sprint name must not be empty.

### SR-02 — Dates are required

A sprint must have both a `start_date` and an `end_date`.

### SR-03 — Date ordering

A sprint's `start_date` must be less than or equal to its `end_date`.

### SR-04 — Date format

All dates must be provided in `YYYY-MM-DD` format.

### SR-05 — Status transitions

Sprint status transitions must follow this sequence:

```
planned --> active --> completed
```

A sprint cannot move backward in status.  
A sprint cannot skip a status.

### SR-06 — Active sprint limit

At most one sprint may have `status = "active"` at any time.

### SR-07 — Completed sprint is immutable

A sprint with `status = "completed"` cannot be updated.  
A sprint with `status = "completed"` cannot be deleted.

### SR-08 — Cascade delete

Deleting a sprint deletes all its tasks. **(DB)**

---

## Task Rules

### TR-01 — Title is required

A task title must not be empty.

### TR-02 — Sprint association is required

A task must belong to a sprint. **(DB)**

### TR-03 — Status values

A task `status` must be one of: `todo`, `in_progress`, `done`.

### TR-04 — Status transitions

Task status transitions must follow this sequence:

```
todo --> in_progress --> done
```

A task cannot move backward in status.

### TR-05 — Priority values

A task `priority` must be one of: `low`, `medium`, `high`.

### TR-06 — Estimate range

A task `estimate` must be a positive integer if provided.  
A task `estimate` of zero is not allowed.

### TR-07 — Tasks in completed sprint

Tasks in a sprint with `status = "completed"` cannot be updated or deleted.

---

## Carry Rules

### CR-01 — Source sprint must exist

The source sprint ID provided to the carry operation must identify an existing sprint.

### CR-02 — Target sprint must exist

The target sprint ID provided to the carry operation must identify an existing sprint.

### CR-03 — Source and target must differ

The source sprint and target sprint must have different IDs.

### CR-04 — Only incomplete tasks are carried

Only tasks with `status != "done"` are copied to the target sprint.

### CR-05 — Carried tasks are new records

Each carried task is a new row in the `tasks` table with a new `id`.  
The original task records in the source sprint are not modified or deleted.

### CR-06 — Carried task fields

A carried task copies the following fields from the source task:  
`title`, `description`, `priority`, `estimate`.  
A carried task always starts with `status = "todo"`.

---

## API Rules

### AR-01 — Content type

All requests with a body must use `Content-Type: application/json`.

### AR-02 — Unknown fields

The server ignores unknown fields in request bodies.

### AR-03 — ID in URL

Resource IDs are provided in the URL path, not in the request body.

### AR-04 — Loopback only

The server only accepts connections from `127.0.0.1`.  
Requests from any other IP address are rejected with HTTP status `403`.

---

## Data Rules

### DR-01 — Datetime storage

All datetimes are stored and returned in UTC.

### DR-02 — Datetime format

All datetimes are formatted as ISO 8601 with time zone suffix `Z`.

### DR-03 — No soft delete

Records are deleted permanently.  
There is no soft-delete mechanism.

### DR-04 — No audit log

There is no built-in audit log.  
The `created_at` and `updated_at` timestamps serve as the only historical record.
