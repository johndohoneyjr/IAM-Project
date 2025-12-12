# TASK-01-01 ADO Permissions Baseline

- Story: STORY-01 ADO Permissions Change
- Estimate: 3 days (gating)
- Dependencies: None
- Steps:
  1. Inventory current ADO permissions for pipeline authors, service connections, and variable groups.
  2. Define least-privilege roles for build, release, and IaC pipelines (align with IAM pilot scope).
  3. Implement role changes and document approvers for each service.
  4. Validate engineers can run builds/releases without escalation.
- Deliverables:
  - Documented permission matrix stored with pipeline governance assets.
  - Confirmed access for engineers working on IAM refactor.
