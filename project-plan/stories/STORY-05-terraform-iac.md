# STORY-05 Terraform and IaC Update

- Goal: Modernize IaC with per-service repos, generic templates, and tag-based pipeline references after permissions are in place.
- Estimate: 4 weeks (Tony)
- Owner: Tony
- Dependencies: STORY-01 ADO Permissions Change, STORY-04 New Release Pipeline
- Acceptance Criteria:
  - Terraform/IaC repos separated per service and versioned together via tags.
  - Top-level definitions moved into service-scoped repos.
  - Pipelines consume IaC directly from tagged releases; cross-ADO org copies removed.
  - Generic templates available for terraform init/plan/apply and ArgoCD deployment.
- Tasks:
  - [TASK-05-01](../tasks/TASK-05-01-service-scoped-iac-repos.md)
  - [TASK-05-02](../tasks/TASK-05-02-generic-iac-templates.md)
  - [TASK-05-03](../tasks/TASK-05-03-pipeline-tag-consumption.md)
  - [TASK-05-04](../tasks/TASK-05-04-remove-cross-org-copies.md)
