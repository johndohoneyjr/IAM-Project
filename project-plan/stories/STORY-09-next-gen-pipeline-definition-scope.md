# STORY-09 Define “Next-Gen” Pipelines and Lock In-Scope Pipelines (4/15/2026)

- Goal: Establish an agreed definition of “Next-Gen” pipelines and produce a management-approved list of pipelines that are in-scope for the **4/15/2026** deadline so the March/April execution plan can be finalized.
- Estimate: 1 week (can run in parallel with build template work)
- Owner: TBD (SWAT lead + governance/ops representative)
- Dependencies: STORY-01 ADO Permissions Change

## Scope
- Define “Next-Gen pipeline” criteria (capabilities, governance, telemetry, and operational expectations).
- Inventory candidate pipelines and determine which are in-scope for the 4/15/2026 deadline.
- Obtain management sign-off so the in-scope list can be used for Sprint/Story planning for March/April.

## Acceptance Criteria
- “Next-Gen” definition is documented and includes minimum required capabilities (gates/policies, artifact standards, security checks, release practices, observability/metrics).
- A single “In-Scope Pipelines for 4/15/2026” list exists with:
  - Pipeline name, repo/service, type (build/release/deploy), environments, owners
  - Rationale for inclusion/exclusion and major dependencies (e.g., ServiceNow)
  - Any required approvals/external process dependencies
- Management approval is recorded (date + approver) and the document is considered the source of truth.

## Tasks
- [TASK-09-01](../tasks/TASK-09-01-next-gen-definition-in-scope-pipelines.md)
