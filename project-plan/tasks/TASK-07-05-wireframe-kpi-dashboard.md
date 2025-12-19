# TASK-07-05 Wireframe KPI / Metrics Dashboard

- Story: STORY-07 Azure DevOps Observability with Dynatrace
- Estimate: 2 days
- Dependencies: TASK-07-00 Metric Identification and SLO Definitions

## Objective
Produce a wireframe for the pipeline KPI dashboard so implementation in Dynatrace is straightforward and consistent across services.

## Steps
1. Confirm intended consumers and use cases:
   - Pipeline engineers (debugging and trend analysis)
   - Ops/on-call (alerts and SLA/SLO posture)
   - Leadership reporting (delivery health over time)
2. Define dashboard navigation structure (wireframe-level):
   - Overview (fleet view)
   - Service view
   - Pipeline/run drill-down
   - Dependency health (3rd-party services like ServiceNow)
3. Wireframe **required tiles/sections**:
   - Success Rate (overall + by stage: Build/Test/Security/Deploy)
   - Duration (commit → artifact) + stage timing breakdown
   - First-time pass / first run time
   - Run statistics (runs/day, retry rate, cancellation rate, queue time)
   - Commit-to-merge time
   - Security scan pass rate + critical block count
   - Change size distribution
   - Deployment frequency per service
   - External dependency health indicators (e.g., ServiceNow API success/latency, connector failures)
4. Define drill-down requirements per tile:
   - Filters/segments: service, repo, pipeline, environment, branch, time window.
   - Links: from KPI anomaly → specific runs → stage logs.
5. Map each tile to an expected data source:
   - Azure DevOps run/stage metadata
   - Dynatrace pipeline events/metrics
   - ServiceNow health/telemetry (if available) or synthetic/API checks
6. Review wireframe with stakeholders and capture feedback for final implementation.

## Deliverables
- Wireframe (documented in markdown) describing dashboard layout, tiles, drill-down paths, and required data sources.
- Agreed MVP dashboard spec to implement in TASK-07-03.
