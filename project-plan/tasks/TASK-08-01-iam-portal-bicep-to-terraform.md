# TASK-08-01 Migrate IAM Portal IaC from Bicep to Terraform

- Story: STORY-08 IAM Portal IaC Refactor (Bicep → Terraform)
- Estimate: 2–3 weeks
- Dependencies: STORY-05 Terraform and IaC Update (repo patterns/templates/state strategy)

## Objective
Replace the IAM Portal’s Bicep-based infrastructure deployment with Terraform while preserving functional parity and enabling standard CI/CD workflows (plan on PR, controlled apply on merge).

## Steps
1. Discovery and parity baseline
   - Identify the current IAM Portal Bicep entrypoints and resources created.
   - Capture required environments (dev/test/prod) and naming conventions.
   - Confirm current backend/state approach (if any) and desired Terraform backend/locking.
2. Terraform design
   - Decide module structure and repo layout consistent with STORY-05 conventions.
   - Define variables, outputs, and environment separation strategy.
   - Define tagging standards for traceability.
3. State and migration strategy
   - Choose approach per environment:
     - **Import strategy** (preferred when preserving existing resources): import resources into Terraform state.
     - **Recreate strategy** (only if safe): controlled cutover with rollback plan.
   - Document rollback steps and safety checks.
4. Implement Terraform
   - Write Terraform configuration to represent the IAM Portal infrastructure.
   - Ensure idempotency and consistent drift detection.
5. Pipeline integration
   - Wire Terraform into the standard pipelines/templates (init/validate/plan on PR; apply gated on merge).
   - Ensure secrets/credentials are handled via approved mechanisms.
6. Validation
   - Validate in a lower environment first (dev/test): plan is clean, apply produces expected resources, and portal functions.
   - Capture drift checks and post-deploy verification steps.
7. Cutover and documentation
   - Execute production cutover (if in-scope) with approvals.
   - Update runbooks and operational documentation (day-2 changes, troubleshooting, rollback).

## Deliverables
- Terraform configuration that fully replaces the IAM Portal Bicep deployment for standard deployments.
- Terraform backend/state configuration and documented locking strategy.
- CI/CD integration for plan/apply consistent with governance.
- Migration documentation (import/cutover/rollback) and verification checklist.
