# STORY-08 IAM Portal IaC Refactor (Bicep → Terraform)

- Goal: Refactor the IAM Portal infrastructure deployment from a Bicep-based build to Terraform as the supported/acceptable IaC tool, while maintaining functional parity and enabling consistent pipeline automation.
- Estimate: 2–3 weeks (TBD)
- Owner: TBD
- Dependencies: STORY-01 ADO Permissions Change; STORY-05 Terraform and IaC Update (patterns/templates/state strategy)

## Scope
- Convert the IAM Portal infrastructure definitions currently authored in Bicep into Terraform.
- Align with the program’s Terraform repo layout, module conventions, and pipeline templates.
- Ensure telemetry, alerting, and operational dependencies remain intact after migration.

## Acceptance Criteria
- IAM Portal infrastructure is represented in Terraform (no Bicep required for standard deployments).
- Terraform state backend and locking strategy are implemented and documented.
- CI/CD supports `plan` on PR and controlled `apply` on approved merges, consistent with governance.
- Migration approach is defined and executed (import existing resources or recreate with controlled cutover), with rollback documented.
- Deployment parity validated in at least one lower environment (dev/test) before production cutover.
- Documentation/runbook updated for day-2 operations and future changes.

## Tasks
- [TASK-08-01](../tasks/TASK-08-01-iam-portal-bicep-to-terraform.md)
