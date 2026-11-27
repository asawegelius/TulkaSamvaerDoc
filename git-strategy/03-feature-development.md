# Feature development

Start from the approved `develop` revision and verify it against the upstream branch. Use a task branch named `feature/<work-item>-<description>`. For parallel work, use an independent checkout or worktree. The [feature diagram](diagrams/feature-development-flow.png) shows isolation and integration in the optional staging model.

1. Link the task and applicable artifact IDs. Keep changes focused and commits reviewable.
2. Fetch `develop` and integrate any relevant changes into the task branch. Resolve conflicts there. Broader changes to an approved base require review.
3. Verify the exact result using the project’s checks and record commands, environment and tested commit.
4. Push the task branch and open a PR targeting `develop`. Recheck affected behavior after material updates. Merge through the repository’s authorized review process.
5. Clean up the completed task branch according to the project’s retention policy when authorized.
