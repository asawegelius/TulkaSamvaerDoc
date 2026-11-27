# Bugfix handling

For a bug in a candidate, branch `fix/<work-item>-<description>` from the affected candidate. If the project uses a permanent `test` branch, start from its tested revision. The [bugfix diagram](diagrams/bugfix-handling.png) shows the staging variant.

1. Identify the failing revision and reproduce or document the observed failure.
2. Fix and verify it on the task branch. Bring in related changes from the affected line and resolve conflicts there.
3. Open reviewed PRs to the affected candidate or staging line and `develop` so the fix reaches subsequent releases.
4. For an urgent stable defect, branch `hotfix/<work-item>-<description>` from the affected stable line. Apply the same verification and human release approval, then propagate the fix to `develop`.
5. Record the revision and actual verification evidence for each affected line.
