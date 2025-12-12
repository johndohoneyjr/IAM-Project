# STORY-02 Code Build Pipeline

- Goal: Refactor the code build pipeline with enforced PR policies, automated tests, and security scans for .NET changes.
- Estimate: 4 weeks (Jeff)
- Owner: Jeff
- Dependencies: STORY-01 ADO Permissions Change
- Acceptance Criteria:
  - PRs for .NET changes require successful unit tests and security scans before merge.
  - Code build pipeline process reviewed and approved.
  - Generic code-build templates available for reuse.
- Tasks:
  - [TASK-02-01](../tasks/TASK-02-01-review-code-build.md)
  - [TASK-02-02](../tasks/TASK-02-02-pr-policies-and-checks.md)
  - [TASK-02-03](../tasks/TASK-02-03-generic-build-templates.md)
