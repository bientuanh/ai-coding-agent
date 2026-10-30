---
name: api
description: "API development conventions for the Fin Mate TypeScript Express backend, including routes, validators, controllers, services, DAOs, models, middleware, and tests."
applyTo: "src/routes/**/*.ts,src/validators/**/*.ts,src/controllers/**/*.ts,src/services/**/*.ts,src/models/**/*.ts,src/middlewares/**/*.ts,src/types/**/*.ts,src/test/**/*.ts"
---

# API Development Instructions

## Architecture

- Follow the request flow: route -> validation middleware -> controller -> service or DAO -> standardized response.
- Keep controllers focused on HTTP concerns and move reusable business logic to services.
- Keep database queries in DAO modules under `src/models/daos/`; do not embed Mongoose queries in controllers.
- Reuse existing helpers, constants, types, and response utilities before creating new abstractions.

## Routes and Validation

- Define routes in `src/routes/` and protect authenticated endpoints with the existing auth middleware.
- Validate request bodies, query parameters, and route parameters at the route boundary with Zod schemas in `src/validators/`.
- Use clear, user-facing validation messages and reject unexpected or malformed input.
- Keep route handlers small and delegate application logic to controllers or services.

## Controllers and Responses

- Use async handlers and pass unexpected errors to `next(err)` so the centralized error handler can process them.
- Return responses through `successResponse()` and `errorResponse()` from `src/utils/response.ts`.
- Use explicit HTTP status codes and stable application error codes from `src/constants/errorCodes.ts`.
- Never return raw response objects when a shared response helper applies.
- Do not expose passwords, tokens, secrets, or sensitive internal error details.

## Database and Models

- Define Mongoose schemas and model types under `src/models/`.
- Use typed interfaces from `src/types/` for API payloads and domain data where applicable.
- Use async/await and call `.exec()` on Mongoose queries when returning query promises explicitly.
- Keep transformations and schema-level defaults in the model definition.
- Check existing DAO methods before adding a duplicate query.

## Authentication and Security

- Use the existing JWT helpers in `src/libs/jwt.ts` and authentication middleware.
- Read secrets and configuration from `src/config/appConfig.ts`; never hard-code credentials.
- Never log raw passwords, access tokens, refresh tokens, or secret configuration values.
- Use secure, `httpOnly` cookies with the project’s existing `sameSite` conventions when setting browser tokens.
- Enforce authorization checks before reading or mutating user-owned resources.

## TypeScript and Style

- Use camelCase for variables and functions, PascalCase for types and models, and descriptive feature-based filenames.
- Prefer explicit types at API and database boundaries; avoid `any` unless unavoidable and documented by the surrounding API.
- Keep imports grouped and follow the existing ESLint and Prettier configuration.
- Keep comments limited to non-obvious behavior.

## Testing and Verification

- Add focused Jest tests for new validators, controllers, services, DAOs, and endpoint behavior.
- Cover successful requests, validation failures, authentication or authorization failures, missing resources, and persistence errors where relevant.
- Before completing API changes, run the narrowest relevant Jest test, then `npm run typecheck`, `npm run lint`, and `npm run build` when the change affects those checks.
