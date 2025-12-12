# TASK-03-05 Generic Container Templates

- Story: STORY-03 Container Build and Packaging
- Estimate: 3 days
- Dependencies: TASK-03-04 Remove Legacy Release Option
- Steps:
  1. Draft reusable container build/package template covering digest capture and artifact publishing.
  2. Parameterize registry, service name, and dependency source.
  3. Validate template inside IAM pipeline; ensure compatibility with Stage 1 step templates.
  4. Publish usage doc and sample invocation.
- Deliverables:
  - Reusable container build template checked in.
  - Documentation for adopting the template across services.
