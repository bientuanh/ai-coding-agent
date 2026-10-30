# Business Overview

## 1. Product purpose

This API supports a Fin Mate application that helps users manage account access, receive market notifications, and fetch financial market data. The product is built around a small set of core business flows: authentication, user profile access, notification delivery, and market information retrieval.

The backend is designed to serve both web and mobile clients while keeping a consistent API contract and secure access model.

## 2. Core business domains

### 2.1 User management

User management is the foundation of the product. The system supports:

- creating a new account
- validating required fields
- checking for duplicate email addresses
- securing password storage with bcrypt hashing
- logging users in with JWT-based credentials
- renewing access tokens via refresh tokens
- retrieving current-user details from the authenticated session

This behavior is implemented in:

- src/controllers/auth.controller.ts
- src/controllers/user.controller.ts
- src/models/daos/user.dao.ts
- src/models/user.model.ts

### 2.2 Authentication and session handling

The API uses JWT for authenticated sessions.

The auth flow supports:

- signup with first name, last name, email, and password
- signin with email/password
- issuing an access token and refresh token
- storing tokens as secure cookies for browser-based clients
- returning tokens directly to mobile clients when needed
- validating protected routes with an auth middleware
- refreshing expired access tokens through the refresh-token endpoint
- clearing tokens on signout

This directly maps to the server’s security model and allows multi-client support without duplicating auth logic.

### 2.3 Notification delivery

The notification feature is a business capability for sending and reading market-related updates. It includes:

- device subscription to an FCM topic
- listing notifications with pagination
- retrieving the latest daily, weekly, and monthly notifications
- fetching unread notifications
- reading a notification by ID
- marking a notification as read
- creating notifications for the app’s alert system

This is implemented through the notification controller and DAO layer, and it uses Firebase Cloud Messaging for topic-based push delivery.

### 2.4 Market and stock data integration

The backend also acts as an integration layer for stock and market data from the SSI API. It provides endpoints and service methods that query market information such as:

- daily index values
- daily OHLC stock records
- daily stock prices

This is not the core of the API’s user domain, but it is a key business feature for a finance-related application because it turns external market data into app-ready responses.

## 3. Main business flows

### 3.1 Signup flow

The signup flow is:

1. Receive first name, last name, email, and password
2. Validate payload with Zod
3. Reject missing or invalid fields
4. Check whether the email already exists
5. Hash the password with bcrypt
6. Create a user record in MongoDB
7. Return a success response

### 3.2 Signin flow

The signin flow is:

1. Receive email and password
2. Look up the user including the hash field
3. Verify password with bcrypt
4. Create an access token and refresh token
5. For web clients, store them as secure cookies
6. For mobile clients, return token values directly in the response
7. Return the user payload and token data

### 3.3 Protected user access

Protected routes use JWT verification to determine the current user. Once validated:

- the request object is enriched with user data
- the user can call current-user and identity-related endpoints
- access is denied with auth errors when no valid token is present

### 3.4 Notification lifecycle

The notification lifecycle supports ongoing engagement with the app:

- a device subscribes to the market topic
- the backend or scheduled process creates new notifications
- the client retrieves recent or unread notifications
- the user marks notifications as read
- the system exposes the latest relevant notification types

### 3.5 Market data retrieval

Market data requests are proxied through a service client that handles SSI token refresh automatically. This allows the app to fetch current market information without embedding the external API token logic in each route or controller.

## 4. Business rules reflected in code

Several important rules are present in the implementation:

- Users must provide required signup fields
- Duplicate emails are forbidden
- Passwords are encrypted before saving
- Authenticated routes require valid access tokens
- Refresh tokens are used to generate a new access token
- Notification reads and subscriptions are validated before processing
- Response payloads follow a consistent success/error contract

These rules make the API predictable and support realistic user and account workflows.

## 5. Data model overview

The main business entities are:

### User

A user contains:

- first name
- last name
- email
- password hash
- role
- optional image
- timestamps

### Notification

A notification contains:

- type
- title
- body
- status
- read timestamp
- created timestamp

These data models are represented in:

- src/models/user.model.ts
- src/models/notifiation.model.ts
- src/types/user.ts
- src/types/notification.ts

## 6. External dependencies and business impact

The backend depends on external services in ways that matter to business operations:

- MongoDB for persistent user and notification data
- Firebase Admin for push notification/topic subscriptions
- SSI API for market and stock information
- JWT providers for secure session management

These integrations define the product’s operational capabilities and allow it to deliver both account features and financial monitoring features.

## 7. Product behavior summary

In plain business terms, the API does three main jobs:

1. Protects user identity and access
2. Keeps users informed through notification flows
3. Exposes financial market data for the Fin Mate product experience

This combination makes the backend a secure operational layer for a finance-oriented mobile/web product, with user access and market data as its core value drivers.
