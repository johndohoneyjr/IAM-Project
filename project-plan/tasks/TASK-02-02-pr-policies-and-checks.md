# TASK-02-02 PR Policies and Checks for .NET Changes

- Story: STORY-02 Code Build Pipeline
- Estimate: 2 weeks
- Dependencies: TASK-02-01 Review Code Build Process
- Steps:
  1. Configure PR branch policies requiring successful unit tests and security scans for .NET changes.
  2. Wire automated unit tests into the PR pipeline with path filters for .NET components.
  3. Add security scan step (e.g., SAST or dependency scan) that blocks merges on findings.
  4. Pilot the policy on IAM repo; iterate based on developer feedback.
- Deliverables:
  - Enforced PR policy with automated unit tests and security scans.
  - Documented instructions for adding the policy to other services.
