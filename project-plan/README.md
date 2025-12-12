# IAM POC Project Plan

This plan translates the planning meeting into actionable user stories and tasks for the IAM pipeline refactor. Estimates follow the source outline (13 weeks for one engineer; 8-9 weeks if 5 dedicated engineers on the SWAT team). All work is scoped to the IAM pilot and aligned with the refactor-plan documentation.

## Folder Structure
- `stories/` — user stories with goals, estimates, acceptance criteria, and linked tasks.
- `tasks/` — executable tasks with steps and deliverables, keyed to their parent story.

## Work Breakdown (Stories)
- STORY-01: ADO Permissions Change (gating, in-flight)
- STORY-02: Code Build Pipeline (4 weeks, Jeff)
- STORY-03: Container Build and Packaging (3 weeks, Jeff/Tony)
- STORY-04: New Release Pipeline (2 weeks, Tony/Jeff)
- STORY-05: Terraform and IaC Update (4 weeks, Tony)
- STORY-06: Split Infrastructure AKS Templates (within Story 05 effort)

## Estimates and Resourcing
- Baseline: 13 weeks for a single engineer (4 + 3 + 2 + 4; ADO permissions pre-work).
- With 5 dedicated engineers: target 8-9 weeks by parallelizing stories after ADO permissions are ready.

## Execution Notes
- Complete STORY-01 (ADO permissions) before starting other stories.
- Story sequencing: Story 02 → Story 03 → Story 04 → Story 05, with Story 06 running inside Story 05.
- Each task lists dependencies to support parallel work once gates are cleared.

## File Map
- Stories: `stories/STORY-01-ado-permissions.md`, `stories/STORY-02-code-build-pipeline.md`, `stories/STORY-03-container-build-package.md`, `stories/STORY-04-release-pipeline.md`, `stories/STORY-05-terraform-iac.md`, `stories/STORY-06-split-infrastructure-aks.md`
- Tasks: see `tasks/` for TASK-01-01 through TASK-06-01, aligned with the story IDs.

## Source Alignment
- Outline and estimates derived directly from planning meeting notes.
- Terminology and sequencing align with existing refactor-plan docs (`stage-1` foundations, `stage-2` IAM pipeline focus).
