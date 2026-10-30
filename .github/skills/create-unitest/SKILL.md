---
name: create-unitest
description: "Use when creating or extending Jest unit tests in this TypeScript Express repository, especially for utilities, validators, controllers, DAOs, services, authentication, and CRUD API behavior. Cover the main success and failure cases and require global coverage above 80%."
---

# Create Unit Tests

Use this skill when adding or improving unit tests for the Fin Mate API. Tests must verify behavior, cover the main branches, and maintain global coverage above 80% for branches, functions, lines, and statements.

## Test Stack

- Jest 30 with the Node test environment.
- `ts-jest` for TypeScript transformation.
- Tests under `src/test/` with the `*.test.ts` suffix.
- `describe()` blocks grouped by function or behavior.
- `test()` cases with direct Jest assertions.

The repository's Jest configuration is in `jest.config.js`. It currently configures the Node environment and `ts-jest`; coverage thresholds should be enforced when running the coverage command below or added to the Jest configuration when the project adopts permanent enforcement.

## Test Location and Naming

Place tests near the matching domain area under `src/test/`:

```text
src/test/
	utils/dateTime.test.ts
	validators/authSchema.test.ts
	controllers/user.controller.test.ts
	models/daos/notification.dao.test.ts
```

Use names that describe the behavior rather than the implementation detail. For example:

```ts
describe("getWorkingDateRange()", () => {
  test("skips weekends when counting working days", () => {
    // Arrange, act, assert
  });
});
```

## Main Test Cases

For every unit under test, cover the important behavior rather than only the happy path.

### Utilities and Pure Functions

Test:

- Normal input and representative expected output.
- Boundary values, such as zero, empty input, month boundaries, and leap years.
- Invalid input and expected thrown errors.
- Date-dependent behavior using controlled system time.
- Both sides of boolean or status branches.

For date-dependent functions, use Jest fake timers and always restore real timers in cleanup:

```ts
afterEach(() => {
  jest.useRealTimers();
});

test("returns true for Friday", () => {
  jest.useFakeTimers().setSystemTime(new Date("2023-12-22"));

  expect(isTodayFriday()).toBe(true);
});
```

The existing date tests cover weekends, date formatting, working-day ranges, Fridays, month ends, leap years, and invalid day counts. Preserve these cases when extending the date utilities.

### Validators

Test:

- A complete valid payload.
- Missing required fields.
- Invalid formats, such as malformed email addresses.
- Minimum and maximum boundaries, such as password length.
- Enum and default-value behavior.
- The parsed output when a schema transforms or supplies defaults.

Prefer `schema.safeParse()` when testing validation results so tests can assert success or error details without relying on thrown implementation errors.

### Controllers

Mock DAOs, services, authentication helpers, and external integrations. Do not require a live MongoDB, SSI API, or Firebase project for unit tests.

Test:

- Successful response with the expected message, data, and status code.
- Missing required input and the expected error code/status.
- Resource-not-found behavior with HTTP 404.
- Conflict behavior, such as an existing email with HTTP 409.
- Unauthorized or invalid-token behavior for protected flows.
- Unexpected dependency failures call `next(error)`.
- Response helpers are called with the expected contract.

When mocking Express objects, provide only the methods used by the handler, such as `status`, `json`, `cookie`, `clearCookie`, and `next`.

### DAOs and Models

Mock the Mongoose model or isolate DAO tests from the database. Test:

- Query filters and identifiers.
- Sorting, pagination, `skip`, and `limit` behavior.
- `null` results for missing records.
- Create/update/delete success and failure results.
- Typed transformation and serialization rules where they are part of the API contract.

Do not make unit tests depend on an active MongoDB connection. Database integration tests should be explicitly separated and configured with their own environment.

### Services and External Clients

Mock Axios clients and external API responses. Test:

- Correct request path, query parameters, and authorization behavior.
- Successful response mapping.
- Empty or malformed external data.
- Token refresh and one-time retry behavior for SSI requests.
- External failures are propagated or handled according to the service contract.

### Authentication and Security

Test:

- Password hashing and comparison outcomes.
- Valid access and refresh tokens.
- Missing, expired, and invalid tokens.
- Cookie-based web signin behavior.
- Mobile signin behavior returning token values.
- No password, secret, or token leakage in ordinary response data or logs.

Never place real secrets, credentials, or service-account JSON in tests.

## Test Structure

Use Arrange, Act, Assert structure and keep each test focused:

```ts
test("returns a user when the DAO finds the record", async () => {
  // Arrange
  userDao.getUserById = jest.fn().mockResolvedValue(user);

  // Act
  await getUserByIdHandler(request, response, next);

  // Assert
  expect(successResponse).toHaveBeenCalledWith(response, "Fetched user successfully", user);
});
```

Use `jest.mock()` or `jest.spyOn()` for dependencies. Reset mocks between tests with `jest.clearAllMocks()` or an appropriate cleanup hook. Avoid asserting private implementation details when an observable response or returned value is available.

## Coverage Requirement

The minimum global target is greater than 80% for all four Jest metrics:

- Branches.
- Functions.
- Lines.
- Statements.

Run the complete suite with an explicit threshold:

```bash
npx jest --coverage --runInBand --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
```

Coverage must be checked after the main behavior tests are added. Do not lower the threshold, exclude source files, or add superficial assertions just to raise the percentage. If coverage is below 80%, identify uncovered branches and add behavior-focused tests.

Coverage reports are generated under `coverage/`. Do not commit generated coverage output unless the repository explicitly requires it.

## Verification Workflow

Run the narrowest test first, then the full verification sequence:

```bash
npx jest src/test/path/to/file.test.ts --runInBand
npx jest --coverage --runInBand --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
npm run typecheck
npm run lint
npx prettier --check .
```

Before finishing, confirm:

- Main success, failure, boundary, and error-propagation cases are covered.
- External dependencies are mocked in unit tests.
- Tests are deterministic and restore timers/mocks.
- Coverage is above 80% for branches, functions, lines, and statements.
- TypeScript, ESLint, and Prettier checks pass.
