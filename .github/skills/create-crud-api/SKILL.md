---
name: create-crud-api
description: "Use when creating or extending a CRUD API feature in this TypeScript Express repository, including routes, Zod validation, controllers, Mongoose models, DAO methods, authentication, standardized responses, and Jest tests."
---

# Create CRUD API

Use this skill for a new resource or for extending an existing resource. Follow the repository's layered flow and keep the implementation focused on one domain entity.

## Repository Flow

The normal request path is:

```text
route -> middleware -> controller -> DAO/service -> Mongoose model -> MongoDB
																			|
																			-> successResponse/errorResponse
```

Implement the layers in this order:

1. Define or update the domain type in `src/types/`.
2. Define or update the Mongoose schema and model in `src/models/`.
3. Add data-access functions in `src/models/daos/`.
4. Add Zod request schemas in `src/validators/` for body validation.
5. Add controller handlers in `src/controllers/`.
6. Register routes in `src/routes/`.
7. Register the route group in `src/index.ts` when it is a new resource.
8. Add focused Jest tests under `src/test/`.
9. Run type checking, linting, formatting, and the relevant tests.

Do not put Mongoose queries directly in controllers. Controllers coordinate request input, DAO calls, status decisions, and response formatting.

## Naming and File Conventions

- Use domain-specific names such as `notification.model.ts`, `notification.dao.ts`, and `notification.controller.ts`.
- Use camelCase for functions and variables, PascalCase for models and types.
- Use handler suffixes for controller entry points, for example `createBookHandler` and `getBookByIdHandler`.
- Use named exports for handlers and DAO functions where the existing feature does so.
- Keep imports sorted using the configured `simple-import-sort` ESLint rule.

## Model and Type Layer

Create an explicit TypeScript interface in `src/types/<resource>.ts`, then use it in the Mongoose document type and DAO return types. Prefer required fields that reflect the business rule and use Mongoose timestamps when the resource needs `createdAt` and `updatedAt`.

Follow existing model conventions:

- Use `mongoose.models.<Name> || mongoose.model(...)` to avoid model re-registration.
- Define unique constraints in the schema when appropriate, such as the unique user email.
- Mark secrets such as passwords with `select: false` and explicitly select them only where authentication needs them.
- Configure `toJSON` transformations to expose `id` and remove `_id` and `__v` from API responses.
- Use schema enums for finite values, as the notification model does for market status.

## DAO Layer

DAOs own MongoDB queries and return typed promises. Use async functions and `.exec()` for Mongoose queries.

Typical methods are:

```ts
getAllResources(): Promise<Resource[] | null>
getResourceById(id: string): Promise<Resource | null>
createResource(data: Partial<Resource>): Promise<Resource>
updateResourceById(id: string, data: Partial<Resource>): Promise<Resource | null>
deleteResourceById(id: string): Promise<boolean>
```

Only add operations required by the feature. The existing code is not a universal CRUD template: users currently expose read operations, while notifications expose create, read, pagination, unread filtering, and a domain-specific mark-as-read update.

For pagination, follow the notification pattern:

- Convert page and limit inputs at the controller boundary.
- Calculate `skip = (page - 1) * limit` in the DAO.
- Fetch the page and total count with `Promise.all`.
- Return `{ data, total }` from the DAO.
- Compute `hasMore` in the controller and include `page`, `limit`, `total`, and `hasMore` in the response.

## Validation

Define Zod schemas in `src/validators/` and attach them at the route boundary with `validate(schema)`:

```ts
router.post("/", validate(createResourceSchema), createResourceHandler);
```

Use clear user-facing messages, including email and minimum-length validation where relevant. The middleware calls `schema.parse(req.body)` and returns the repository's standard validation error response with HTTP 400.

Do not rely only on controller checks. Controller checks may still be used for domain-specific conditions or values not covered by the route schema.

## Controller Layer

Controllers should:

- Use `async (req, res, next)` and wrap asynchronous work in `try/catch`.
- Call `next(err)` for unexpected errors so `src/middlewares/errorHandler.ts` handles them centrally.
- Read route parameters from `req.params`, query values from `req.query`, and body values from `req.body`.
- Return an explicit 404 response when a requested resource does not exist.
- Use `successResponse(res, message, data?, statusCode?, code?)` for success.
- Use `errorResponse(res, code, message, statusCode?, errors?)` for failures.
- Use existing error codes from `src/constants/errorCodes.ts`; add a domain-specific code only when no existing code fits.

Example shape:

```ts
export const getResourceByIdHandler = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const resource = await getResourceById(req.params.id);
    if (!resource) {
      errorResponse(res, ERROR_CODES.RESOURCE_NOT_FOUND, "Resource not found", 404);
      return;
    }

    successResponse(res, "Fetched resource successfully", resource);
  } catch (err) {
    next(err);
  }
};
```

Avoid returning raw `res.json(...)` objects and avoid leaking passwords, tokens, or other secrets in response data or logs.

## Routes and Authentication

Create a router in `src/routes/<resource>.routes.ts` and mount it under `/api/<resource>` in `src/index.ts`.

Protect resource routes with `verifyAuth` unless the operation is intentionally public:

```ts
router.get("/", verifyAuth, getAllResourcesHandler);
router.get("/:id", verifyAuth, getResourceByIdHandler);
router.post("/", verifyAuth, validate(createResourceSchema), createResourceHandler);
```

Route ordering matters. Register fixed paths such as `/latest`, `/unread`, or `/me` before `/:id` so they are not interpreted as identifiers.

The existing authentication middleware accepts access tokens from:

- The `accessToken` HTTP-only cookie for web clients.
- The `Authorization: Bearer <token>` header for mobile clients.

Do not add a second authentication mechanism for a new CRUD feature.

## Response Contract

Successful responses use this shape:

```json
{
  "success": true,
  "message": "Fetched resource successfully",
  "data": {}
}
```

Error responses use this shape:

```json
{
  "success": false,
  "code": "RESOURCE_NOT_FOUND",
  "message": "Resource not found",
  "errors": []
}
```

Keep messages consistent with nearby handlers and use appropriate HTTP status codes: 200 for reads/updates, 201 only when the existing feature explicitly requires it, 400 for invalid input, 401 for missing or invalid authentication, 404 for missing resources, and 409 for conflicts such as duplicate emails.

## Test and Verification Checklist

Add tests for the resource's meaningful behavior, including:

- Validation accepts valid input and rejects malformed or missing required fields.
- DAO/controller behavior for a found resource and a missing resource.
- Authentication behavior for protected routes when practical.
- Create, update, or delete behavior only when those operations are part of the feature.
- Pagination metadata and boundary cases when pagination is added.
- Domain-specific filtering or status transitions.

Run the narrowest useful checks first, then the repository checks:

```bash
npm test -- --runInBand
npm run typecheck
npm run lint
npx prettier --check .
```

Before finishing, confirm the route is mounted, validation is attached at the boundary, all database access goes through a DAO or service, responses use the shared helpers, and no secrets appear in source or logs.
