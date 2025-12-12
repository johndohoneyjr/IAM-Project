# TASK-05-04 Remove Cross-ADO Org Copies

- Story: STORY-05 Terraform and IaC Update
- Estimate: 2 days
- Dependencies: TASK-05-03 Pipeline Tag Consumption
- Steps:
  1. Identify cross-ADO org IaC copies used today.
  2. Remove redundant copies and point pipelines to service-scoped repos via tags.
  3. Confirm no downstream jobs rely on old copies.
- Deliverables:
  - Deprecated cross-org copies removed or archived.
  - Pipelines verified against new IaC source locations.
