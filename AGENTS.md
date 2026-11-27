# Agent working instructions

Read [README.md](README.md) and [CONVENTIONS.md](CONVENTIONS.md) before changing project documentation. These conventions apply equally to people and agents.

## Before editing

1. Identify the requested outcome, authorized scope and affected artifact IDs.
2. Read relevant requirements, use cases, decisions, tests and local instructions. Follow references needed for the task.
3. Identify unknown facts and conflicting requirements. Make reversible progress where possible; surface decisions that change acceptance criteria or exceed authorized scope.
4. Inspect existing IDs and open work before allocating an ID. Never renumber records for cosmetic consistency.

## While working

- Copy canonical templates into project folders; keep examples separate from project records.
- Preserve stable IDs and canonical relationship fields. Use explicit acceptance criteria and expected results.
- Keep related documentation and implementation changes reviewable together. Avoid unrelated restructuring.
- Do not invent requirements, approvals, command output or test results. Label proposals and assumptions.
- Follow the user's authorization and project review roles. Document status does not itself grant permission to merge, release or deploy.
- Use the adopting project's actual validation commands. This starter supplies no automated validator, application tests or CI workflow.

## Before handing off

- Review changed links, IDs and relationship targets against the conventions.
- Check requirement coverage, including requirements without use cases.
- Run relevant available checks; report commands and observed results. If a check was not run, state why.
- Reset affected execution summaries to `not_run` when earlier evidence no longer applies.
- Summarize changes, affected IDs, evidence, unresolved questions and any next action.

## Git guidance

Use a task branch and pull request when working through GitHub. Follow the [Git strategy](git-strategy/README.md) and the project's authorization for integration, promotion and release.

Examples, templates and documented procedures do not themselves authorize agents to create environment branches, merge, tag, publish releases or deploy. Such actions require authorization under the applicable project governance.

For an approved template adoption, follow the [upgrade procedure](UPGRADING.md). Pin the revisions, obtain scope approval and preserve project-owned records. Record the result and effective source anchors in the adoption PR.
