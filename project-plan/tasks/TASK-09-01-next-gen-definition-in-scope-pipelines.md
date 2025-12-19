# TASK-09-01 Define “Next-Gen” Pipelines and In-Scope Pipelines List (4/15/2026)

- Story: STORY-09 Define “Next-Gen” Pipelines and Lock In-Scope Pipelines (4/15/2026)
- Estimate: 3–5 days
- Dependencies: STORY-01 ADO Permissions Change

## Objective
Create a clear definition of “Next-Gen” pipelines and a management-approved list of in-scope pipelines for the **4/15/2026** deadline.

## Steps
1. Define “Next-Gen” pipeline **minimum requirements** (draft):
   - Governance: PR policies/checks, approvals model, branch strategy expectations.
   - Build: standardized build template usage, repeatable artifact creation, versioning.
   - Security: required scans and minimum pass criteria; how exceptions are handled.
   - Release/Promotion: promotion strategy, environment gating, rollback expectations.
   - Observability: required metrics (success rate, duration, stage timing, queue time), dashboards, alerting.
   - Compliance/Traceability: tagging, audit trails, change records where required.
2. Review the draft with key stakeholders (SWAT leads, platform/ops, security) and incorporate feedback.
3. Identify candidate pipelines across IAM and related repos:
   - Build pipelines (CI)
   - Release pipelines (artifact promotion)
   - Deployment pipelines (if relevant to the 4/15/2026 milestone and owned scope)
4. Produce the “In-Scope Pipelines for 4/15/2026” list, including dependencies and constraints:
   - External dependencies that can cause failures (e.g., ServiceNow) and their interfaces (API/webhook/connector/manual gate).
   - Required approvals and lead times.
   - Ownership (team + on-call/ops contact).
5. Hold a management review and record decisions:
   - Confirm in-scope list
   - Document exclusions and rationale
   - Identify any scope risks that require escalation
6. Publish final documents and link them from Sprint/Story planning assets.

## Deliverables
- Documented “Next-Gen” pipeline definition with minimum requirements.
- Management-approved “In-Scope Pipelines for 4/15/2026” list.

## In-Scope Pipelines List (to be populated)
| Pipeline | Repo/Service | Type (Build/Release/Deploy) | Environments | Owner | Key Dependencies | Notes / Rationale |
|---|---|---|---|---|---|---|
| TBD | TBD | TBD | TBD | TBD | TBD | TBD |
