---
name: typescript
description: "TypeScript conventions for the Fin Mate API, including typing, async code, imports, errors, formatting, and quality checks."
applyTo: "**/*.ts"
---

# TypeScript Instructions

## Types

- Prefer explicit types at public boundaries such as controller inputs, API responses, service contracts, DAO methods, and Mongoose model interfaces.
- Reuse existing types from `src/types/` instead of duplicating domain shapes.
- Prefer `type` or `interface` based on the surrounding code and keep names descriptive in PascalCase.
- Avoid `any`; use a narrower type, `unknown` with type narrowing, or a reusable shared type instead.
- Keep nullable and optional values explicit and handle them before use.
- Use type guards or schema validation when narrowing data from external sources.

## Functions and Async Code

- Use camelCase for functions and variables, and keep functions small and focused.
- Use async/await for database, network, and other asynchronous operations.
- Preserve the existing error propagation pattern; in Express handlers, pass unexpected errors to `next(err)`.
- Avoid unhandled promises and floating asynchronous calls unless they are deliberately managed.
- Return consistent, declared value types from reusable functions.

## Imports and Modules

- Keep imports grouped by external packages followed by internal modules.
- Prefer existing project utilities and constants before adding new helpers.
- Use the project’s existing module style and avoid introducing a second import or export convention.
- Remove unused imports, variables, parameters, and unreachable branches.

## Error Handling and Safety

- Validate external input at the application boundary before using it as a trusted domain value.
- Do not expose or log passwords, tokens, secrets, or sensitive request data.
- Use typed errors or the project’s shared error handling utilities when available.
- Avoid non-null assertions unless the invariant is guaranteed by a nearby check or framework contract.
- Handle errors from external services and persistence layers without hiding the original failure context.

## Code Style

- Follow the repository’s ESLint and Prettier configuration; do not reformat unrelated code.
- Use descriptive names rather than one-letter variables.
- Prefer straightforward control flow over clever type-level abstractions.
- Keep comments minimal and add them only for non-obvious behavior or important invariants.
- Preserve existing public APIs unless a requested change requires an intentional contract update.

## Verification

- Run the narrowest relevant test after changing behavior.
- Run `npm run typecheck` for type-related changes.
- Run `npm run lint` for style, imports, and static-analysis changes.
- Run `npm run build` when changes affect compilation, module boundaries, or production behavior.
