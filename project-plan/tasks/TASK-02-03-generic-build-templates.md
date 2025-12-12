# TASK-02-03 Generic Code Build Templates

- Story: STORY-02 Code Build Pipeline
- Estimate: 1 week
- Dependencies: TASK-02-02 PR Policies and Checks for .NET Changes
- Steps:
  1. Draft reusable build template covering code build, unit tests, and security scan steps.
  2. Parameterize service name, test paths, and scan severity thresholds.
  3. Validate template within IAM pipeline; ensure compatibility with Stage 1 templates.
  4. Publish template usage doc and example YAML snippet.
- Deliverables:
  - Checked-in build template ready for reuse across services.
  - Example IAM pipeline snippet demonstrating template consumption.
