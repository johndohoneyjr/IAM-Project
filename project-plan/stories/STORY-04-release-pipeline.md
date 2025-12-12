# STORY-04 New Release Pipeline

- Goal: Create a new release pipeline that orchestrates container promotion using generic templates and service-specific approvals.
- Estimate: 2 weeks (Tony/Jeff)
- Owners: Tony and Jeff
- Dependencies: STORY-03 Container Build and Packaging
- Acceptance Criteria:
  - Release pipeline calls generic templates for container promotion and deployment.
  - Manual approval gates are service-specific and mapped to named approvers.
  - Pipeline name and artifacts clearly reflect IAM service scope.
- Tasks:
  - [TASK-04-01](../tasks/TASK-04-01-design-release-pipeline.md)
  - [TASK-04-02](../tasks/TASK-04-02-manual-approvals.md)
  - [TASK-04-03](../tasks/TASK-04-03-wire-generic-templates.md)
