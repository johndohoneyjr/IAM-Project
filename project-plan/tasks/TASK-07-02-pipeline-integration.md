# TASK-07-02 Pipeline Integration

- Story: STORY-07 Azure DevOps Observability with Dynatrace
- Estimate: 5 days
- Dependencies: TASK-07-01 Dynatrace Setup and Configuration
- Steps:
  1. Integrate Dynatrace tasks into IAM service build pipeline templates.
  2. Configure pipeline metadata tags and labels for better tracking and filtering (service, repo, pipeline, stage, environment, branch, run reason, run id).
  3. Implement/emit metrics needed for the KPI catalog (success rate by stage; duration by stage; queue time; first-time pass where applicable).
  4. Add Dynatrace monitoring to release pipeline with stage-level visibility.
  5. Test integration across all IAM pipeline variants (dev, staging, production).
  6. Validate that pipeline failures and performance issues are captured in Dynatrace.
- Deliverables:
  - Updated pipeline YAML templates with Dynatrace integration steps.
  - Pipeline metadata properly tagged for Dynatrace filtering and analysis.
  - Verified data collection from all pipeline stages and environments.

