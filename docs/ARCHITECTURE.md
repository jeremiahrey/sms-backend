# SMS Backend — Architecture

## Overview

The Student Management System backend uses Node.js, Express.js, Prisma ORM, and PostgreSQL.

The project separates HTTP routing, controller logic, application services, and database access into distinct modules.

This structure helps keep request-handling responsibilities separate from student-management and authentication logic.

## Request Lifecycle

A typical authenticated student request follows this sequence:

```text
Client
  |
  v
Express Application
  |
  v
Student Route
  |
  v
JWT Authentication Middleware
  |
  v
Student Controller
  |
  v
Student Service
  |
  v
Prisma Client
  |
  v
PostgreSQL
```

Service errors are passed to the global Express error handler. Authentication middleware can also return errors directly.

Authentication endpoints follow a similar controller/service structure without requiring an access-token guard.

## Application Layers

### Routes

Located in `src/routes/`.

The application has two route modules:

- `authRoutes.js`
- `studentRoutes.js`

Authentication routes support registration, login, refresh, and logout.

Student routes expose the CRUD endpoints and apply JWT authentication middleware.

### Controllers

Located in `src/controllers/`.

Controllers retrieve request parameters and bodies, invoke services, and send responses using the shared success-response helper.

The main controller files are:

- `authControllers.js`
- `studentControllers.js`

### Services

Located in `src/services/`.

Services contain most of the application and database logic.

`authService.js` handles:

- Account registration
- Login and password verification
- Access-token generation
- Refresh-token generation and rotation
- Refresh-token revocation

`studentService.js` handles:

- Student creation
- Student retrieval and search
- Pagination and sorting
- Student updates and deletion
- Field normalization and selected validation rules

### Middleware

Located in `src/middlewares/`.

- `authMiddlewares.js` — verifies JWT access tokens
- `validate.js` — contains reusable validation functions
- `errorHandler.js` — formats errors forwarded through Express

Authentication rate limiting and CORS are configured in `src/app.js`.

### Utilities

Located in `src/utils/`.

- `normalizeStudentData.js` — trims and normalizes selected student fields
- `response.js` — shared success and error-response helpers
- `token.js` — generates opaque refresh tokens and computes SHA-256 token hashes

## Database Design

The PostgreSQL schema is maintained using Prisma.

### User

Stores user identity information and bcrypt password hashes.

One user can have multiple associated refresh-token records.

### Student

Stores student profile and contact information.

The `prefix` field uses a Prisma enum with four values:

- MR
- MS
- DR
- OTHER

The optional `customPrefix` field supports titles outside those standard options.

### RefreshToken

Stores hashed refresh-token records.

Each record includes:

- Token hash
- Related user ID
- Creation timestamp
- Expiration timestamp
- Revocation state

The relationship to User uses cascading deletion.

### Indexes

The schema defines indexes on:

- Student.createdAt
- RefreshToken.userId
- RefreshToken.expiresAt
- RefreshToken.revoked + expiresAt

User email, student email, and refresh-token hash are unique.

## Authentication Design

### Access Tokens

The backend generates JWT access tokens after successful login.

Each access token:

- Includes the user's ID and email
- Is signed using `JWT_SECRET`
- Expires after 15 minutes

Protected routes expect the token in the Authorization header.

### Refresh Tokens

The Remember Me feature optionally creates a refresh token.

Refresh tokens are generated using cryptographically secure random bytes.

The raw token is returned to the client as an HttpOnly cookie, while its SHA-256 hash is saved in PostgreSQL.

The default expiration is seven days.

### Rotation

The refresh endpoint:

1. Reads the refresh-token cookie.
2. Hashes the supplied token.
3. Looks up the matching database record.
4. Checks its revocation and expiration state.
5. Generates a replacement refresh token.
6. Updates the stored record with the new token hash and expiration.
7. Issues a new JWT access token and refresh-token cookie.

### Logout

Logout looks up the current refresh-token record and marks it revoked.

On successful logout, the controller clears the refresh-token cookie.

This is database-backed session management, but the implementation has not been independently verified as production-hardened.

## Student Query Design

`GET /students` uses Prisma `count` and `findMany` operations within a Prisma transaction.

The service builds optional search conditions against student names, email addresses, and mobile numbers.

Pagination is implemented using `skip` and `take`.

Sorting uses a dynamically selected field passed to Prisma's `orderBy`.

The current implementation permits search, pagination, and sorting to be combined in the same request.

## Error Handling

Controllers pass most service errors to the global error handler.

Service errors can contain a `statusCode` and a descriptive message.

The global handler generates JSON errors and conditionally includes stack traces when `NODE_ENV=development`.

JWT middleware returns authentication errors directly through a shared helper.

This results in some differences in the presence of an `error` property across error responses.

## Implementation Boundaries

The following limitations are present in the checked codebase and are candidates for improvement:

**Authorization:** Authentication is implemented, but no administrator role or role-based permission check exists.

**Registration validation:** The first-name and last-name requirement is not consistently enforced individually before database insertion.

**Query validation:** Pagination parameters and sortable fields need stricter validation.

**Student updates:** Cross-field prefix validation and missing-record error handling need improvement.

**Tests:** Automated integration and regression test suites are not established in the checked repository.

**Deployment:** The current `postinstall` script applies database migrations during dependency installation. A separate migration step would provide clearer deployment control.

## Future Architecture Improvements

Potential improvements include:

- Dedicated request-validation schemas
- Role-based authorization middleware
- More consistent API error handling
- Automated tests for authentication and CRUD flows
- Explicit pagination limits and sorting allowlists
- Improved database migration/deployment separation
- More complete refresh-token lifecycle testing

These describe planned work rather than features already implemented.

---

See the [API Reference](API.md) for endpoint details or return to the [main README](../README.md).
