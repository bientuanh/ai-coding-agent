---
name: implement-plan
description: "Use when implementing an approved plan from docs/plans/. Read the plan and repository context, execute exactly one approved step at a time, validate each step, update plan status, and stop for review before continuing."
---

# Implement Plan

Use this skill when the user asks to implement an existing approved plan stored under
`docs/plans/`.

This skill is the execution counterpart of `create-plan`. It does not create a new
plan and does not silently redesign the approved implementation.

## Preconditions

Before starting implementation, read all documentation files under:

- `docs/architecture/**`
- `docs/business/**`

These documents provide the system architecture, coding conventions, domain
context, business rules, user stories, and existing implementation plans. Use
them to understand the intended behavior and constraints before modifying source
code. Do not rely only on the selected plan or nearby source files.

Before changing source code:

1. Locate the relevant plan under `docs/plans/`.
2. Read the complete plan, including:
   - Status
   - Goal
   - Scope
   - Assumptions and decisions
   - Implementation steps
   - Risks and open questions
   - Acceptance criteria
   - Change history
3. Confirm that the plan status is `Approved` or that the user explicitly asked to
   implement a specific approved step.
4. Confirm which step is next and verify that all prerequisite steps are completed.
5. Read the repository instructions that apply to the files in the selected step.
6. Inspect the existing implementation, tests, and documentation referenced by the step.

Do not implement a plan with unresolved architectural, security, or behavioral
decisions. Mark the plan as `Blocked` or `Changes requested` and ask the user for
clarification instead.

## Execution Rules

- Implement only one pending plan step per turn.
- Do not combine multiple pending steps without explicit user approval.
- Keep changes limited to the files and behavior described by the selected step.
- Follow existing architecture, helpers, naming, formatting, and response contracts.
- Add or update directly related tests and documentation when required by the step.
- Do not perform unrelated refactoring or cleanup.
- Do not change dependencies, environment variables, API contracts, or database
  behavior unless the plan explicitly includes those changes.
- Never commit or push changes unless the user explicitly requests it.
- Never claim a step is complete before its validation passes.

## Step Workflow

For each approved step:

### 1. Prepare the step

1. Re-read the current plan.
2. Confirm the selected step has status `Pending`.
3. Check that its dependencies and prerequisite steps are complete.
4. Change the plan status to `In progress`.
5. Add a short change-history entry describing the step being started.

### 2. Implement the step

1. Read the relevant source files and nearby tests.
2. Make the smallest complete change that satisfies the step.
3. Preserve existing behavior outside the plan scope.
4. Use the repository's standard patterns for:
   - Routes and middleware
   - Zod validation
   - Controllers and error handling
   - Services and DAOs
   - Mongoose models
   - Standardized responses
   - Jest tests
5. Surface errors through the existing error-handling flow.
6. Avoid silent fallbacks, broad catches, unsafe casts, and raw response objects.

### 3. Validate immediately

Run the narrowest checks specified by the step. Depending on the change, this may
include:

```bash
npx jest src/test/path/to/file.test.ts --runInBand
npm run typecheck
npm run lint
npx prettier --check .
npm run build