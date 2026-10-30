# Financial Events CRUD API

## Status

`Completed`

## Goal

Create a CRUD API for managing financial events (sự kiện tài chính quan trọng). This allows users to create, read, update, and delete financial events with important dates and notification settings. The API will support:

- Users selecting an event date (e.g., 18/09/2026)
- Users specifying notification advance days (e.g., 7 days)
- The system storing this data for later use by a daily cron job (9h each day) to send notifications

Example: If a user sets an event for 18/09/2026 with 7 days advance notification, the system will send notifications from 11/09/2026 through 18/09/2026.

## Scope

### In Scope

- Create TypeScript type for FinancialEvent in `src/types/financialEvent.ts`
- Create Mongoose schema and model in `src/models/financialEvent.model.ts`
- Create DAO methods in `src/models/daos/financialEvent.dao.ts` (CRUD operations)
- Create Zod validation schemas in `src/validators/financialEventSchema.ts`
- Create controller handlers in `src/controllers/financialEvent.controller.ts`
- Create routes in `src/routes/financialEvent.routes.ts`
- Register the routes in `src/index.ts`
- Add basic Jest tests in `src/test/`

### Out Of Scope

- Cron job implementation (scheduled for later work)
- Firebase notification sending (separate feature)
- Frontend implementation
- User role/permission logic (will use existing verifyAuth middleware)

## Current Context

- Project uses Express + TypeScript + Mongoose + Zod
- Existing patterns: models in `src/models/`, DAOs in `src/models/daos/`, controllers in `src/controllers/`
- Response standardization via `successResponse()` and `errorResponse()` from `src/utils/response.ts`
- Authentication via `verifyAuth` middleware for protected routes
- Pagination pattern used in notifications: page, limit, skip, total, hasMore
- Existing error codes in `src/constants/errorCodes.ts`

## Assumptions And Decisions

1. **FinancialEvent belongs to a user**: The API will track which user created/owns each event via `userId` field and use `verifyAuth` middleware for all protected routes.
2. **Date fields as ISO strings**: `eventDate` stored as Date in MongoDB, serialized as ISO string in responses.
3. **Notification field naming**:
   - `eventDate`: The date of the event
   - `notifyDaysBefore`: Number of days before event to start notifications (0 = day of event only)
4. **Status field**: Events can be active or inactive, allowing users to enable/disable notifications without deletion.
5. **Pagination**: Implement pagination for list endpoint (page, limit) following notification pattern.
6. **Error handling**: Use existing error codes where possible; will check if new codes are needed.
7. **No database unique constraints**: Multiple users can create events for the same date; userId + eventDate combination not enforced as unique to allow flexibility.

## Implementation Steps

### Step 1: Create TypeScript type

- Status: `Completed`
- Files: `src/types/financialEvent.ts`
- Changes: Define `FinancialEvent` interface with fields: `id`, `userId`, `title`, `description` (optional), `eventDate`, `notifyDaysBefore`, `isActive`, `createdAt`, `updatedAt`
- Validation: TypeScript file compiles without errors
- Done when: Type file created and can be imported elsewhere

### Step 2: Create Mongoose Model

- Status: `Completed`
- Files: `src/models/financialEvent.model.ts`
- Changes:
  - Create schema with fields matching FinancialEvent type
  - Add timestamps (createdAt, updatedAt)
  - Add schema.set("toJSON") to expose id and remove \_id, \_\_v
  - Export model using `mongoose.models.FinancialEvent || mongoose.model()`
- Validation: TypeScript compilation, model registration
- Done when: Model file created and type-safe

### Step 3: Create DAO layer

- Status: `Completed`
- Files: `src/models/daos/financialEvent.dao.ts`
- Changes:
  - `getEventById(id: string)`: Fetch single event by ID
  - `getEventsByUserId(userId: string, page: number, limit: number)`: Fetch paginated events for user
  - `createEvent(data: Partial<FinancialEvent>)`: Create new event
  - `updateEventById(id: string, data: Partial<FinancialEvent>)`: Update event
  - `deleteEventById(id: string)`: Delete event
  - `getEventsForNotification(targetDate: Date)`: Get active events where targetDate falls between (eventDate - notifyDaysBefore) and eventDate (for cron job)
- Validation: DAO functions return correct types, async/await usage
- Done when: All DAO methods implemented and tested locally

### Step 4: Create Zod Validation Schemas

- Status: `Completed`
- Files: `src/validators/financialEventSchema.ts`
- Changes:
  - `createFinancialEventSchema`: title (string, min 5), description (string, optional), eventDate (ISO date), notifyDaysBefore (number, 0-30)
  - `updateFinancialEventSchema`: same fields but all optional
- Validation: Zod schemas parse valid and reject invalid data
- Done when: Schemas created and can validate request bodies

### Step 5: Create Controller Handlers

- Status: `Completed`
- Files: `src/controllers/financialEvent.controller.ts`
- Changes:
  - `createEventHandler`: POST, validate schema, extract userId from req.user (via auth middleware), call DAO, return 201 success
  - `getEventsByUserHandler`: GET with pagination (page, limit), extract userId, call DAO, return paginated data
  - `getEventByIdHandler`: GET/:id, verify ownership (event.userId === req.user.id), return event or 404
  - `updateEventHandler`: PATCH/:id, verify ownership, validate update schema, call DAO, return updated event
  - `deleteEventHandler`: DELETE/:id, verify ownership, call DAO, return success
- Validation: Controllers handle errors via try/catch and next(err), use successResponse/errorResponse
- Done when: All handlers implemented and structured correctly

### Step 6: Create Routes

- Status: `Completed`
- Files: `src/routes/financialEvent.routes.ts`
- Changes:
  - POST / → validate(createFinancialEventSchema), verifyAuth, createEventHandler
  - GET / → verifyAuth, getEventsByUserHandler (with pagination query params)
  - GET /:id → verifyAuth, getEventByIdHandler
  - PATCH /:id → validate(updateFinancialEventSchema), verifyAuth, updateEventHandler
  - DELETE /:id → verifyAuth, deleteEventHandler
- Validation: Routes register handlers in correct order (fixed paths before /:id)
- Done when: Routes file created and imports all handlers

### Step 7: Register Routes in Index

- Status: `Completed`
- Files: `src/index.ts`
- Changes: Add `import financialEventsRoutes` and `app.use("/api/financial-events", financialEventsRoutes)` in appropriate location
- Validation: API server starts without errors
- Done when: Route is mounted and accessible at /api/financial-events

### Step 8: Type Checking, Linting, and Testing

- Status: `Completed`
- Files: Various created files
- Changes: Run npm commands to verify code quality
- Validation:
  ```bash
  npm run typecheck     # No TypeScript errors
  npm run lint          # No linting errors
  npm run format        # Code formatted
  npm test -- --runInBand  # Tests pass
  ```
- Done when: All commands pass without errors

## Technical Notes

- **User context**: The `verifyAuth` middleware should populate `req.user` with decoded JWT data (userId)
- **Ownership check**: When fetching/updating/deleting, verify `event.userId === req.user.id` to prevent cross-user access
- **Pagination pattern**: Follow notification model's pattern: `{ data: [], pagination: { page, limit, total, hasMore } }`
- **Response status codes**:
  - 200 for GET/PATCH
  - 201 for POST (create)
  - 204 or 200 for DELETE
  - 400 for validation errors
  - 404 for not found
  - 401 for unauthorized/missing auth
  - 403 for ownership violation (optional: use 404 to hide resource existence)
