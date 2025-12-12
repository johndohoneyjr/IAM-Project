# STORY-06 Split Infrastructure AKS Templates

- Goal: Separate shared AKS infrastructure YAML/script templates into their own repo for independent versioning.
- Estimate: Included within STORY-05 (no additional weeks beyond the 4-week IaC allocation)
- Owner: Tony (with IaC support)
- Dependencies: STORY-05 Terraform and IaC Update
- Acceptance Criteria:
  - Shared AKS YAML/script templates moved to a dedicated repo.
  - Versioning strategy documented and aligned with service-scoped IaC tagging.
  - Pipelines reference the new repo for AKS template consumption.
- Tasks:
  - [TASK-06-01](../tasks/TASK-06-01-split-aks-templates.md)
