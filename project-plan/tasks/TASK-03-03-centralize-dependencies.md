# TASK-03-03 Centralize Service Dependencies

- Story: STORY-03 Container Build and Packaging
- Estimate: 4 days
- Dependencies: TASK-03-02 Break Out Package to Service
- Steps:
  1. Inventory IAM service dependencies and artifacts currently spread across pipeline steps.
  2. Push build-time dependencies to central storage account for reuse.
  3. Update pipeline steps to pull dependencies from the central location.
  4. Validate deterministic builds using centralized inputs.
- Deliverables:
  - Central storage path documented and used by IAM builds.
  - Deterministic build output confirmed.
