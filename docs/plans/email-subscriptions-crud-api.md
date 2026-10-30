# Email Subscriptions CRUD API

## Status

`Completed`

## Goal

Add an email subscription API so a website form (email input + subscribe button) can store subscriber emails and support future outbound communication (email campaigns or product notifications).

## Scope

### In Scope

- Add a new subscription domain with type, model, DAO, validator, controller, and routes.
- Provide a public endpoint for subscribe from UI form.
- Provide controlled endpoints to list subscribers and update subscription status for operations team.
- Keep standardized API responses using shared response helpers.
- Add focused tests for validation and controller/DAO behavior.

### Out Of Scope

- Actual email sending or notification campaign execution.
- Background job or queue setup for campaign dispatch.
- Third-party ESP integration (SendGrid, SES, Mailchimp, etc.).
- UI implementation details.

## Current Context

- The project uses Express + TypeScript + MongoDB (Mongoose) + Zod.
- Existing architecture follows route -> validation -> controller -> DAO -> model.
- Responses are standardized with `successResponse` and `errorResponse`.
- Existing error codes already include `EMAIL_ALREADY_EXIST`, `BAD_REQUEST`, and `INTERNAL_SERVER_ERROR`.
- There is no current dedicated subscriber model in `src/models/`.

## Assumptions And Decisions

- Public subscribe endpoint does not require auth because it is submitted from a public UI form.
- The same email can re-subscribe if it was unsubscribed before (reactivation flow).
- Soft state is preferred over hard delete for auditability (`status` and timestamps).
- Email uniqueness is enforced case-insensitively using normalized lowercase email.
- Admin or internal operations endpoints are protected by `verifyAuth`.
- Double opt-in is not included in this phase; this is single-step opt-in.

## Database Design

Collection: `email_subscriptions`

Document shape:

- `email: string` (required, normalized lowercase)
- `status: "subscribed" | "unsubscribed" | "bounced"` (default: `subscribed`)
- `source: "web" | "mobile" | "admin"` (default: `web`)
- `subscribedAt: Date | null` (set when status becomes `subscribed`)
- `unsubscribedAt: Date | null` (set when status becomes `unsubscribed`)
- `lastNotificationAt: Date | null` (reserved for future campaign tracking)
- `tags: string[]` (optional, default empty, for segmentation later)
- `createdAt: Date` (mongoose timestamps)
- `updatedAt: Date` (mongoose timestamps)

Indexes:

- Unique index on `email`.
- Compound index on `{ status: 1, createdAt: -1 }` for list/filter + pagination.

Mongoose notes:

- Use `toJSON` transform to expose `id` and hide `_id`, `__v`.
- Use enum constraints for `status` and `source`.

## API Contract Draft

Base path: `/api/email-subscriptions`

Public endpoints:

- `POST /api/email-subscriptions/subscribe`
  - body: `{ email: string }`
  - behavior:
    - create new record with `status=subscribed`, or
    - if existing unsubscribed record: switch back to subscribed.

Protected endpoints (verifyAuth):

- `GET /api/email-subscriptions?page=1&limit=20&status=subscribed`
- `PATCH /api/email-subscriptions/:id/unsubscribe`
- `PATCH /api/email-subscriptions/:id/resubscribe`
- Optional in same phase: `GET /api/email-subscriptions/:id`

Response rules:

- Use `successResponse` and `errorResponse` helpers.
- Use `409` for duplicate active subscription attempts.
- Use `404` when resource is missing.

## Implementation Steps

### Step 1: Add domain types

- Status: `Completed`
- Files: `src/types/emailSubscription.ts`
- Changes: Add `EmailSubscription`, `SubscriptionStatus`, and `SubscriptionSource` types.
- Validation: `npm run typecheck`
- Done when: Types are reusable in model, DAO, and controllers without `any`.

### Step 2: Add Mongoose model

- Status: `Completed`
- Files: `src/models/emailSubscription.model.ts`
- Changes:
  - Add schema fields from Database Design.
  - Add unique index on normalized `email`.
  - Add enum constraints, timestamps, and `toJSON` transform.
- Validation: `npm run typecheck`
- Done when: Model exports successfully and schema includes required indexes.

### Step 3: Add DAO layer

- Status: `Completed`
- Files: `src/models/daos/emailSubscription.dao.ts`
- Changes:
  - `findByEmail(email)`
  - `subscribeByEmail(email, source)` (create or reactivate)
  - `getSubscriptionsWithPagination(page, limit, status?)`
  - `getSubscriptionById(id)`
  - `updateStatusById(id, status)`
- Validation: focused DAO tests and `npm run typecheck`
- Done when: Controllers can perform all required operations without direct queries.

### Step 4: Add validation schemas

- Status: `Completed`
- Files: `src/validators/emailSubscriptionSchema.ts`
- Changes:
  - `subscribeEmailSchema` for public endpoint.
  - query schema for pagination/filter.
  - param/body schema for status transition endpoints.
- Validation: validator unit tests
- Done when: Invalid emails and malformed query values are rejected at route boundary.

### Step 5: Add controller handlers

- Status: `Completed`
- Files: `src/controllers/emailSubscription.controller.ts`
- Changes:
  - `subscribeEmailHandler`
  - `getEmailSubscriptionsHandler`
  - `getEmailSubscriptionByIdHandler`
  - `unsubscribeEmailHandler`
  - `resubscribeEmailHandler`
  - Handle 404/409 and use standardized responses.
- Validation: controller unit tests (success + failure paths)
- Done when: Handlers call DAO only and all responses follow project contract.

### Step 6: Register routes

- Status: `Completed`
- Files: `src/routes/emailSubscription.routes.ts`
- Changes:
  - Add route mapping with public/protected split.
  - Attach `validate(...)` middleware for each route.
  - Ensure fixed routes are defined before `/:id` routes.
- Validation: route/controller integration smoke tests
- Done when: Endpoints are reachable and validation middleware runs.

### Step 7: Mount route in app entry

- Status: `Completed`
- Files: `src/index.ts`
- Changes: Mount `app.use("/api/email-subscriptions", emailSubscriptionRoutes)`.
- Validation: app boots and route group is available.
- Done when: New route group is active in server startup.

### Step 8: Add tests and run checks

- Status: `Completed`
- Files:
  - `src/test/validators/emailSubscriptionSchema.test.ts`
  - `src/test/controllers/emailSubscription.controller.test.ts`
  - optional DAO tests if current pattern supports DB mocking
- Changes:
  - Add validation tests for valid/invalid emails.
  - Add handler tests for create, duplicate, reactivate, list pagination, not found.
- Validation:
  - `npm test -- --runInBand`
  - `npm run typecheck`
  - `npm run lint`
- Done when: New tests pass and no type/lint regressions are introduced.

## Risks And Open Questions

- Risk: Without double opt-in, invalid or third-party emails can be subscribed.
- Risk: Unique email index creation may fail if legacy duplicates exist.
- Open question: Do you want unsubscribe via public tokenized link in this phase or later?
- Open question: Should admin list endpoint require admin role check (not just authenticated user)?

## Acceptance Criteria

- [x] Public API can subscribe a valid email from website form.
- [x] Duplicate active subscription returns a clear conflict response.
- [x] Unsubscribed email can be reactivated without creating a second record.
- [x] Protected list endpoint supports pagination and optional status filter.
- [x] Subscription status transitions are persisted with timestamps.
- [x] All new endpoints use route-level validation and standardized response helpers.
- [x] Relevant tests, typecheck, and lint pass.

## Change History

- 2026-09-22: Created draft plan for email subscription API and database design.
- 2026-09-22: Implemented types, model, DAO, validator, controller, routes, and app route mounting.
- 2026-09-22: Added validator and controller tests; ran focused tests, typecheck, and lint successfully.
