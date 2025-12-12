# IAM POC Sprint Plan (2-week sprints)

This plan sequences the IAM POC stories into four 2-week sprints. Critical gating work (ADO permissions) and pipeline hardening begin in Sprint 1 to unblock the rest of the refactor. Staffing assumes the initial SWAT profile called out in the outline.

## Staffing Profile
- Jeff: Code build / pipeline refactor
- Tony: Release pipeline / IaC
- Chuck: ADO permissions
- SWAT-Eng-1: Containers and packaging
- SWAT-Eng-2: IaC support / AKS templates

## Sprint Goals and Scope
- **Sprint 1 (Weeks 1-2)**: Clear permissions gate and lock in code-build guardrails.
  - Stories/Tasks: STORY-01; STORY-02 tasks TASK-02-01, TASK-02-02; start TASK-02-03 draft.
  - Deliverables: Permissions matrix applied; PR policies enforcing unit tests + security scans; draft generic build template.
- **Sprint 2 (Weeks 3-4)**: Finish build templates and modernize container build/package flow.
  - Stories/Tasks: Complete TASK-02-03; STORY-03 tasks TASK-03-01, TASK-03-02, TASK-03-03.
  - Deliverables: Reusable build template validated in IAM; IAM packaging split per service with centralized deps.
- **Sprint 3 (Weeks 5-6)**: Finalize container work and stand up the new release pipeline.
  - Stories/Tasks: STORY-03 tasks TASK-03-04, TASK-03-05; STORY-04 tasks TASK-04-01, TASK-04-02, TASK-04-03.
  - Deliverables: Legacy release option removed; generic container templates in use; release pipeline with approvals wired to templates.
- **Sprint 4 (Weeks 7-8)**: IaC modernization and AKS template split.
  - Stories/Tasks: STORY-05 tasks TASK-05-01, TASK-05-02, TASK-05-03, TASK-05-04; STORY-06 TASK-06-01.
  - Deliverables: Service-scoped IaC repos with tagged consumption; generic IaC templates; cross-org copies removed; AKS templates in dedicated repo.

## Notes and Dependencies
- STORY-01 must complete before Sprints 2-4 work is merged; treat as Sprint 1 exit criteria.
- STORY-06 work rides inside STORY-05 and can start late Sprint 3 if IaC repo layout is ready.
- If capacity drops below 5 engineers, extend Sprint 4 into Sprint 5 to finish IaC and AKS split.

## Tracking
- Reference source stories in `project-plan/stories/` and tasks in `project-plan/tasks/` for detailed steps and acceptance criteria.
- Update sprint burndown using the story/task IDs above to maintain traceability to the original plan.
