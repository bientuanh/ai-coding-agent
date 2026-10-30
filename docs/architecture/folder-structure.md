# Folder Structure

This project is organized as a modular Express + TypeScript API. The structure is intentionally feature-oriented, with infrastructure and business logic separated into distinct folders.

## Root-level structure

```text
fin-mate-api/
├─ .github/
│  ├─ copilot-instructions.md
│  └─ instructions/
│     ├─ api.instructions.md
│     ├─ react.instructions.md
│     ├─ styling.instructions.md
│     ├─ testing.instructions.md
│     └─ typescript.instructions.md
├─ docs/
│  └─ architecture/
│     ├─ overview.md
│     └─ folder-structure.md
├─ src/
│  ├─ config/
│  ├─ constants/
│  ├─ controllers/
│  ├─ libs/
│  ├─ middlewares/
│  ├─ models/
│  ├─ routes/
│  ├─ scripts/
│  ├─ services/
│  ├─ test/
│  ├─ types/
│  ├─ utils/
│  ├─ validators/
│  ├─ index.ts
│  └─ ...
├─ .gitignore
├─ docker-compose.yml
├─ eslint.config.cjs
├─ jest.config.js
├─ package.json
├─ README.md
├─ tsconfig.json
├─ vercel.json
└─ ...
```

## Purpose of each top-level area

### .github/

Contains repository-level automation and guidance files for Copilot and project instructions. This folder is used to describe coding conventions, project expectations, and agent-specific guidance.

### docs/

Stores project documentation. The architecture section is focused on describing the application structure, behavior, and conventions used across the backend.

### src/

This is the main application codebase. It contains the Express app bootstrap, config, business logic, validation, database models, and route layer.

## src folder breakdown

### src/index.ts

This is the application entry point. It creates the Express app, enables middleware, establishes MongoDB connection, registers route groups, and starts the server.

### src/config/

Contains runtime configuration and environment loading. The main configuration file is:

- src/config/appConfig.ts

This file centralizes values such as:

- NODE_ENV
- PORT
- BASE_URL
- CLIENT_URLS
- MONGO_DB_URI
- JWT secret values
- SSI API credentials
- Firebase service account config

### src/constants/

Stores shared constant values used throughout the app, such as:

- error codes
- FCM topics
- stock constants
- system messages

Examples:

- src/constants/errorCodes.ts
- src/constants/fcmTopics.ts
- src/constants/stock.ts
- src/constants/sysMessages.ts

### src/controllers/

Contains request handlers. Controllers orchestrate logic for each feature and are responsible for calling DAO/service methods and returning standardized responses.

Examples:

- src/controllers/auth.controller.ts
- src/controllers/user.controller.ts
- src/controllers/notification.controller.ts

### src/libs/

Contains shared libraries and integration clients used across the application.

Examples:

- src/libs/jwt.ts: access/refresh token generation and verification
- src/libs/axios/axiosBase.ts: Axios base configuration
- src/libs/axios/helper.ts: shared Axios helper logic
- src/libs/axios/ssiClient.ts: SSI market API client
- src/libs/firebase/: Firebase Admin integration for push notifications
- src/libs/mongoDB/db.ts: MongoDB connection logic

### src/middlewares/

Contains Express middleware used before route handlers run.

Examples:

- src/middlewares/validate.ts: request validation middleware
- src/middlewares/verifyAuth.ts: JWT authentication guard
- src/middlewares/errorHandler.ts: centralized error response handling

### src/models/

Stores Mongoose schemas and persisted domain models.

Examples:

- src/models/user.model.ts
- src/models/notifiation.model.ts

The project also keeps database queries separated from the model definitions by using DAO modules under src/models/daos/.

### src/models/daos/

Data access layer for database operations. These modules isolate Mongoose queries from controller logic.

Examples:

- src/models/daos/user.dao.ts
- src/models/daos/notification.dao.ts

This is a central pattern in the codebase: controllers call DAO functions instead of directly interacting with models.

### src/routes/

Contains route registration for each major feature area. Routes are thin and delegate validation and controller logic.

Examples:

- src/routes/auth.routes.ts
- src/routes/user.routes.ts
- src/routes/notification.routes.ts

### src/services/

Contains service logic when a flow is more than a simple DAO call or when external integrations are abstracted from controllers.

Example:

- src/services/stock.service.ts

This module wraps market data requests to the SSI API.

### src/scripts/

Holds maintenance or batch utility scripts that are not part of the normal request lifecycle.

Examples:

- src/scripts/daily-task.ts
- src/scripts/vnIndexNotifications.ts

### src/test/

Contains automated tests for project behavior and utilities.

Example:

- src/test/utils/dateTime.test.ts

### src/types/

Holds TypeScript interfaces and domain types used across the project.

Examples:

- src/types/user.ts
- src/types/notification.ts
- src/types/stock.ts
- src/types/dateTime.ts

### src/utils/

Reusable helper functions and shared one-off utilities.

Examples:

- src/utils/response.ts: standardized success/error response helpers
- src/utils/dateTime.ts: date utilities
- src/utils/stock.ts: stock-related utility logic

### src/validators/

Contains Zod schemas that validate request payloads before route handlers execute.

Examples:

- src/validators/authSchema.ts
- src/validators/userSchema.ts

## Architectural pattern used by this repo

The folder structure follows a clear layered design:

```text
routes -> validators -> controllers -> services/daos -> models -> database/external APIs
```

This means:

- routes define API surface
- validation checks incoming data
- controllers orchestrate the request
- DAO/service modules handle persistence or integration logic
- models define the persistence schema
- utils and shared libraries provide common helpers

## Feature-based organization

The app is split by feature rather than by technology alone. For example:

- auth feature: routes, controller, validator, JWT helpers
- user feature: route, controller, DAO, model, type
- notification feature: route, controller, DAO, Firebase integration
- stock feature: service and SSI API client integration

This makes the code easier to navigate and extend as new features are added.

## Summary

The repository is organized in a way that keeps responsibilities clear:

- app configuration and runtime concerns live in src/config and src/index.ts
- API surface lives in src/routes
- validation lives in src/validators
- request handling lives in src/controllers
- database access lives in src/models/daos
- domain models live in src/models
- utilities and integrations live in src/utils and src/libs
- documentation and repo guidance live in .github and docs

This structure is consistent with the project’s backend conventions and supports scalable API development.
