# CarrySprint Documentation Project

CarrySprint is a tool for budget-first agile development.
It helps teams decide which tasks to execute now and which tasks to carry over to the next sprint.

Unlike traditional backlog and progress management tools, CarrySprint does not aim to:

- Measure productivity
- Perform management control
- Monitor progress
- Assume everything must be completed

CarrySprint focuses on one goal: select only high-impact tasks within a limited budget.

## Document Structure

The documentation is organized by intent so both humans and AI tools can locate the right source quickly.

```text
/docs
  /requirements
    software_requirements_specification.md
  /architecture
    system_architecture.puml
    software_architecture.puml
    database_design.puml
  /software-method-design
    software_method_design.md
    /diagrams
      /dfd
        dfd_level1.puml
      /activity
        activity_uc02_get_sprint_workspace.puml
        activity_uc06_apply_carryover.puml
  /software-detailed-design
    software_detailed_design.md
  /ui
    screen_design.puml
    screen_project_select.puml
    screen_sprint_workspace.puml
    screen_resource_settings.puml
    screen_calendar_settings.puml
    screen_carryover_dialog.puml
  /glossary
    terms.md
```

- Start with `docs/requirements/software_requirements_specification.md` for the full specification.
- Use `docs/architecture` for structural diagrams.
- Use `docs/software-method-design` for phase output documents of software method design.
- Use `docs/software-detailed-design` for phase output documents of software detailed design.
- Use `docs/ui` for screen flow and screen-specific mock diagrams.
- Use `docs/glossary/terms.md` for domain vocabulary.

## Problems CarrySprint Solves

### 1. Budget-First Development

In modern software development, budget often becomes a stronger constraint than schedule.
CarrySprint is designed for this premise.

### 2. Infinite Task Growth

In the AI era, candidate tasks grow rapidly.
Teams must explicitly choose not to do some tasks now and prune the list.

### 3. Productivity Is Not a Performance Metric

As AI takes over more implementation work, productivity becomes less suitable as a human evaluation metric.
CarrySprint does not use productivity to evaluate people or teams.

### 4. Difference from Management-Centric Tools

Most existing tools assume all tasks should eventually be completed.
CarrySprint focuses on selection support rather than execution management.

## Core Principles

- Budget first: Decide remaining work-hours first.
- Impact driven: Evaluate task value by impact.
- Pruning is not deletion: Defer tasks and re-evaluate in the next sprint.
- Task-level forecasting: Forecast per task, not by rough global averages.
- User-driven decisions: The tool does not decide; the user decides.
- No management UI: Do not include progress, delay, or productivity-evaluation UI.

## UI Structure (Initial Draft)

### 1. Budget and Coefficient Display

- Remaining work-hours: 12h
- Productivity coefficient: 0.82

CarrySprint uses productivity only as a calculation aid.

### 2. Task-Level Forecasting

```text
---------- In Budget ----------
□ Task A (4.1h) impact: High
□ Task B (3.2h) impact: Medium

---------- Out of Budget ----------
□ Task C (12.2h) impact: Low
□ Task D (15.8h) impact: High (In Progress)
```

CarrySprint visualizes execution feasibility through in-budget and out-of-budget sections.
This classification does not deny task value.

### 3. Meaning of Pruning

Tasks that move out of budget are not discarded; teams carry them over.
Teams re-evaluate them in the next sprint and bring them back in-budget when priorities change.
