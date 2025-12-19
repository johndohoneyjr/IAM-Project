# STORY-07 Azure DevOps Observability with Dynatrace

- Goal: Implement comprehensive observability for Azure DevOps pipelines using Dynatrace to track delivery health KPIs, identify bottlenecks, and improve build/release reliability.
- Estimate: 3 weeks (TBD MSFT Dynatrace SME, Steve Rogers)
- Owner: TBD MSFT Dynatrace SME and Steve Rogers
- Dependencies: STORY-01 ADO Permissions Change

## KPI Scope (define first)
Metric identification and SLO/threshold definition is the first deliverable for this story. Suggested KPIs include:

- **First Run Time / First-Time Pass**: Detect “works on retry” behavior and flaky pipelines.
- **Run Statistics**: Runs/day, retry rate, cancellation rate, and queued time.
- **Pipeline Success Rate**: Successful runs / total runs; **95% goal**, **90% minimum**; track by stage (Build/Test/Security/Deploy).
- **Pipeline Duration (Commit → Deployable Artifact)**: Target <10–15 minutes for most services; warning >30 minutes; include queued vs execution and stage timings.
- **Commit-to-Merge Time**: Target <24 hours; elite <4 hours.
- **Security Scan Pass Rate**: % builds blocked by critical vulns/policy failures; time-to-remediate critical.
- **Change Size**: Lines changed per deploy/merge; trend metric.
- **Deployment Frequency**: Deploys per service per week; trend over time.

Also explicitly track **3rd-party services** that can cause pipeline failures (e.g., ServiceNow) including their interfaces (API/webhook/connector/manual gates) and alert thresholds.

## Acceptance Criteria
- KPI catalog exists with definitions, formulas, targets (including 95% goal / 90% minimum success rate), and alert thresholds.
- Dynatrace extension installed and configured in Azure DevOps organization.
- Pipeline metrics (duration, success/failure rates, stage timings, queue time) flow to Dynatrace.
- Dashboards implemented per wireframe, including external dependency health indicators (e.g., ServiceNow).
- Alerting and notification routing configured (Teams/email/ServiceNow as appropriate).
- Documentation + runbook for troubleshooting pipeline issues using Dynatrace data.
- Team trained on using Dynatrace for pipeline analysis and optimization.

## Tasks
  - [TASK-07-00](../tasks/TASK-07-00-metric-identification.md)
  - [TASK-07-01](../tasks/TASK-07-01-dynatrace-setup.md)
  - [TASK-07-02](../tasks/TASK-07-02-pipeline-integration.md)
  - [TASK-07-05](../tasks/TASK-07-05-wireframe-kpi-dashboard.md)
  - [TASK-07-03](../tasks/TASK-07-03-dashboards-alerts.md)
  - [TASK-07-04](../tasks/TASK-07-04-documentation-training.md)
