# TASK-05-02 Generic IaC Templates

- Story: STORY-05 Terraform and IaC Update
- Estimate: 4 days
- Dependencies: TASK-05-01 Service-Scoped IaC Repositories
- Steps:
  1. Build generic templates for `terraform init/plan/apply` and ArgoCD application deployment.
  2. Parameterize backend config, workspaces, and app identifiers for each service.
  3. Validate templates against IAM IaC repo.
- Deliverables:
  - Reusable IaC templates stored with service repos.
  - Example IAM pipeline invoking the templates.
