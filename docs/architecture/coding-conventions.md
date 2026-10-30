# Coding Conventions

This document describes the coding patterns currently used by the Fin Mate API. New code should follow these conventions unless a feature has a documented reason to use a different approach.

## Project Structure

The API uses a layered, feature-oriented backend structure:

| Directory          | Responsibility                                                                     |
| ------------------ | ---------------------------------------------------------------------------------- |
| `src/index.ts`     | Application bootstrap, middleware registration, route mounting, and server startup |
| `src/config/`      | Environment-backed application configuration                                       |
| `src/constants/`   | Error codes, system messages, stock definitions, and FCM topics                    |
| `src/routes/`      | Express route definitions and middleware composition                               |
| `src/controllers/` | Request handlers and response orchestration                                        |
| `src/middlewares/` | Validation, authentication, and error handling                                     |
| `src/models/`      | Mongoose schemas/models and DAO modules                                            |
| `src/models/daos/` | Database queries and persistence operations                                        |
| `src/services/`    | Business or external-domain services, such as SSI market data                      |
| `src/libs/`        | Shared integrations and clients, including JWT, MongoDB, Axios, and Firebase       |
| `src/validators/`  | Zod request schemas                                                                |
| `src/types/`       | Shared TypeScript interfaces and union types                                       |
| `src/utils/`       | Reusable response, date, and stock-analysis helpers                                |
| `src/test/`        | Jest tests                                                                         |

Keep business logic in controllers or services and keep database access in DAOs. Do not place Mongoose queries directly in route files or controllers.

## Naming

- Use `camelCase` for variables, functions, and handler names.
- Use `PascalCase` for model names, classes, and TypeScript type names.
- Use descriptive, domain-specific file names such as `auth.controller.ts`, `user.dao.ts`, and `authSchema.ts`.
- Use `*.routes.ts` for route modules, `*.controller.ts` for controllers, `*.dao.ts` for DAOs, and `*.model.ts` for models.
- Use a `Handler` suffix for Express handlers, such as `signupHandler`, `getUserByIdHandler`, and `readNotificationHandler`.
- Use names that describe the domain operation, such as `getNotificationsWithPagination` rather than a generic `queryData`.

## TypeScript

- Use strict TypeScript settings from `tsconfig.json`.
- Prefer explicit interfaces for API payloads and domain models in `src/types/`.
- Reuse shared types instead of duplicating object shapes across controllers, DAOs, and services.
- Give asynchronous DAO and service functions explicit `Promise<...>` return types where practical.
- Avoid `any`; use a specific type or a narrow extension of an Express request when request state is added.
- Use `unknown` in catch paths when the error type is not known.
- Keep functions small and focused on one responsibility.
- Use optional properties only when the domain allows the value to be absent, for example `password?: string` and `readAt?: string | null`.
- Use string unions for finite domain values, such as user roles and notification market statuses.

## Imports and Formatting

Imports are grouped and sorted as enforced by ESLint's `simple-import-sort` rule:

1. External packages.
2. Internal project modules using relative paths.

Use the repository's Prettier configuration:

- Two-space indentation.
- Double quotes.
- Semicolons.
- Trailing commas where supported.
- A 100-character print width.
- No tabs in source files.

Do not reformat unrelated files while making a focused change. Keep comments short and use them only for non-obvious behavior or operational constraints.

## Request Flow

Feature requests follow this path:

```text
route -> middleware -> controller -> DAO/service -> model/database
																			|
																			-> response helper
```

Implement new API features in this order:

1. Define or update types in `src/types/`.
2. Define or update the Mongoose model in `src/models/`.
3. Add or update DAO functions in `src/models/daos/`.
4. Add Zod schemas in `src/validators/`.
5. Implement controller handlers in `src/controllers/`.
6. Register and protect routes in `src/routes/`.
7. Mount a new route group in `src/index.ts`.
8. Add focused tests in `src/test/`.

## Routes and Middleware

Create one router per domain and mount it under an `/api` prefix. Compose middleware at the route boundary:

```ts
router.post("/signup", validate(createUserSchema), signupHandler);
router.get("/", verifyAuth, getAllUsersHandler);
```

Use `validate(schema)` for request-body validation and `verifyAuth` for protected routes. Do not implement a separate authentication mechanism inside a controller.

Register fixed paths before parameterized paths. For example, `/me`, `/latest`, and `/unread` must appear before `/:id` so Express does not treat them as identifiers.

The existing authentication middleware accepts:

- The `accessToken` cookie for web clients.
- An `Authorization: Bearer <token>` header for mobile clients.

## Validation

Define Zod schemas in `src/validators/` with domain-specific names such as `signinSchema` and `createUserSchema`:

```ts
export const signinSchema = z.object({
  email: z.string().email("Email is not valid"),
  password: z.string().min(6, "Password must be at least 6 characters long"),
});
```

Attach schemas at the route boundary with `validate(schema)`. Use clear, user-facing validation messages. Keep controller checks for domain-specific conditions or values not covered by the schema.

The validation middleware parses `req.body` and returns a standard 400 response with the collected Zod messages when parsing fails.

## Controllers

Use the Express handler signature and forward unexpected errors to the centralized handler:

```ts
export const getUserByIdHandler = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const user = await getUserById(req.params.id);
    if (!user) {
      errorResponse(res, ERROR_CODES.USER_NOT_FOUND, "User not found", 404);
      return;
    }

    successResponse(res, "Fetched user successfully", user);
  } catch (err) {
    next(err);
  }
};
```

Controller conventions:

- Use `async` for database and network operations.
- Wrap asynchronous work in `try/catch`.
- Call `next(err)` for unexpected failures.
- Read body, query, and path values from `req.body`, `req.query`, and `req.params` respectively.
- Return after sending an error response to prevent the handler from continuing.
- Return 404 when a requested resource does not exist.
- Use existing error codes from `src/constants/errorCodes.ts`.
- Do not return raw JSON objects when a shared response helper applies.

## Response Format

Use `successResponse()` and `errorResponse()` from `src/utils/response.ts`.

Successful responses have this shape:

```json
{
  "success": true,
  "message": "Fetched user successfully",
  "data": {}
}
```

Error responses have this shape:

```json
{
  "success": false,
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "errors": []
}
```

Use explicit HTTP status codes:

- `200` for successful reads and operations.
- `400` for invalid input.
- `401` for missing or invalid authentication.
- `404` when a resource cannot be found.
- `409` for conflicts such as an existing email.

The route-level validation and controller errors use the shared response shape. The current global `errorHandler` logs the error and returns `{ message }` with the error status or 500; preserve this behavior unless standardization of unexpected errors is part of the feature being changed.

## Models and Mongoose

Models use typed Mongoose schemas and guard against model re-registration:

```ts
const UserModel: Model<IUserDocument> =
  mongoose.models.User || mongoose.model<IUserDocument>("User", UserSchema);
```

Follow these conventions:

- Keep models under `src/models/`.
- Use `timestamps: true` when created and updated timestamps are part of the resource.
- Use schema-level `required`, `unique`, and `enum` constraints for database invariants.
- Use `select: false` for stored secrets such as passwords.
- Explicitly select hidden fields only where required, as `getUserByEmail(email, true)` does for signin.
- Centralize API serialization in the schema's `toJSON` transform.
- Expose `id` and remove MongoDB-specific `_id` and version fields from serialized responses.

## DAOs and Database Queries

DAOs own database operations and use async/await with `.exec()`:

```ts
export const getUserById = async (id: string): Promise<IUser | null> => {
  return UserModel.findById(id).exec();
};
```

DAO conventions:

- Import and query the model from the DAO module.
- Use typed return values such as `Promise<IUser | null>`.
- Keep query composition and persistence details out of controllers.
- Use `Promise.all` when independent queries can be performed together, as in paginated notification queries.
- Keep pagination calculations and sorting rules close to the query that owns them.

## Authentication and Security

- Hash passwords with `bcryptjs`; never store plaintext passwords.
- Use the existing JWT helpers in `src/libs/jwt.ts` for access and refresh tokens.
- Keep JWT secrets and external service credentials in environment variables loaded by `src/config/appConfig.ts`.
- Never log passwords, access tokens, refresh tokens, API credentials, or service-account data.
- Use secure, HTTP-only cookies for web authentication where cookies are used.
- Do not include hidden password fields in ordinary user responses.
- Keep CORS origins and credential behavior configured through `appConfig`.

## Testing

Tests use Jest with `ts-jest` and the Node environment. Place tests under `src/test/` and name them `*.test.ts`.

The existing tests use `describe()` blocks for a function or behavior and `test()` cases with direct assertions:

```ts
describe("isWeekend()", () => {
  test("returns true for Saturday", () => {
    expect(isWeekend(new Date("2023-12-23"))).toBe(true);
  });
});
```

For date-dependent code, use Jest fake timers and restore real timers after each case. Test both normal and boundary behavior, including invalid input, missing records, empty results, and pagination boundaries when relevant.

## Verification Commands

Run the narrowest relevant check first, then the project checks as needed:

```bash
npm test -- --runInBand
npm run typecheck
npm run lint
npm run format
npm run lint:secrets
```

Before finalizing a change, confirm that TypeScript compiles, validation is attached to new body-bearing routes, protected resources use `verifyAuth`, database access remains in DAOs or services, responses use the shared helpers, and no secrets were added.
