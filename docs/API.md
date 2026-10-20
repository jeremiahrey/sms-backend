# SMS Backend — API Reference

This document describes the API endpoints implemented in the `sms-backend` repository.

It reflects the current GitHub source code, not necessarily an older deployed build.

## Base URL

Local development:

```text
http://localhost:5000
```

The current Express application mounts authentication routes at `/auth` and student routes at `/students`. There is no `/api` prefix in the checked source code.

All JSON request bodies should use `Content-Type: application/json`.

## Authentication

Authentication routes are rate-limited to **10 requests per 15-minute window** through the Express rate limiter mounted on `/auth`.

### POST `/auth/register`

Registers a new user account.

**Authentication required:** No

Request:

```json
{
  "email": "demo@example.com",
  "password": "ExamplePass1!",
  "firstName": "Demo",
  "lastName": "User"
}
```

Password requirements enforced by the current validation function:

- Minimum eight characters
- One uppercase letter
- One lowercase letter
- One digit
- One special character

**Success — 201:**

```json
{
  "success": true,
  "statusCode": 201,
  "message": "User Registered Successfully!",
  "data": {
    "userId": 1,
    "email": "demo@example.com",
    "firstName": "Demo",
    "lastName": "User",
    "createdAt": "2025-10-01T07:00:00.000Z"
  }
}
```

Possible failures include missing credentials, invalid password strength, and duplicate email (400).

Note: Registration is publicly accessible. The current application does not assign or enforce administrator roles. First-name and last-name validation also needs improvement.

### POST `/auth/login`

Authenticates a registered account.

**Authentication required:** No

Request:

```json
{
  "email": "demo@example.com",
  "password": "ExamplePass1!",
  "rememberMe": true
}
```

**Success — 200:**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Login Successfully!",
  "data": {
    "accessToken": "<jwt_access_token>",
    "user": {
      "userId": 1,
      "email": "demo@example.com",
      "firstName": "Demo",
      "lastName": "User",
      "createdAt": "2025-10-01T07:00:00.000Z"
    }
  }
}
```

When `rememberMe` is true, the server also sets the `refreshToken` cookie.

The access token expires after 15 minutes. The refresh-token lifetime defaults to seven days.

**Common errors:** Missing credentials (400), invalid credentials (401), rate limit exceeded (429).

### POST `/auth/refresh`

Rotates an existing refresh token and issues a new access token.

**Authentication required:** Valid `refreshToken` cookie

No request body or bearer access token is required.

**Success — 200:**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Token refreshed successfully",
  "data": {
    "accessToken": "<new_jwt_access_token>",
    "userId": 1
  }
}
```

A new refresh-token cookie is issued.

**Common errors:**

| Status | Reason                             |
| ------ | ---------------------------------- |
| 400    | No refresh token provided          |
| 403    | Invalid or revoked refresh token   |
| 403    | Expired refresh token              |
| 429    | Authentication rate limit exceeded |

### POST `/auth/logout`

Revokes the refresh token associated with the request cookie.

**Authentication required:** Valid `refreshToken` cookie

No JSON request body or bearer token is required.

**Success — 200:**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Logged out successfully!",
  "data": {}
}
```

The server also clears the refresh-token cookie.

A missing, revoked, or already-invalidated session results in a 400 response in the current implementation.

## Student Management

All student routes require a JWT access token.

```http
Authorization: Bearer <access_token>
```

The JWT middleware verifies the token before allowing access to the controller.

### Student Fields

| Field        | Type           | Notes                         |
| ------------ | -------------- | ----------------------------- |
| studentId    | Integer        | Database-generated identifier |
| prefix       | Enum           | MR, MS, DR, OTHER             |
| customPrefix | String or null | Used with OTHER               |
| firstName    | String         | Required on creation          |
| lastName     | String         | Required on creation          |
| mobile       | String         | 11 digits beginning with 09   |
| email        | String         | Unique                        |
| createdAt    | DateTime       | Database-generated            |
| updatedAt    | DateTime       | Automatically updated         |

The student ID sequence is configured to start at 1000 by a migration. Existing records and sequence state may affect actual IDs.

### POST `/students/create`

Creates a new student record.

**Request:**

```json
{
  "prefix": "MR",
  "firstName": "John",
  "lastName": "Doe",
  "mobile": "09123456789",
  "email": "john.doe@example.com"
}
```

To use a custom title:

```json
{
  "prefix": "OTHER",
  "customPrefix": "Prof.",
  "firstName": "Jane",
  "lastName": "Doe",
  "mobile": "09987654321",
  "email": "jane.doe@example.com"
}
```

Prefix validation accepts case-insensitive input and normalizes it to uppercase.

`customPrefix` is required for `OTHER` and must not be provided as a nonempty value with another prefix.

**Success — 201:**

```json
{
  "success": true,
  "statusCode": 201,
  "message": "New Student Created Successfully!",
  "data": {
    "studentId": 1000,
    "prefix": "MR",
    "customPrefix": null,
    "firstName": "John",
    "lastName": "Doe",
    "mobile": "09123456789",
    "email": "john.doe@example.com",
    "createdAt": "2025-10-01T08:00:00.000Z",
    "updatedAt": "2025-10-01T08:00:00.000Z"
  }
}
```

Common validation errors (400) include missing fields, invalid prefix, invalid email, duplicate email, and invalid mobile format.

### GET `/students`

Returns student records with optional searching, pagination, and sorting.

**Default query behavior:**

- Page: 1
- Limit: 10
- Sort field: createdAt
- Sort order: desc

**Example request:**

```http
GET /students
```

**Example success response — 200:**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Students retieved successfully!",
  "data": {
    "total": 1,
    "page": 1,
    "totalPages": 1,
    "limit": 10,
    "students": [
      {
        "studentId": 1000,
        "prefix": "MR",
        "customPrefix": null,
        "firstName": "John",
        "lastName": "Doe",
        "mobile": "09123456789",
        "email": "john.doe@example.com",
        "createdAt": "2025-10-01T08:00:00.000Z",
        "updatedAt": "2025-10-01T08:00:00.000Z"
      }
    ]
  }
}
```

The spelling of `retieved` matches the current controller response.

The current controller returns 404 when the student collection is empty or a search has no results.

### GET `/students/:id`

Returns one student by ID.

```http
GET /students/1000
```

**Success — 200:** Returns the student object inside the standard `data` property.

**Common errors:**

- Invalid ID (400)
- Student not found (404)
- Invalid or missing JWT (401)

### PUT `/students/update/:id`

Updates selected fields of an existing student record.

```http
PUT /students/update/1000
```

**Example request:**

```json
{
  "firstName": "Jonathan",
  "mobile": "09987654321"
}
```

Allowed fields:

- prefix
- customPrefix
- firstName
- lastName
- mobile
- email

The endpoint supports partial updates using PUT. `studentId`, `createdAt`, and `updatedAt` are not permitted in the update payload.

Nonempty email and mobile values are validated. Empty string values are removed from the update object.

**Success — 200:** Returns the updated student record.

**Important implementation notes:**

- Custom-prefix validation depends on whether `prefix` is supplied in the update.
- A `customPrefix`-only update is not consistently rejected by the current service.
- An update targeting a nonexistent student is not explicitly translated into a 404 by the current service; Prisma errors may reach the global error handler.

### DELETE `/students/delete/:id`

Permanently removes a student record.

```http
DELETE /students/delete/1000
```

**Success — 200:** Returns the deleted student record.

**Common errors:**

- Invalid ID (400)
- Student not found (404)
- Missing or invalid JWT (401)

There is no soft-delete mechanism in the current implementation.

## Search, Pagination, and Sorting

All query options are supplied to `GET /students`.

| Parameter | Default   | Description                                     |
| --------- | --------- | ----------------------------------------------- |
| search    | None      | Matches first name, last name, email, or mobile |
| page      | 1         | Requested page                                  |
| limit     | 10        | Records per page                                |
| sortBy    | createdAt | Field used for sorting                          |
| order     | desc      | asc or desc                                     |

**Search:**

```http
GET /students?search=Doe
```

**Pagination:**

```http
GET /students?page=2&limit=5
```

**Sorting:**

```http
GET /students?sortBy=firstName&order=asc
```

**Combined query:**

```http
GET /students?search=Doe&page=1&limit=5&sortBy=firstName&order=asc
```

Search is case-insensitive and splits multi-word input for matching.

The current implementation does not strictly validate numeric pagination values or restrict `sortBy` through an explicit allowlist.

The response can contain string values for `page` and `limit` when supplied through URL query parameters.

## Response Formats

### Successful response

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Operation completed successfully.",
  "data": {}
}
```

### Global error-handler response

```json
{
  "success": false,
  "statusCode": 400,
  "message": "Validation error."
}
```

The global error handler can include an additional `error` field. In development, it can also include a stack trace.

Authentication middleware sometimes returns errors directly using a helper that includes an `error` object.

### Status Codes

| Status | Meaning                            |
| ------ | ---------------------------------- |
| 200    | Success                            |
| 201    | Resource created                   |
| 400    | Invalid request                    |
| 401    | Missing or invalid authentication  |
| 403    | Invalid or expired refresh token   |
| 404    | Resource not found                 |
| 429    | Authentication rate limit exceeded |
| 500    | Internal server error              |

## Postman

The Postman collection can be used to explore and manually test these endpoints.

The Postman collection is being updated to reflect the current implementation. A public link will be added once its endpoints, examples, and credentials have been reviewed.

Before public release, synchronize the collection's base URL, remove hardcoded tokens, correct invalid sample requests, and separate unimplemented Grade routes.

This documentation describes the checked GitHub implementation; it does not represent the results of a fresh live API test.
