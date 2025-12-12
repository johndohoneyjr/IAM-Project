# TASK-03-04 Remove Legacy Release Option

- Story: STORY-03 Container Build and Packaging
- Estimate: 2 days
- Dependencies: TASK-03-03 Centralize Service Dependencies
- Steps:
  1. Identify legacy "release" option usage in IAM build/package steps.
  2. Remove or disable release toggle in pipeline and scripts.
  3. Validate build and promotion still succeed without the legacy option.
- Deliverables:
  - Updated pipeline with release option removed.
  - Validation results confirming unaffected build and promotion.
