# Agent Instructions

## Project source of truth

The product, architecture, scope, acceptance criteria and implementation phases are defined in `docs/project-plan.md`.

Read the relevant sections of that document before planning or implementing substantial work.

Do not treat the target architecture as a requirement for the current phase. Follow the phased build order and decision gates in the project plan.

## Core working principle

Prove the product before proving the infrastructure.

Prefer the simplest implementation that tests the current hypothesis. Do not introduce infrastructure, abstractions or features merely because they appear in a later phase of the project plan.

A feature or technical investment should either:

1. improve or test user value, or
2. deliberately demonstrate an engineering capability required by the project.

Otherwise, leave it out.

## Before implementing

For substantial changes:

1. Identify the current project phase and relevant acceptance criteria.
2. Inspect the existing implementation before proposing changes.
3. State the intended approach and any important trade-offs.
4. Break large changes into small, independently testable steps.
5. Flag any proposed departure from `docs/project-plan.md` before implementing it.

Do not silently expand the requested scope.

## Implementation

Prefer simple, explicit code over premature abstraction.

Keep clear boundaries between:

- content ingestion and parsing
- canonicalisation and deduplication
- AI classification
- relevance ranking
- persistence
- user-facing presentation

Treat model output and external feed data as untrusted input. Validate them at system boundaries.

Keep external AI providers behind a small interface so model/provider changes do not leak through the application.

Preserve enough metadata to understand and reproduce important AI behaviour.

## Testing

Test meaningful behaviour, not implementation details.

Prioritise tests around:

- URL canonicalisation and deduplication
- structured AI output validation
- relevance/ranking behaviour
- persistence and state transitions
- failure and fallback behaviour

Add integration tests when behaviour crosses real system boundaries.

Do not change or weaken a valid test merely to make an implementation pass.

Run the relevant tests and static checks after making changes.

## AI development

AI behaviour must be evaluated rather than judged from a few convincing examples.

When changing classification, ranking or prompts:

- preserve representative examples and failures
- compare behaviour against the existing approach where practical
- keep summaries grounded in supplied source material
- prefer structured outputs with schema validation
- record model/prompt configuration when the project reaches the phase that requires it

Do not introduce embeddings, vector search or sophisticated ranking until simpler approaches have demonstrated a concrete limitation.

## Dependencies and infrastructure

Before adding a dependency or infrastructure component, explain what current problem it solves.

Do not introduce later-phase components such as Celery, Redis, OpenTelemetry, Terraform or production cloud infrastructure before the project plan calls for them unless there is a demonstrated need.

Prefer standard-library or existing-project solutions when they are adequate.

## Engineering decisions

Significant architectural decisions or intentional departures from the project plan should be documented under `docs/decisions/`.

Decision notes should briefly record:

- context
- decision
- alternatives considered
- reasoning
- consequences

Avoid documenting trivial implementation choices.

## Completion

Before considering a task complete:

1. Confirm the requested behaviour works.
2. Run relevant tests and checks.
3. Check the change against the current phase's acceptance criteria.
4. Report what changed and any unresolved issues.
5. Do not automatically begin the next project phase.