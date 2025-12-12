# STORY-01 ADO Permissions Change

- Goal: Establish prerequisite Azure DevOps permissions so pipeline refactoring work can proceed without blockers.
- Estimate: Pre-work (in-flight); treat as gate before other stories begin.
- Owner: Chuck (with support from SWAT team)
- Dependencies: None
- Acceptance Criteria:
  - Engineers can create/update pipelines and service connections without requesting ad-hoc elevation.
  - Least-privilege roles mapped to build, release, and IaC pipelines.
  - Approval workflows documented for services in scope.
- Tasks:
  - [TASK-01-01](../tasks/TASK-01-01-ado-permissions.md)
