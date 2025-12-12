# IAM Project Repository

This repo captures the IAM pipeline refactoring initiative and the POC plan agreed during the planning meeting. Use this README to navigate the key assets.

## Folder Map
- `refactor-plan/` — Core architectural and staged refactor guidance (monolith → service pipelines), diagrams, and stage-specific docs for the IAM pilot and beyond.
- `project-plan/` — User stories and executable tasks derived from the planning meeting (IAM POC). Each story links to tasks with owners, estimates, and dependencies.
- `sprint-plan/` — 2-week sprint schedule with staffing profile and sequencing of the critical stories (starts with permissions and code-build guardrails).
- `.github/` — GitHub configuration and Copilot instructions.

## IAM POC Planning Assets
- Planning meeting outputs are reflected in `project-plan/` (stories/tasks) and `sprint-plan/` (timeboxed execution). Estimates: ~13 weeks single engineer; 8–9 weeks with 5 dedicated engineers.
- Critical path: ADO permissions gate → code-build hardening → container build/packaging → release pipeline → Terraform/IaC updates (+ AKS template split).

## Reading Order
1) Skim `refactor-plan/README.md` and architecture docs for context.
2) Review `project-plan/` stories/tasks to understand scope and acceptance criteria.
3) Check `sprint-plan/README.md` for the current 2-week cadence and staffing assumptions.
