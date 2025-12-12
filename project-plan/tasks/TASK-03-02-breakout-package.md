# TASK-03-02 Break Out Package to Service

- Story: STORY-03 Container Build and Packaging
- Estimate: 1 week
- Dependencies: TASK-03-01 Review Container Build Process
- Steps:
  1. Split packaging steps so IAM artifacts are built and versioned independently.
  2. Update build scripts to use IAM-specific Dockerfile and context.
  3. Validate build output and digest capture for IAM service.
- Deliverables:
  - Service-scoped packaging workflow for IAM.
  - Verified container digest captured for promotion.
