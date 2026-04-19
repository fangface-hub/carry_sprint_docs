# CarrySprint Software Requirements Specification

## 1. System Architecture

This diagram shows the runtime structure of CarrySprint.

| Element | Description |
| --- | --- |
| Client | Browser-based user access |
| Server | Go-based server runtime |
| Inter-process communication path | ZeroMQ-based request/response path |
| Project storage | SQLite-based per-project storage |

```plantuml
!include ../architecture/system_architecture.puml
```

### 1.1 Design Writing Policy

- Do not use `or` to express alternatives.
  - Bad example: `Use HTTP/1.1 or HTTP/2 over HTTPS.`
  - Good example:
    - Split guideline: Keep the design element in each item. Use the following options:
      - `Use HTTP/1.1 over HTTPS.`
      - `Use HTTP/2 over HTTPS.`
- Do not use `and` to join parallel elements.
  - Bad example: `Define conditions and behavior explicitly.`
  - Good example:
    - Split guideline: Keep the design context in each item. Cover all of the following:
      - `Define conditions explicitly.`
      - `Define behavior explicitly.`
- Do not use `except` to define exclusion.
- Do not use passive voice.
- Do not use `it is necessary`.
- Write one sentence for one meaning.
- Define conditions explicitly.
- Define behavior explicitly.
- Treat the `else` clause of a Go `if` statement as an exception path. Do not write assumed normal handling in an `else` clause.
- Treat the `default` clause of a Go `switch` statement as an exception path. Do not write assumed normal handling in a `default` clause.

## 2. Software Architecture

This diagram shows the software component structure of CarrySprint.

| Element | Description |
| --- | --- |
| Browser client | User-facing web client |
| API gateway | HTTP entry point with request routing boundary |
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
- Manage resource information in SQLite.
- Manage the working-day calendar in SQLite.
- Manage user information in SQLite.
- Manage project role assignments in SQLite.

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
- Exclude progress-monitoring UI.
- Focus on selection.
- Focus on adjustment.
- Center the screen layout on budget-in visualization.
- Center the screen layout on budget-out visualization.

### 4.2 Common User Interface Requirements

#### 4.2.1 UI Placement URL

The browser client shall place each major UI at the following URL.

| UI | URL | Purpose |
| --- | --- | --- |
| Top Page | `/` | Entry point of the browser client |
| Sign-In Screen | `/signin` | User sign-in UI |
| Project Select Screen | `/projects` | Project selection UI |
| Project Register Screen | `/projects/new` | Project registration UI |
| Sprint Workspace Screen | `/projects/{project_id}/sprints/{sprint_id}/workspace` | Sprint work review UI for one project, one sprint |
| Resource Settings Screen | `/projects/{project_id}/resources` | Resource settings UI for one project |
| Working-Day Calendar Screen | `/projects/{project_id}/calendar` | Working-day calendar UI for one project |
| User Management Screen | `/users` | User registration, user update, user deletion, project role assignment UI |
| Carry-Over Review Dialog | `/projects/{project_id}/sprints/{sprint_id}/workspace?dialog=carryover` | Carry-over review UI on sprint workspace |

- Use stable URLs for direct browser access.
- Use path parameters for project context.
- Use path parameters for sprint context.
- Open the carry-over review dialog on the sprint workspace URL.
- Use the `dialog=carryover` query parameter for the carry-over review dialog state.

- Implement the UI structure defined in this specification.
- Implement the UI elements defined in this specification.
- Provide a top page as a major screen.
- Provide a sign-in screen as a major screen.
- Display top-page menu entries as buttons.
- Display the sign-in screen when the user is not signed in.
- Display the requested UI when the user signs in successfully.
- Sign out the current user from the top page.
- Display the sign-in screen after sign-out completes.
- Support locale settings per user.
- Allow the user to specify locale from user settings.
- Support menu visibility settings per user.
- Display only enabled menu buttons for the signed-in user.
- Ensure major screens render correctly on desktop widths.
- Ensure major screens render correctly on mobile widths.
- Display the following values with consistent visual rules:
  - Budget-in
  - Budget-out
  - Impact
  - Estimated hours
- Do not display the following monitoring metrics:
  - Progress rate
  - Delay rate
  - Productivity evaluation, etc.
- Make primary operations executable within three clicks.
- Represent states with the following methods:
  - Labels
  - Icons
  - Not color alone
- Resolve the default locale from explicit user settings when the user specifies locale.
- Resolve the default locale from the client language plus client region when locale configuration is available.
- Resolve the default locale from the client region when no language plus region match exists.
- Resolve the default locale from the client language when no region-based match exists.

### 4.3 Screen-specific Visual Design

#### 4.3.1 Sign-In Screen

This diagram shows the external appearance of the sign-in screen.

| Element | Description |
| --- | --- |
| User ID input | Input area for user ID |
| Password input | Input area for password |
| Sign-in action | Action area for sign-in execution |

```plantuml
!include ../ui/screen_sign_in.puml
```

#### 4.3.2 Project Select Screen

This diagram shows the external appearance of the project selection screen.

| Element | Description |
| --- | --- |
| Project search | Search area for project filtering |
| Project list | List area for selectable projects |
| Project summary | Summary area for the selected project |

```plantuml
!include ../ui/screen_project_select.puml
```

#### 4.3.3 Top Page Screen

This diagram shows the external appearance of the top page screen.

| Element | Description |
| --- | --- |
| Menu area | Area for menu buttons |
| Menu visibility settings | Area for user-specific menu ON/OFF settings |
| Locale settings | Area for user-specific locale selection |

```plantuml
!include ../ui/screen_top_page.puml
```

#### 4.3.4 Sprint Workspace Screen

This diagram shows the external appearance of the sprint workspace screen.

| Element | Description |
| --- | --- |
| Budget-in tasks | Tasks selected within budget |
| Budget-out tasks | Deferred tasks outside budget |
| Task editing | Editing area for task details |

```plantuml
!include ../ui/screen_sprint_workspace.puml
```

#### 4.3.5 Resource Settings Screen

This diagram shows the external appearance of the resource settings screen.

| Element | Description |
| --- | --- |
| Resource table | List of configured resources |
| Resource edit form | Edit area for one resource |

```plantuml
!include ../ui/screen_resource_settings.puml
```

#### 4.3.6 Working-Day Calendar Screen

This diagram shows the external appearance of the working-day calendar screen.

| Element | Description |
| --- | --- |
| Calendar grid | Monthly calendar area |
| Calendar control area | Calendar operation controls |

```plantuml
!include ../ui/screen_calendar_settings.puml
```

#### 4.3.7 Carry-Over Review Dialog

This diagram shows the external appearance of the carry-over review dialog.

| Element | Description |
| --- | --- |
| Deferred task review | Review area for deferred tasks |
| Carry-over decision input | Input area for carry-over decisions |

```plantuml
!include ../ui/screen_carryover_dialog.puml
```

#### 4.3.8 User Management Screen

This diagram shows the external appearance of the user management screen.

| Element | Description |
| --- | --- |
| User list | List area for registered users |
| User edit form | Edit area for user registration/modification |
| Project role assignment | Assignment area for project administrator role, project assignee role |

```plantuml
!include ../ui/screen_user_management.puml
```

#### 4.3.9 Project Register Screen

This diagram shows the external appearance of the project registration screen.

| Element | Description |
| --- | --- |
| Basic information input | Input area for project name, key, description |
| Initial sprint input | Input area for initial sprint period and basic settings |
| Administrator assignment | Input area for initial project administrator |

```plantuml
!include ../ui/screen_project_register.puml
```

## 5. User Management Requirements

### 5.0 Initial User Requirements

- Prepare an initial user with the user ID `admin`.
- Set the initial password of the user ID `admin` to `admin`.

### 5.1 User Operation Requirements

- Register a user from the user management screen.
- Modify registered user information from the user management screen.
- Delete a registered user from the user management screen.

### 5.2 Project Role Requirements

- Assign an administrator role to a user for a specified project from the user management screen.
- Assign an assignee role to a user for a specified project from the user management screen.

## 6. Internationalization Requirements

### 6.1 Default Locale Resolution

- Change the default locale according to the client language.
- Change the default locale according to the client region.
- Use `en` as the default locale when locale configuration is not prepared.
