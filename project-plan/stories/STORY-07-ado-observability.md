# STORY-07 Azure DevOps Observability with Dynatrace

- Goal: Implement comprehensive observability and monitoring for Azure DevOps pipelines using Dynatrace to track pipeline performance, identify bottlenecks, and ensure build/release reliability.
- Estimate: 3 weeks (TBD MSFT Dynatrace SME, Steve Rogers)
- Owner: TBD MSFT Dynatrace SME and Steve Rogers
- Dependencies: STORY-01 ADO Permissions Change
- Acceptance Criteria:
  - Dynatrace extension installed and configured in Azure DevOps organization.
  - Pipeline metrics (duration, success/failure rates, stage timings) flowing to Dynatrace dashboards.
  - Custom dashboards created for IAM service pipelines with alerting on key performance indicators.
  - Documentation for monitoring setup and runbook for troubleshooting pipeline issues.
  - Team trained on using Dynatrace for pipeline analysis and optimization.
- Tasks:
  - [TASK-07-01](../tasks/TASK-07-01-dynatrace-setup.md)
  - [TASK-07-02](../tasks/TASK-07-02-pipeline-integration.md)
  - [TASK-07-03](../tasks/TASK-07-03-dashboards-alerts.md)
  - [TASK-07-04](../tasks/TASK-07-04-documentation-training.md)
