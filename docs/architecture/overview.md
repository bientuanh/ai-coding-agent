# Architecture Overview

## 1. Purpose

This project is a TypeScript Express API for Fin Mate. It exposes authenticated endpoints for user management, JWT-based session handling, notifications, and market/stock data integration.

The service follows a layered backend pattern:

- HTTP routes handle incoming requests
- middleware validates auth and request payloads
- controllers orchestrate request flow
- DAO modules encapsulate database access
- models define MongoDB schemas and persistence rules
- shared utilities keep response shape and cross-cutting behavior consistent

## 2. Runtime and technology stack

- Node.js runtime
- Express 5 web framework
- TypeScript
- MongoDB with Mongoose
- JWT for access and refresh tokens
- Zod for validation
- Firebase Admin for push notification topic subscriptions
- Axios for external SSI market API integrations
- Jest for tests

## 3. Application bootstrap

The entry point is [src/index.ts](../../src/index.ts).

At startup, the app:

- creates the Express instance
- enables cookie parsing and CORS
- reads JSON request bodies
- connects to MongoDB
- registers route groups for users, auth, and notifications
- attaches a global error handler
- starts listening on the configured host and port

Key behavior:

- in production, it listens on the configured port only
- in development, it listens on a configured host such as localhost or 0.0.0.0

## 4. Configuration layer

Environment configuration is centralized in [src/config/appConfig.ts](../../src/config/appConfig.ts).

This file loads process environment values for:

- runtime mode and server address
- MongoDB connection string
- JWT secrets and token lifetimes
- SSI API credentials
- Firebase service account configuration

This pattern keeps external configuration explicit and prevents secrets from being hardcoded in application code.

## 5. Route structure

Routes are defined by feature under [src/routes](../../src/routes):

- [src/routes/auth.routes.ts](../../src/routes/auth.routes.ts): signup, signin, signout, refresh-token
- [src/routes/user.routes.ts](../../src/routes/user.routes.ts): user listing, current-user lookup, user detail
- [src/routes/notification.routes.ts](../../src/routes/notification.routes.ts): list, detail, unread, latest, read, subscribe operations

The route layer is intentionally thin. It delegates to validation middleware and controllers rather than embedding business logic.

## 6. Request validation

Validation is handled with Zod schemas in [src/validators](../../src/validators):

- [src/validators/authSchema.ts](../../src/validators/authSchema.ts)
- [src/validators/userSchema.ts](../../src/validators/userSchema.ts)

The validation middleware, defined in [src/middlewares/validate.ts](../../src/middlewares/validate.ts), checks payloads before the controller runs. This creates a consistent boundary between raw HTTP input and application logic.

## 7. Controllers and business flow

Controllers live in [src/controllers](../../src/controllers) and contain request/response orchestration.

Examples:

- [src/controllers/auth.controller.ts](../../src/controllers/auth.controller.ts): signup, signin, token refresh, logout
- [src/controllers/user.controller.ts](../../src/controllers/user.controller.ts): list users, fetch current user, fetch user by ID
- [src/controllers/notification.controller.ts](../../src/controllers/notification.controller.ts): notification listing, read state, subscription, pagination, latest/unread views

Controller responsibilities include:

- extracting request data
- validating required conditions
- calling DAO/service methods
- returning standardized success or error responses
- delegating unexpected exceptions to the central error handler

## 8. Data access layer

The project separates database logic into DAO modules under [src/models/daos](../../src/models/daos):

- [src/models/daos/user.dao.ts](../../src/models/daos/user.dao.ts)
- [src/models/daos/notification.dao.ts](../../src/models/daos/notification.dao.ts)

This pattern keeps Mongoose query logic out of controllers and makes persistence logic easier to test and review.

## 9. MongoDB models

The domain models live in [src/models](../../src/models):

- [src/models/user.model.ts](../../src/models/user.model.ts)
- [src/models/notifiation.model.ts](../../src/models/notifiation.model.ts)

The user schema defines fields such as firstName, lastName, email, password, role, and image, and uses MongoDB timestamps. The schema also applies a toJSON transform to expose id while removing the Mongo \_id field from serialized responses.

## 10. Authentication and authorization

Authentication is implemented through JWT helpers in [src/libs/jwt.ts](../../src/libs/jwt.ts).

The flow is:

- signin creates an access token and refresh token
- web clients store tokens in secure HTTP-only cookies
- mobile clients may receive tokens directly in the response body
- protected routes use [src/middlewares/verifyAuth.ts](../../src/middlewares/verifyAuth.ts) to read tokens from cookies or Authorization headers
- the request object is enriched with user information before continuing

This design supports both browser and mobile clients while allowing a single auth middleware to enforce authorization.

## 11. Response conventions

All API responses follow a shared helper pattern in [src/utils/response.ts](../../src/utils/response.ts).

There are two standard helpers:

- successResponse(): returns success: true with message and optional data
- errorResponse(): returns success: false with code, message, and optional validation errors

This keeps the API consistent and predictable across controllers.

## 12. Error handling

The central error handler is [src/middlewares/errorHandler.ts](../../src/middlewares/errorHandler.ts).

It logs the error and returns a structured HTTP error response. In controllers, async functions typically call next(err) to delegate errors to the global handler instead of manually writing response logic.

## 13. External service integrations

The project integrates with external systems in [src/libs](../../src/libs):

- [src/libs/axios/ssiClient.ts](../../src/libs/axios/ssiClient.ts): configured Axios client for the SSI stock/market API
- [src/libs/firebase](../../src/libs/firebase): Firebase Admin client and FCM topic subscription logic

The stock service in [src/services/stock.service.ts](../../src/services/stock.service.ts) wraps external market API requests such as:

- daily index data
- daily OHLC stock data
- daily stock price data

This keeps external API call details out of route and controller logic.

## 14. Notification domain

The notification subsystem uses a dedicated DAO and controller pattern.

Key behaviors:

- subscribe a device token to an FCM topic
- fetch paginated notifications
- read notification state
- retrieve latest notifications by frequency
- fetch unread notifications
- create and send notification records

This is an example of the project’s feature-based layering: route -> controller -> DAO -> model -> response.

## 15. Architectural principles used in this repo

The codebase consistently follows a few core principles:

1. Thin routes
2. Validated requests at the boundary
3. Controller-level orchestration only
4. DAO-based database access
5. Shared response helpers for all API payloads
6. Secure auth using JWT and cookies where needed
7. Feature-based organization instead of broad cross-cutting abstractions

## 16. Typical request lifecycle

A typical request follows this flow:

1. Express receives the HTTP request
2. middleware runs (auth, validation, CORS, parsing)
3. route matches and dispatches to a controller
4. controller calls DAO or service logic
5. database or external service is accessed if needed
6. controller returns a success/error response via shared utilities
7. global error handler handles any exceptions

## 17. Summary

The application architecture is a classic layered Express backend:

- HTTP layer for routes

- validation and auth middleware
- controller orchestration
- DAO/model persistence logic
- utility wrappers for consistency
- external service integrations isolated behind shared client/service modules

This keeps the project modular, readable, and compatible with the conventions already in use across the codebase.
