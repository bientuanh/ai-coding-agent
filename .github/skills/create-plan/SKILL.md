---
name: create-plan
description: "Use when planning a task before implementation. Read the requirements and repository context, create or update a plan in docs/plans/, stop for user review, and execute only one approved plan step at a time."
---

# Create Plan

Use this skill when the user wants to understand and plan a task before code changes begin. The plan is a reviewable artifact stored under `docs/plans/`.

The default workflow has two phases:

1. **Planning:** understand the requirement, inspect the relevant code and documentation, and create or update a plan file. Stop and wait for the user's review.
2. **Execution:** after approval, implement exactly one approved step at a time, validate it, update the plan status, and wait for approval before starting the next step.

Do not implement production code during the planning phase.

## Planning Phase

When the user asks for a plan:

1. Read the complete user requirement and identify the desired outcome.
2. Identify ambiguity, constraints, affected modules, risks, and acceptance criteria.
3. Read the relevant project instructions before inspecting or changing matching files.
4. Inspect the closest existing implementation, tests, architecture documents, and configuration.
5. Identify the smallest set of files likely to change.
6. Create or update one plan file under `docs/plans/`.
7. Validate the plan document with Prettier when available.
8. Report the plan path and stop for user review.

If the requirement is ambiguous in a way that changes the architecture, behavior, security, or scope, ask a focused question before writing the plan. Otherwise, document the assumption in the plan and continue.

## Plan File Naming

Use a descriptive kebab-case filename:

```text
docs/plans/<short-task-name>.md
```

Examples:

```text
docs/plans/add-password-reset.md
docs/plans/improve-notification-pagination.md
docs/plans/add-stock-alert-endpoint.md
```

When a matching plan already exists, read it first and update it instead of creating a duplicate. Preserve completed steps and record meaningful revisions in the change history.

## Required Plan Structure

Every plan should contain these sections:

```markdown
# <Task Name>

## Status

`Draft - awaiting review`

## Goal

<What the task should achieve and why it is needed.>

## Scope

### In Scope

- <item>

### Out Of Scope

- <item>

## Current Context

- <Relevant existing behavior, files, and constraints.>

## Assumptions And Decisions

- <Assumption or decision.>

## Implementation Steps

### Step 1: <Step name>

- Status: `Pending`
- Files: `<file paths>`
- Changes: <specific changes to make>
- Validation: `<focused test, typecheck, lint, or other check>`
- Done when: <observable completion criteria>

### Step 2: <Step name>

- Status: `Pending`
- Files: `<file paths>`
- Changes: <changes>
- Validation: `<focused check>`
- Done when: <completion criteria>

## Risks And Open Questions

- <Risk, tradeoff, or unresolved question.>

## Acceptance Criteria

- [ ] <criterion>

## Change History

- <date>: Created plan.
```

Each step must be independently understandable and testable. Keep steps small enough that one step can be implemented and validated in one agent turn. Include concrete file paths, symbols, dependencies, and a falsifiable validation command whenever possible.

## Plan Quality Rules

- Base the plan on repository evidence, not generic assumptions.
- Prefer existing architecture, helpers, naming, and test patterns.
- Separate behavior changes from refactoring and unrelated cleanup.
- Include validation for success, failure, boundary, and security-sensitive behavior when relevant.
- Call out database migrations, environment variables, API contract changes, and backward compatibility.
- Do not hide unresolved decisions inside implementation steps.
- Do not claim a step is complete before its validation passes.
- Keep the plan concise enough to review, but detailed enough to implement without rediscovering the architecture.

## Review Gate

After creating or updating the plan:

1. Summarize what was inspected.
2. Link the plan file.
3. State assumptions and open questions.
4. Ask the user to approve the plan or request changes.
5. Stop. Do not edit production source files until the user approves.

Use these status values:

- `Draft - awaiting review`: plan is ready for user review.
- `Changes requested`: user found issues and the plan needs revision.
- `Approved`: user approved the plan for execution.
- `In progress`: one step is currently being implemented.
- `Blocked`: implementation cannot continue without a decision or external prerequisite.
- `Completed`: all approved steps and validation are complete.

## Execution Phase

Enter execution only when the user clearly approves the plan or asks to proceed with a specific approved step.

For each step:

1. Read the current plan and confirm the step is approved and `Pending`.
2. Update the plan status to `In progress` before editing source files.
3. Implement only that step and its directly required tests or documentation.
4. Run the step's focused validation immediately after the edit.
5. Repair local failures and rerun the same validation before widening scope.
6. Update the step to `Completed` only after validation passes.
7. Add a short entry to `Change History`.
8. Report what changed, what was validated, and what step is next.
9. Stop and wait for the user's approval to continue.

Do not silently combine multiple pending steps. If implementation reveals that the plan is incomplete, mark the plan `Changes requested` or `Blocked`, explain why, and return to the review gate.

## Validation Expectations

Choose the narrowest executable check for the step:

- Focused Jest test for a behavior change.
- Typecheck for a type or contract change.
- ESLint for TypeScript quality.
- Prettier check for documentation or formatting.
- Build when emitted output or module integration is affected.

For API work, also verify route registration, validation middleware, standardized responses, authentication behavior, and DAO/service boundaries as applicable. Do not report success based only on a diff review when an executable check is available.

## Final Completion

When every approved step is complete:

1. Run the relevant final checks from the plan.
2. Mark the plan `Completed`.
3. Record the completed validation in `Change History`.
4. Summarize changed files, behavior, tests, and any residual risks.

Never commit or push changes as part of this skill unless the user explicitly requests it.
