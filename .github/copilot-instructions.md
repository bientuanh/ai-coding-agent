# Copilot Instructions for Fin Mate API

## Project overview

This repository is a TypeScript Express API for Fin Mate. The project follows a layered backend structure built around Express, MongoDB/Mongoose, JWT authentication, and Zod validation.

Primary goals:

- Keep the API modular and easy to follow
- Use typed TypeScript interfaces for request/response data
- Keep business logic in controller/service layers and database access in DAO modules
- Standardize API responses and error handling
- Prefer secure auth patterns and explicit validation

## Tech stack

- Runtime: Node.js
- Framework: Express
- Language: TypeScript
- Database: MongoDB with Mongoose
- Auth: JWT access + refresh tokens
- Validation: Zod
- Logging/monitoring: console-based app logging
- Linting/formatting: ESLint + Prettier
- Testing: Jest

## Repository conventions

### Folder layout

- src/index.ts: app bootstrap and route registration
- src/config/: environment configuration
- src/constants/: shared constants and system messages
- src/controllers/: request handlers
- src/libs/: shared helpers, external clients, and DB connection helpers
- src/middlewares/: Express middleware, validation, auth checks, error handler
- src/models/: Mongoose models and DAO files
- src/routes/: route definitions
- src/services/: business/service logic when needed
- src/types/: TypeScript interfaces/types
- src/utils/: reusable helper functions
- src/validators/: Zod schemas
- src/test/: Jest tests

### Naming conventions

- Use camelCase for variables and functions
- Use PascalCase for model and type names
- Keep file names descriptive and domain-specific
- Prefer feature-based naming such as auth.controller.ts, user.dao.ts, authSchema.ts
- Use handler suffixes for controller functions like signupHandler, signinHandler

## API design conventions

### Request flow

Follow the project pattern closely:

1. Route definition in src/routes/
2. Validation via Zod in src/validators/
3. Controller logic in src/controllers/
4. DAO or service access in src/models/daos/ or src/services/
5. Response via successResponse() / errorResponse() helpers

### Response standards

Use the shared response utilities from src/utils/response.ts.

- successResponse(res, message, data?, statusCode?, code?)
- errorResponse(res, code, message, statusCode?, errors?)

Avoid returning raw JSON objects without the standard shape.

### Error handling

- Use next(err) in async controller functions
- Keep a centralized error handler in src/middlewares/errorHandler.ts
- Return structured error payloads with success: false and a code
- Prefer explicit HTTP status codes for validation and auth errors

## Authentication conventions

### JWT behavior

- Access token and refresh token are used for auth
- Web clients use secure cookies
- Mobile clients may receive token values directly in the response
- Use the existing JWT helpers in src/libs/jwt.ts

### Security rules

- Never log raw passwords, tokens, or secrets
- Never store secrets in source files
- Ensure environment variables are read from process.env in src/config/appConfig.ts
- Use secure cookies with httpOnly and sameSite when applicable

## Validation conventions

- Define request validation schemas using Zod
- Validate payloads at the route boundary using validate middleware
- Keep schema names consistent with the endpoint domain, e.g. signinSchema, createUserSchema
- Prefer clear, user-facing validation messages

## Database conventions

### Mongoose usage

- Keep Mongoose models under src/models/
- Keep data access logic in separate DAO modules under src/models/daos/
- Use async/await with Mongoose queries
- Use .exec() when returning a promise explicitly
- Keep model transformations centralized in the schema definition

### Example pattern

- UserModel for the schema
- getUserByEmail() in user.dao.ts for query logic
- controller calls DAO methods instead of embedding queries directly

## TypeScript conventions

- Prefer explicit types for API payloads and model interfaces
- Reuse interfaces from src/types/
- Avoid any unless absolutely necessary
- Keep functions small and readable
- Use async functions for all DB or network operations

## Code style and editing expectations

- Follow the existing project style rather than introducing a different pattern
- Keep imports sorted and grouped by external/internal modules
- Prefer existing helper utilities before creating new ones
- Keep comments minimal and only where they explain non-obvious logic
- Do not add unnecessary abstraction layers for small features

## Commands

Use the project scripts from package.json:

```bash
npm run dev
npm run build
npm run test
npm run lint
npm run lint:fix
npm run typecheck
npm run format
npm run format:fix
```

## Environment variables

Configuration is centralized in src/config/appConfig.ts. When adding new env variables:

- Add them to appConfig
- Use a safe default when appropriate
- Keep names consistent with existing patterns
- Never commit real secrets or local .env values

Example areas already configured:

- NODE_ENV
- PORT
- BASE_URL
- CLIENT_URLS
- MONGO_DB_URI
- JWT_SECRET
- JWT_REFRESH_SECRET
- SSI\_\* values
- FIREBASE_SERVICE_ACCOUNT

## Implementation guidance for future work

When adding new features:

1. Create or update the route in src/routes/
2. Add or update Zod validation in src/validators/
3. Implement controller logic in src/controllers/
4. Add DAO/service logic in the relevant model layer
5. Return standardized success/error responses
6. Keep naming and structure aligned with current codebase conventions

## Quality bar

Before finalizing changes, ensure:

- TypeScript compiles without errors
- Lint passes
- Validation is enforced where required
- Responses are consistent with current API conventions
- No credentials or sensitive data are included in code or config

## Forbidden

- Do not install new dependencies without asking.
- Do not change architecture without asking.
- Do not rewrite working code unnecessarily.
- Do not create duplicate utilities.
