# Tulka documentation

Requirements, decisions, architecture and verification records for **Tulka**, a members webapp for family and friends around a farmhouse whose barn is a guest apartment and pottery studio. Sections include a shared calendar, booking, events, recipes, chat and a garden diary; a robot workshop and a music studio are planned.

The app is an Angular frontend ([AngularAWTemplate](https://github.com/asawegelius/AngularAWTemplate)) and a Spring Boot backend, structured so that each section is one frontend feature plus one backend module. Start with the [platform overview](architecture/platform-overview.md) and the [decisions](decisions/README.md).

This repository follows [documents-template](https://github.com/asawegelius/documents-template).

## Domains

Documentation domains match the section keys: `BOOKING`, `CALENDAR`, `EVENTS`, `RECIPES`, `CHAT`, `GARDEN`, platform domains `IDENTITY`, `GROUPS`, `VISIBILITY`, `MEDIA`, and `SYSTEM` for system-wide requirements. Planned: `ROBOT`, `MUSIC`.

## Start here

1. Read the [documentation conventions](CONVENTIONS.md).
2. Follow the [registration example](examples/README.md) from requirement to test design and evidence.
3. Copy the necessary [templates](templates/README.md) into the destination folders below.
4. Review documentation and implementation together in a pull request.

Agents also read [AGENTS.md](AGENTS.md). People and agents use the same artifact definitions and acceptance criteria.

## Structure

| Location | Responsibility |
|:---------|:---------------|
| [requirements/](requirements/README.md) | Required outcomes, constraints and measurable acceptance criteria |
| [use-cases/](use-cases/README.md) | Actor goals, flows and exceptions |
| [test-cases/](test-cases/README.md) | Repeatable verification scenarios and evidence references |
| [decisions/](decisions/README.md) | Significant design decisions and their consequences |
| [traceability/](traceability/README.md) | Requirement coverage and visible gaps |
| [glossary/](glossary/README.md) | Shared meaning of domain terms |
| [architecture/](architecture/README.md) | Platform and per-module service overviews |
| [api/](api/README.md) | API conventions and endpoint contracts |
| [git-strategy/](git-strategy/README.md) | Branching, review and release guidance |
| [templates/](templates/README.md) | Reusable starting documents |
| [examples/](examples/README.md) | Illustrative records, excluded from live project coverage |

## Smallest useful workflow

1. Write a testable requirement and identify its owner and source.
2. Add a use case when an actor interaction needs explanation. Quality constraints can be tested directly.
3. Design tests with explicit inputs, expected results and requirement IDs.
4. Implement within the agreed scope; link the work item and pull request to the affected IDs.
5. Run relevant checks. Record the tested commit, environment, result and evidence location.
6. Refresh coverage and review the changes. An approved document or an existing test case does not prove working software.

Use additional architecture, API, state, security and operational templates when needed. Choose and document their destination folders before using them; not every template is needed for every change.

## Adopting this template

- Start with the named adopter archive from a published release or open a scoped upgrade PR for an existing project. [Adoption and verification guide](UPGRADING.md)
- Replace this introduction with the project's purpose, scope and entry points.
- Choose domain prefixes, document owners, reviewers and the project's Definition of Done.
- Keep illustrative records in `examples/` or remove that folder; never count them as project evidence.
- Choose a branching and release model. The supplied Git strategy supports `develop` and `main` with optional shared staging.
- Configure actual validation commands and required checks. No CI or branch protections are installed by this template.
- Choose a license before distributing the template for reuse; no license is currently supplied.

## Current maturity

This is a documentation starter. [Conventions](CONVENTIONS.md) govern project records and traceability. The [Git strategy](git-strategy/README.md) covers verification and template releases. When adapting the starter, choose and run the checks that apply to your project.
