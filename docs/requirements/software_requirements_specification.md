# CarrySprint Software Requirements Specification

## 1. System Architecture

This diagram shows the runtime structure of CarrySprint.

| Element | Description |
| --- | --- |
| Client | Browser-based user access |
| Server | Go-based server runtime |
| Inter-process communication path | ZeroMQ-based request and response path |
| Project storage | SQLite-based per-project storage |

```plantuml
!include ../architecture/system_architecture.puml
```

### 1.1 Design Writing Policy

- Do not use `or` to express alternatives.
- Do not use `and` to join parallel elements.
- Do not use `except` to define exclusion.
- Do not use passive voice.
- Do not use `it is necessary`.
- Write one sentence for one meaning.
- Define conditions and behavior explicitly.
- Treat the `else` clause of a Go `if` statement as an exception path. Do not write assumed normal handling in an `else` clause.
- Treat the `default` clause of a Go `switch` statement as an exception path. Do not write assumed normal handling in a `default` clause.

## 2. Software Architecture

This diagram shows the software component structure of CarrySprint.

| Element | Description |
| --- | --- |
| Browser client | User-facing web client |
| API gateway | HTTP entry point and request routing boundary |
| Application component | MVC-based application logic |
| SQLite | Per-project persistent storage |

```plantuml
!include ../architecture/software_architecture.puml
```

## 3. Database Design

This diagram shows the logical database structure for each project.

| Element | Description |
| --- | --- |
| Main entities | Core tables in the project database |
| Main attributes | Representative columns for each entity |
| Relationships | Logical relationships between entities |
| Storage unit | One SQLite file per project |

```plantuml
!include ../architecture/database_design.puml
```

### 3.1 Database Policy

- Use SQLite as the database engine.
- Isolate data by project with one SQLite file per project.
- Use project_{id}.sqlite as the standard file naming format.
- Manage resource information and the working-day calendar in SQLite.

## 4. Screen Design

This diagram shows the screen structure of the browser client.

| Element | Description |
| --- | --- |
| Major screens | Primary screens in the browser client |
| Navigation paths | Screen transitions between primary screens |

```plantuml
!include ../ui/screen_design.puml
```

### 4.1 Screen Design Policy

- Run the client in a web browser.
- Require no additional software installation on the client side.
- Exclude progress-monitoring UI and focus on selection and adjustment.
- Center the screen layout on budget-in and budget-out visualization.

### 4.2 Common User Interface Requirements

- Implement the UI structure and elements defined in this specification.
- Ensure major screens render correctly on desktop and mobile widths.
- Display budget-in, budget-out, impact, and estimated hours with consistent visual rules.
- Do not display monitoring metrics such as progress rate, delay rate, or productivity evaluation.
- Make primary operations executable within three clicks.
- Represent states with labels and icons, not color alone.

### 4.3 Screen-specific Visual Design

#### 4.3.1 Project Select Screen

This diagram shows the external appearance of the project selection screen.

| Element | Description |
| --- | --- |
| Project search | Search area for project filtering |
| Project list | List area for selectable projects |
| Project summary | Summary area for the selected project |

```plantuml
!include ../ui/screen_project_select.puml
```

#### 4.3.2 Sprint Workspace Screen

This diagram shows the external appearance of the sprint workspace screen.

| Element | Description |
| --- | --- |
| Budget-in tasks | Tasks selected within budget |
| Budget-out tasks | Deferred tasks outside budget |
| Task editing | Editing area for task details |

```plantuml
!include ../ui/screen_sprint_workspace.puml
```

#### 4.3.3 Resource Settings Screen

This diagram shows the external appearance of the resource settings screen.

| Element | Description |
| --- | --- |
| Resource table | List of configured resources |
| Resource edit form | Edit area for one resource |

```plantuml
!include ../ui/screen_resource_settings.puml
```

#### 4.3.4 Working-Day Calendar Screen

This diagram shows the external appearance of the working-day calendar screen.

| Element | Description |
| --- | --- |
| Calendar grid | Monthly calendar area |
| Calendar control area | Calendar operation controls |

```plantuml
!include ../ui/screen_calendar_settings.puml
```

#### 4.3.5 Carry-Over Review Dialog

This diagram shows the external appearance of the carry-over review dialog.

| Element | Description |
| --- | --- |
| Deferred task review | Review area for deferred tasks |
| Carry-over decision input | Input area for carry-over decisions |

```plantuml
!include ../ui/screen_carryover_dialog.puml
```
