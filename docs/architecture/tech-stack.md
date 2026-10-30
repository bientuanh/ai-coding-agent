# Fin Mate API Tech Stack

## Overview

Fin Mate API is a TypeScript backend for user authentication, user data, Vietnamese stock-market insights, and push notifications. It exposes an Express HTTP API and also includes a standalone daily task for generating VN-Index notifications.

## Core Stack

| Area               | Technology                    | Usage                                                    |
| ------------------ | ----------------------------- | -------------------------------------------------------- |
| Runtime            | Node.js                       | Runs the API and standalone scripts                      |
| Language           | TypeScript 5.8                | Strictly typed application source                        |
| HTTP framework     | Express 5.1                   | Middleware, routing, controllers, and error handling     |
| Database           | MongoDB Atlas with Mongoose 8 | Persistence for users and notifications                  |
| Configuration      | `dotenv`                      | Loads environment variables from the runtime environment |
| HTTP client        | Axios 1.13                    | Shared client and SSI market-data requests               |
| Validation         | Zod 3.25                      | Request-body validation at route boundaries              |
| Authentication     | `jsonwebtoken` and `bcryptjs` | JWT access/refresh tokens and password hashing           |
| Push notifications | Firebase Admin SDK            | Firebase Cloud Messaging topic subscriptions and sends   |

## Application Structure

The source follows a layered backend structure:

```text
src/
	index.ts              Express bootstrap and route registration
	config/               Environment-backed application configuration
	routes/               HTTP route definitions
	controllers/          Request handling and response orchestration
	middlewares/          Validation, authentication, and error handling
	models/               Mongoose schemas and DAO modules
	services/             External-domain services, including SSI market data
	libs/                 JWT, MongoDB, Axios, and Firebase integrations
	validators/           Zod request schemas
	utils/                Shared response, date, and stock-analysis helpers
	scripts/              Standalone market-notification tasks
	types/                Shared TypeScript domain types
	constants/            Error codes, messages, stock groups, and FCM topics
```

## HTTP API

The application is created in `src/index.ts` and uses:

- `cookie-parser` for web authentication cookies.
- `cors` with configured client origins and credentials enabled.
- `express.json()` for JSON request bodies.
- A centralized error handler registered after all routes.

The current route groups are:

| Prefix               | Responsibilities                                                                 | Authentication                                                    |
| -------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `/api`               | Health-style greeting endpoint                                                   | Public                                                            |
| `/api/auth`          | Signup, signin, signout, and access-token refresh                                | Signup/signin/signout are public; refresh uses the refresh cookie |
| `/api/users`         | List users, fetch a user, and fetch the current user                             | Protected by `verifyAuth`                                         |
| `/api/notifications` | Paginated, latest, unread, detail, create, read, and FCM subscription operations | Protected by `verifyAuth`                                         |

Controllers return a consistent response shape through `successResponse()` and `errorResponse()`:

```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": {}
}
```

Validation failures return `success: false`, an error code, a message, and optional field errors.

## Authentication

Authentication supports web and mobile clients:

- Passwords are hashed with `bcryptjs` before they are stored.
- Access and refresh tokens are signed with separate JWT secrets.
- Access tokens default to a 15-minute lifetime; refresh tokens default to 7 days.
- Web clients receive `httpOnly`, `secure`, `sameSite: "none"` cookies.
- Mobile clients receive access and refresh token values in the signin response when `isMobile` is true.
- Protected routes accept the access token from the web cookie or an `Authorization: Bearer <token>` header.

## Data Layer

MongoDB access is centralized through Mongoose:

- `User` stores names, a unique email, a selectively hidden password, role, image, and timestamps.
- `Notification` stores the notification type, title, body, market status, read timestamp, and timestamps.
- DAO modules provide the query and persistence operations used by controllers.
- Mongoose JSON transforms expose `id` and remove the MongoDB `_id` and version key from responses.

## Market Data and Notifications

The SSI integration uses Axios with a configured base URL. It obtains an SSI access token, adds it to requests, and refreshes and retries once after a `401` response. The stock service currently requests:

- Daily VN-Index data.
- Daily OHLC data for individual stocks.
- Daily stock-price data.

The `daily-task` script connects to MongoDB and generates daily, weekly, and monthly VN-Index notifications. It combines index movement with potential-stock and sector/ecosystem analysis, sends the result through Firebase Cloud Messaging to the `vnindex` topic, and stores the notification in MongoDB. The script disconnects from MongoDB when it finishes.

## Configuration

Runtime configuration is centralized in `src/config/appConfig.ts`. The main environment variables are:

- `NODE_ENV`, `PORT`, `BASE_URL`, and `CLIENT_URLS`.
- `MONGO_DB_URI`.
- `JWT_SECRET`, `JWT_REFRESH_SECRET`, `ACCESS_TOKEN_LIFETIME_MINUTES`, and `REFRESH_TOKEN_LIFETIME_DAYS`.
- `SSI_API_URL`, `SSI_CONSUMER_ID`, and `SSI_CONSUMER_SECRET`.
- `FIREBASE_SERVICE_ACCOUNT` as a JSON service-account value.

Secrets must be supplied through the environment and must not be committed to the repository.

## Build and Quality Tooling

| Concern                  | Tool or command                                             |
| ------------------------ | ----------------------------------------------------------- |
| Compile                  | `npm run build`                                             |
| Type checking            | `npm run typecheck`                                         |
| Local development        | `npm run dev` with Nodemon                                  |
| Production start         | `npm start`                                                 |
| Tests                    | Jest with `ts-jest` and the Node test environment           |
| Linting                  | ESLint 9 with TypeScript, Prettier, and import-sort plugins |
| Formatting               | Prettier                                                    |
| Dependency audit         | `npm run dep-audit`                                         |
| Secret scanning          | `npm run lint:secrets`                                      |
| Market notification task | `npm run daily-task`                                        |

TypeScript targets ES2020, emits CommonJS output to `dist/`, and uses strict type checking. Tests currently live under `src/test/` and cover shared date and working-day calculations used by market notifications.
