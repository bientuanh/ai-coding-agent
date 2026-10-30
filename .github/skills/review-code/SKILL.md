---
name: review-code
description: "Use when reviewing Fin Mate Express and Node.js code for bugs, security issues, behavioral regressions, architecture violations, missing validation, response contract problems, and test gaps."
---

# Review Code

Review code as a senior Express and Node.js developer. Prioritize correctness, security, data integrity, maintainability, and regression risk over style preferences.

## Review Workflow

1. Identify the changed files and understand the surrounding request flow before judging the implementation.
2. Read the relevant route, middleware, controller, service, DAO, model, validator, type, and test code as one behavior path.
3. Compare the implementation with `docs/architecture/coding-conventions.md` and the applicable project instructions.
4. Trace success, validation failure, authentication failure, missing-resource, conflict, and unexpected-error paths.
5. Check whether the existing tests exercise the changed behavior and important branches.
6. Report findings first, ordered by severity. Keep the summary secondary.

## Findings Format

Only report actionable findings supported by the code. For every finding, include:

- Severity: `Blocker`, `High`, `Medium`, or `Low`.
- A clickable file reference with the relevant line or narrow line range.
- The concrete problem and why it matters.
- A concise remediation direction when the fix is not obvious.

Use this structure:

```text
### [High] Short issue title
[src/path/file.ts](src/path/file.ts#L42)

Explain the observable failure, affected behavior, and why the issue should be fixed.
```

Do not report purely subjective formatting preferences when ESLint or Prettier already enforces the rule. Do not report hypothetical concerns without a plausible execution path.

## Express and Request Flow

- Confirm the flow remains `route -> middleware -> controller -> DAO/service -> model/database -> response helper`.
- Check that body, query, and path parameters are read from the correct request properties.
- Verify body-bearing routes use the existing `validate(schema)` middleware at the route boundary.
- Verify protected resources use `verifyAuth` and do not implement a second authentication mechanism in controllers.
- Check fixed paths such as `/me`, `/latest`, and `/unread` appear before parameterized paths such as `/:id`.
- Confirm async handlers catch unexpected failures and call `next(error)` exactly as the centralized error handler expects.
- Ensure handlers return after sending an error response so execution cannot continue or send a second response.

## Validation and Types

- Check Zod schemas cover required fields, formats, bounds, enums, and unexpected or malformed input.
- Confirm validation messages are clear and user-facing.
- Check request and response boundaries use explicit shared types from `src/types/` where appropriate.
- Flag unsafe `any`, unchecked `unknown`, unjustified non-null assertions, and duplicated domain shapes when they can cause runtime errors.
- Verify optional and nullable fields are handled according to the domain rather than merely silenced by TypeScript.

## Controllers and Responses

- Confirm controllers orchestrate HTTP behavior and do not contain database queries or unrelated business logic.
- Confirm database access stays in `src/models/daos/` and reusable domain or external logic stays in services.
- Verify success paths use `successResponse()` and failures use `errorResponse()` from `src/utils/response.ts`.
- Check status codes: `200` for successful operations, `400` for invalid input, `401` for authentication failures, `404` for missing resources, and `409` for conflicts.
- Verify existing error codes from `src/constants/errorCodes.ts` are reused consistently.
- Check that responses do not expose passwords, tokens, secrets, internal stack traces, or hidden Mongoose fields.

## MongoDB and Mongoose

- Check every query belongs in a DAO, uses the correct model and filter, and returns the expected typed result.
- Verify queries use `.exec()` where the project convention requires an explicit promise.
- Inspect pagination calculations, sorting, limits, skips, and empty-result behavior for off-by-one and unbounded-query bugs.
- Check concurrent independent queries use `Promise.all` when appropriate without masking individual failures.
- Verify models use schema constraints for database invariants such as `required`, `unique`, and `enum`.
- Check secret fields use `select: false` where required and are explicitly selected only for the narrow operation that needs them.
- Verify `toJSON` transformations preserve the API contract by exposing `id` and removing `_id`, version fields, and sensitive data.
- Check model registration does not fail during hot reload or repeated imports.

## Authentication and Security

- Verify password handling uses `bcryptjs` and never stores or compares plaintext values incorrectly.
- Confirm access and refresh tokens use the existing JWT helpers and configured secrets.
- Check both supported auth paths where relevant: secure `accessToken` cookies for web clients and `Authorization: Bearer <token>` for mobile clients.
- Inspect authorization, not only authentication: users must not read or mutate another user’s resources.
- Flag leaked credentials in logs, responses, tests, source files, or configuration.
- Check cookie options, CORS behavior, token refresh, expiration, and invalid-token handling for regressions.
- Review external requests for missing timeout, unsafe user-controlled URLs, unvalidated response data, or accidental credential forwarding.

## Error Handling and Resilience

- Check expected domain failures are translated to the project’s standard response shape.
- Check unexpected failures preserve error context and reach the centralized error handler.
- Look for swallowed promises, missing `await`, unhandled rejections, duplicate retries, and retry loops.
- Inspect external API, Firebase, and database failures for useful fallback behavior without hiding operational failures.
- Verify mutations are not partially applied when a dependent operation fails, or flag the missing transaction/compensation when consistency is required.

## Tests and Regression Risk

- Check new behavior has focused Jest tests under `src/test/` with `*.test.ts` naming.
- Expect success, validation, unauthorized, missing-resource, conflict, boundary, and dependency-failure cases where applicable.
- Verify external services, Mongoose, DAOs, and authentication helpers are mocked in unit tests.
- Check date-dependent tests use controlled time and restore fake timers and mocks.
- Flag tests that assert implementation details while missing the public response, status, or data contract.
- Check the relevant coverage, typecheck, lint, and format verification is possible without a live MongoDB or external service.

## Final Review Response

- List findings first, ordered from most severe to least severe.
- Include open questions or assumptions only after findings.
- Add a brief change summary last.
- If no issues are found, say so clearly and mention remaining test gaps or residual risk.
- Never claim a command passed unless it was actually run; name unavailable checks explicitly.
