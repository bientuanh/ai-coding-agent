# Authentication Architecture

## Overview

Fin Mate API uses password-based authentication with short-lived access tokens and longer-lived refresh tokens. The same authentication flow supports web clients and mobile clients:

- Web clients authenticate with secure HTTP-only cookies.
- Mobile clients authenticate with a Bearer token in the `Authorization` header.
- Protected routes use the shared `verifyAuth` middleware.
- JWT creation and verification are centralized in `src/libs/jwt.ts`.
- Password hashing and comparison use `bcryptjs`.

## Authentication Components

| Component                            | Responsibility                                                          |
| ------------------------------------ | ----------------------------------------------------------------------- |
| `src/routes/auth.routes.ts`          | Registers signup, signin, signout, and refresh-token endpoints          |
| `src/controllers/auth.controller.ts` | Validates credentials, creates users, issues tokens, and clears cookies |
| `src/middlewares/validate.ts`        | Parses request bodies with Zod schemas                                  |
| `src/middlewares/verifyAuth.ts`      | Reads and verifies access tokens for protected routes                   |
| `src/libs/jwt.ts`                    | Signs and verifies access and refresh JWTs                              |
| `src/models/daos/user.dao.ts`        | Finds users by email or ID and creates users                            |
| `src/models/user.model.ts`           | Defines user persistence and hides passwords by default                 |
| `src/config/appConfig.ts`            | Loads JWT secrets, lifetimes, CORS, and runtime settings                |
| `src/utils/response.ts`              | Returns standardized success and error responses                        |

## Request Flow

```text
Client
	|
	v
POST /api/auth/signup or /api/auth/signin
	|
	v
Zod validation middleware
	|
	v
Authentication controller
	|
	+--> User DAO --> MongoDB
	|
	+--> bcrypt password comparison/hash
	|
	+--> JWT access + refresh token generation
	|
	+--> Web cookies or mobile JSON response
```

Protected requests follow this path:

```text
Request
	|
	v
verifyAuth
	|
	+--> accessToken cookie, or Authorization: Bearer <token>
	|
	+--> verifyToken()
	|
	+--> req.user = { userId, email, role }
	|
	v
Protected controller
```

## Signup

Endpoint: `POST /api/auth/signup`

The route applies `validate(createUserSchema)` before `signupHandler`.

Required input:

```json
{
  "firstName": "An",
  "lastName": "Nguyen",
  "email": "an@example.com",
  "password": "at-least-6-characters"
}
```

Flow:

1. Zod validates names, email format, password length, and the optional role enum.
2. The controller checks that required values are present.
3. `getUserByEmail(email)` checks for an existing account without selecting the password.
4. The password is hashed with `bcrypt.hash(password, 10)`.
5. A user is saved with the default application role `User`.
6. The endpoint returns a standard success response without issuing tokens.

Responses include:

- `200` when the user is created successfully.
- `400` with `MISSING_FIELDS` when required values are absent.
- `409` with `EMAIL_ALREADY_EXIST` when the email is already registered.
- `400` with `DATA_NOT_VALID` when Zod validation fails.

## Signin

Endpoint: `POST /api/auth/signin`

The route applies `validate(signinSchema)` before `signinHandler`.

Flow:

1. The controller reads `email`, `password`, and the optional `isMobile` flag.
2. `getUserByEmail(email, true)` explicitly selects the password field, which is hidden by default in the user model.
3. `bcrypt.compare()` checks the submitted password against the stored hash.
4. The controller creates an access token and a refresh token with this payload:

```json
{
  "userId": "user-id",
  "email": "an@example.com",
  "role": "User"
}
```

5. The response format depends on the client mode.

### Web Client

When `isMobile` is absent or false, the API sets two cookies:

| Cookie         | Default lifetime | Attributes                               |
| -------------- | ---------------- | ---------------------------------------- |
| `accessToken`  | 15 minutes       | `httpOnly`, `secure`, `sameSite: "none"` |
| `refreshToken` | 7 days           | `httpOnly`, `secure`, `sameSite: "none"` |

The response body contains the user data, but not the token values.

### Mobile Client

When `isMobile` is true, the API returns the user, `accessToken`, and `refreshToken` in the JSON response. The client should send the access token on protected requests as:

```http
Authorization: Bearer <accessToken>
```

Authentication failures use `401` with `INVALID_INPUT` and the generic message `Email or password is not correct`, avoiding disclosure of whether the email or password was wrong.

## Access Token Verification

Protected routes apply `verifyAuth`, for example the user and notification route groups. The middleware checks credentials in this order:

1. `req.cookies.accessToken` for web clients.
2. `Authorization: Bearer <token>` for mobile clients.

If an access token is found, `verifyToken()` validates its signature and expiration using `JWT_SECRET`. On success, the decoded payload is assigned to `req.user` and the request continues.

Failure behavior:

- No access token but a refresh cookie exists: `401 TOKEN_EXPIRED`.
- No access or refresh token: `401 UNAUTHORIZED`.
- Invalid or expired access token: `401 INVALID_TOKEN`.

The middleware does not automatically refresh an expired access token. The client must call the refresh endpoint and retry the original request.

## Refresh Token

Endpoint: `POST /api/auth/refresh-token`

Current implementation:

1. Reads `refreshToken` from `req.cookies`.
2. Verifies it with the separate `JWT_REFRESH_SECRET`.
3. Creates a new access token using the decoded `userId`, `email`, and `role`.
4. Sets a new `accessToken` cookie with the configured access-token lifetime.
5. Returns a standard success response.

The current handler is cookie-based. Although mobile signin returns a refresh token in JSON, the refresh endpoint does not currently read a refresh token from the request body or an authorization header. Mobile refresh support therefore requires a follow-up implementation if mobile clients cannot provide the refresh token as a cookie.

When the refresh cookie is missing, the endpoint returns `401 MISSING_FIELDS`. Verification exceptions are forwarded to the global error handler.

## Signout

Endpoint: `POST /api/auth/signout`

The handler clears the `accessToken` and `refreshToken` cookies, then returns a standard success response. There is no server-side token blacklist or refresh-token persistence, so already-issued JWTs are invalidated by expiration rather than revocation storage.

## Configuration

Authentication configuration is loaded from environment variables in `src/config/appConfig.ts`:

| Variable                        | Purpose                                       | Default                 |
| ------------------------------- | --------------------------------------------- | ----------------------- |
| `JWT_SECRET`                    | Access-token signing and verification secret  | None                    |
| `JWT_REFRESH_SECRET`            | Refresh-token signing and verification secret | None                    |
| `ACCESS_TOKEN_LIFETIME_MINUTES` | Access-token lifetime                         | `15`                    |
| `REFRESH_TOKEN_LIFETIME_DAYS`   | Refresh-token lifetime                        | `7`                     |
| `CLIENT_URLS`                   | Allowed CORS client origins                   | `http://localhost:8081` |

`cookie-parser` is registered before routes, and CORS enables credentials so web clients can send authentication cookies.

## User Data Protection

The user model uses `select: false` for `password`, so normal user queries do not include it. Signin explicitly opts in with `.select("+password")`. The model's JSON transform exposes `id` and removes `_id` and the Mongoose version key.

Authentication code must continue to follow these rules:

- Never store or log plaintext passwords.
- Never log access tokens, refresh tokens, JWT secrets, or service credentials.
- Keep secrets in environment variables, not source files.
- Keep web tokens HTTP-only and secure.
- Do not return the password field in API responses.
- Use generic credential-failure messages.

## Current Limitations

- Refresh tokens are not stored server-side and cannot be revoked individually.
- The refresh endpoint accepts only the refresh cookie, despite mobile signin returning a refresh token in JSON.
- Cookies are configured with `secure: true`, which requires HTTPS in deployed or browser-based environments.
- The controller's invalid refresh-token branch is only reached if `verifyRefreshToken()` returns a falsy value; JWT verification errors are instead passed to the global error handler.
