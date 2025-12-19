# TASK-07-00 Metric Identification and SLO Definitions

- Story: STORY-07 Azure DevOps Observability with Dynatrace
- Estimate: 2 days
- Dependencies: STORY-01 ADO Permissions Change

## Objective
Define *what* we measure and *why* before wiring tooling. This task produces a concrete, agreed KPI catalog with formulas, targets, and alert thresholds, including upstream/downstream dependencies (e.g., ServiceNow) that impact delivery health.

## Steps
1. Confirm scope of pipelines in-scope for reporting (build → deployable artifact, and any release stages owned by the SWAT scope).
2. Define the **KPI catalog** (names, formulas, units, and collection window) for pipeline effectiveness:
   - **First Run Time (First-Time Pass)**
     - Measure: Duration + outcome for the *first run after a change is merged* (or the first CI run for a PR, depending on agreed definition).
     - Purpose: Detect flaky pipelines and “works on retry” behavior.
   - **Run Statistics**
     - Measure: Total runs, success/fail, retries, cancellation rate, queued time distribution.
     - Purpose: Baseline noise and capacity constraints.
   - **Pipeline Success Rate**
     - Measure: Successful runs / total runs.
     - Target: **≥95% goal**, **≥90% minimum** (success metric).
     - Track by stage: Build, Test, Security, Deploy.
     - Notes: Low success rate usually means flaky tests, unstable dependencies, or brittle checks.
   - **Pipeline Duration (Commit → Deployable Artifact)**
     - Measure: Code push/merge → artifact published (or “deploy-ready” output).
     - Good: <10–15 minutes for most services; Warning: >30 minutes.
     - Breakdowns: queued time vs execution time; stage timings.
   - **Commit-to-Merge Time**
     - Measure: PR opened → PR merged (or first commit in PR → merge; decide and document).
     - Target: <24 hours; Elite: <4 hours.
     - Notes: Track reviewers, approval time, and policy gate times where possible.
   - **Security Scan Pass Rate**
     - Measure: % builds blocked by critical vulnerabilities and policy failures.
     - Track: Time-to-remediate critical findings.
   - **Change Size**
     - Measure: Lines changed per deploy (or per merge).
     - Notes: Smaller changes correlate with lower risk; use as trend, not a punitive metric.
   - **Deployment Frequency**
     - Measure: Deploys per service per week; trend over time.
3. Identify **critical 3rd-party services** impacting pipeline reliability and what “healthy” means:
   - Example: **ServiceNow** availability/latency affecting gates, approvals, ticket creation, or change records.
   - For each dependency, capture:
     - Interface type: API, webhook, email integration, connector, manual gate.
     - Failure modes: outage, auth failures, latency, rate limits.
     - Detection signals: synthetic check, API error rate, timeout rate, backlog/queue depth.
4. Define **alerting policy** aligned to KPIs and dependencies:
   - Thresholds (warning/critical), evaluation window (e.g., 15m/1h/24h), suppression rules.
   - Notification routing (Teams/email/ServiceNow) and ownership (who responds).
5. Specify required **tags/metadata** for correlation:
   - service name, repo, pipeline name, stage name, environment, branch, run reason (PR vs main), build id/run id.
6. Review/approve KPI catalog with stakeholders (SWAT leads + operations + security), and publish as the source of truth.

## Deliverables
- KPI catalog with definitions, formulas, and targets (including **95% goal / 90% minimum** success rate).
- Dependency inventory (e.g., ServiceNow) with interfaces and failure modes.
- Alert threshold matrix and routing plan.
- Required tagging/metadata specification for dashboards and drill-down.
