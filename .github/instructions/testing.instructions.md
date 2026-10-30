---
name: testing
description: "Testing conventions for the Fin Mate TypeScript Express API, including Jest unit tests, mocks, coverage, validation, controllers, DAOs, services, and authentication."
applyTo: "src/test/**/*.ts"
---

# Testing Instructions

## Test Organization

- Use Jest with `ts-jest` and place tests under `src/test/` with the `*.test.ts` suffix.
- Group tests with `describe()` by function or observable behavior.
- Name test cases after the expected behavior, not private implementation details.
- Keep each test focused and structure it as Arrange, Act, Assert.

## What to Test

- Cover normal success cases, invalid input, boundary values, missing resources, and dependency failures where relevant.
- For utilities and pure functions, test representative values, empty input, boundary dates, invalid input, and both sides of important branches.
- For Zod validators, test valid payloads, missing required fields, invalid formats, boundaries, enums, defaults, and transformed output.
- For controllers, verify status codes, response data, error codes, response helper calls, and `next(error)` for unexpected failures.
- For DAOs, verify query filters, identifiers, sorting, pagination, create/update/delete results, and missing records.
- For services and external clients, verify request paths, parameters, response mapping, empty data, retries, and propagated failures.
- For authentication, cover valid, missing, expired, and invalid tokens, password outcomes, cookie behavior, and token leakage prevention.

## Isolation and Mocks

- Mock DAOs, services, Mongoose models, authentication helpers, Axios clients, Firebase, and other external integrations in unit tests.
- Do not require a live MongoDB database, SSI API, Firebase project, or real credentials.
- Use minimal Express mocks containing only the methods under test, such as `status`, `json`, `cookie`, `clearCookie`, and `next`.
- Reset mocks between tests with `jest.clearAllMocks()` or an appropriate cleanup hook.
- Use fake timers for date-dependent behavior and always restore real timers after each test.
- Never include real secrets, credentials, tokens, or service-account files in tests.

## Assertions

- Prefer assertions against observable return values, responses, status codes, and side effects.
- Use `safeParse()` when testing Zod schemas so both success and error details can be asserted directly.
- Verify the public contract of shared response helpers without over-asserting private implementation details.
- Include assertions for important error paths, not only the happy path.

## Coverage

- Maintain global coverage above 80% for branches, functions, lines, and statements.
- Run the suite with an explicit threshold:

```bash
npx jest --coverage --runInBand --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
```

- Add behavior-focused tests for uncovered branches; do not lower thresholds, exclude source files, or add superficial assertions.
- Do not commit generated `coverage/` output unless the repository explicitly requires it.

## Verification

- Run the narrowest relevant test first:

```bash
npx jest src/test/path/to/file.test.ts --runInBand
```

- Before completing a test change, run `npm run typecheck`, `npm run lint`, and `npm run format` as applicable.
- Keep tests deterministic, restore timers and mocks, and ensure failures provide useful diagnostic context.
