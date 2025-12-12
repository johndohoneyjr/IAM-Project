# TASK-05-01 Service-Scoped IaC Repositories

- Story: STORY-05 Terraform and IaC Update
- Estimate: 3 weeks
- Dependencies: STORY-01 ADO Permissions Change
- Steps:
  1. Create separate IaC repository per service and migrate IAM top-level definitions into the IAM repo.
  2. Ensure versioning strategy supports packaging repos together while remaining independently versioned.
  3. Update repo structure to align with refactor-plan architecture (terraform + ArgoCD assets per service).
- Deliverables:
  - Service-scoped IaC repository for IAM with migrated definitions.
  - Documented versioning approach for multi-service packaging.
