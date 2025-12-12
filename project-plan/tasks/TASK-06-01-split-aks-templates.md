# TASK-06-01 Split AKS Templates into Dedicated Repo

- Story: STORY-06 Split Infrastructure AKS Templates
- Estimate: 3 days (within IaC 4-week window)
- Dependencies: STORY-05 Terraform and IaC Update
- Steps:
  1. Identify shared AKS YAML/scripts currently reused across services.
  2. Create dedicated repo for AKS templates with versioning enabled.
  3. Update pipelines to consume templates from the new repo.
  4. Validate deployments using the new template source.
- Deliverables:
  - Dedicated AKS template repository with initial version tag.
  - Pipelines updated to pull from the new repo.
