# Student Management System — Backend API

A REST API for managing student records, built with **Node.js, Express.js, PostgreSQL, and Prisma ORM**.

The backend handles account authentication, student CRUD operations, searching, pagination, and sorting. It uses JWT access tokens and optional database-backed refresh tokens for session management.

This project was developed to practice backend architecture, relational database integration, authentication, and API design. A separate frontend application consumes the API.

## Features

**Authentication**

- Account registration and login
- Password hashing with bcrypt
- JWT access tokens with a 15-minute expiration
- Optional refresh tokens through the Remember Me feature
- Refresh-token rotation and revocation
- HttpOnly cookies for refresh-token storage
- Rate limiting on authentication routes

**Student Management**

- Create, retrieve, update, and delete student records
- Search by student name, email, or mobile number
- Pagination and sorting
- Student prefix and custom-prefix support
- Email uniqueness checks and input validation

**Backend Structure**

- Separation of routes, controllers, and services
- Reusable authentication and validation utilities
- Centralized error handling
- PostgreSQL integration through Prisma ORM
- Database migrations and development-data seeding

## Tech Stack

| Category       | Technologies                                    |
| -------------- | ----------------------------------------------- |
| Backend        | Node.js, Express.js, JavaScript                 |
| Database       | PostgreSQL                                      |
| ORM            | Prisma                                          |
| Authentication | JWT, bcrypt, HttpOnly cookies                   |
| Middleware     | express-rate-limit, cors, cookie-parser, Morgan |
| Development    | Git, GitHub, Nodemon                            |
| API Testing    | Postman                                         |
| Deployment     | Render                                          |

## My Contributions

I was responsible for the backend development of the Student Management System, including:

- Structuring the application using routes, controllers, services, and middleware
- Implementing student CRUD endpoints and database queries
- Designing Prisma models and managing PostgreSQL migrations
- Building JWT authentication and refresh-token functionality
- Implementing input validation and centralized error handling
- Testing and documenting API behavior using Postman
- Configuring and deploying the backend API to Render

Frontend development was handled separately.

## Architecture

The application separates request handling, business logic, and database access.

```text
HTTP Request
     |
     v
Express Route
     |
     v
Middleware (where applicable)
     |
     v
Controller
     |
     v
Service
     |
     v
Prisma ORM
     |
     v
PostgreSQL
```

Student routes require JWT authentication. Authentication routes use a separate rate limiter.

Controllers handle HTTP requests and responses, while services contain validation and database operations.

For a more detailed explanation of the application structure, database models, and authentication flow, see [Architecture Documentation](docs/ARCHITECTURE.md).

## Project Structure

```text
sms-backend/
├── prisma/
│   ├── migrations/
│   ├── schema.prisma
│   └── seed.js
├── src/
│   ├── config/
│   ├── controllers/
│   ├── middlewares/
│   ├── routes/
│   ├── services/
│   ├── utils/
│   ├── app.js
│   └── server.js
├── docs/
│   ├── API.md
│   └── ARCHITECTURE.md
├── .gitignore
├── package.json
└── package-lock.json
```

The `docs/` directory contains supporting documentation and is intended to be added alongside this README.

## Getting Started

### Prerequisites

- A compatible Node.js and npm installation
- PostgreSQL
- Git

### 1. Clone the repository

```bash
git clone https://github.com/jeremiahrey/sms-backend.git
cd sms-backend
```

### 2. Configure PostgreSQL

Create a local PostgreSQL database and configure a database user with the required permissions.

### 3. Configure environment variables

Create `.env` in the repository root:

```env
DATABASE_URL="postgresql://username:password@localhost:5432/sms_db?schema=public"
JWT_SECRET="replace-with-a-long-random-secret"
PORT=5000
REFRESH_TOKEN_EXPIRES_DAYS=7
FRONTEND_URL="http://localhost:5173"
```

Replace placeholder values with your local configuration.

**Important:** Configure the database before installing dependencies. The current `postinstall` script automatically generates the Prisma client and applies existing migrations.

### 4. Install dependencies

```bash
npm install
```

The current `postinstall` script runs:

```bash
npx prisma generate
npx prisma migrate deploy
```

### 5. Start the development server

```bash
npm run dev
```

The API runs at `http://localhost:5000` unless another port is configured.

A `GET /` request returns an application welcome message.

### 6. Seed development data (optional)

```bash
npm run seed
```

The seed script creates a demonstration account and 50 student records.

Only run the seed script against a disposable development database. It uses a predefined demonstration password and does not represent a production provisioning process.

### Available Scripts

| Command                | Purpose                            |
| ---------------------- | ---------------------------------- |
| `npm run dev`          | Start development server           |
| `npm start`            | Start in production mode           |
| `npm run build:dev`    | Generate Prisma client             |
| `npm run migrate:prod` | Apply existing database migrations |
| `npm run seed`         | Run development seed script        |
| `npm run studio:dev`   | Open development Prisma Studio     |
| `npm run db:reset`     | Reset development database         |

**Warning:** `db:reset` deletes database data.

## API Overview

The paths below match the current GitHub source code.

**Local base URL:** `http://localhost:5000`

### Authentication

| Method | Endpoint         | Description                                       |
| ------ | ---------------- | ------------------------------------------------- |
| POST   | `/auth/register` | Register a user account                           |
| POST   | `/auth/login`    | Log in and receive an access token                |
| POST   | `/auth/refresh`  | Rotate refresh token and issue a new access token |
| POST   | `/auth/logout`   | Revoke the active refresh-token session           |

### Student Management

| Method | Endpoint               | Description                                   |
| ------ | ---------------------- | --------------------------------------------- |
| POST   | `/students/create`     | Create a student                              |
| GET    | `/students`            | Retrieve, search, sort, and paginate students |
| GET    | `/students/:id`        | Retrieve one student                          |
| PUT    | `/students/update/:id` | Update student information                    |
| DELETE | `/students/delete/:id` | Delete a student                              |

All student endpoints require:

```http
Authorization: Bearer <access_token>
```

### Example: Student Search

```http
GET /students?search=Doe&page=1&limit=5&sortBy=firstName&order=asc
```

Query parameters are optional and can be combined.

### Example: Create Student

```http
POST /students/create
Content-Type: application/json
Authorization: Bearer <access_token>
```

```json
{
  "prefix": "MR",
  "firstName": "John",
  "lastName": "Doe",
  "mobile": "09123456789",
  "email": "john.doe@example.com"
}
```

A successful creation returns HTTP `201` with the created student record.

For request bodies, validation rules, response examples, and authentication details, see [API Documentation](docs/API.md).

## API Testing and Postman

Postman was used for manual endpoint testing and API documentation.

The collection contains authentication, student-management, and query examples.

The Postman collection is being updated to reflect the current implementation. A public link will be added once its endpoints, examples, and credentials have been reviewed.

The Postman collection must be synchronized with the current routes, stripped of real credentials and tokens, and made publicly accessible before relying on this link as public documentation.

## Database

The application uses three Prisma models:

- **User:** Stores account information and password hashes
- **Student:** Stores student records, including prefixes and contact information
- **RefreshToken:** Stores hashed refresh-token records and their expiration/revocation state

The `Student.prefix` field uses the enum `MR`, `MS`, `DR`, or `OTHER`. The optional `customPrefix` field supports additional titles.

Database relationships, indexes, and the token lifecycle are described in [Architecture Documentation](docs/ARCHITECTURE.md).

## Deployment

The backend has been configured for deployment to Render.

The production start command is:

```bash
npm start
```

The current installation process also applies existing Prisma migrations through the `postinstall` script.

Production environment variables should be configured through the hosting provider. Database migrations and deployment settings should be verified before deploying to a database with existing records.

The previously documented Render URL may use a different route prefix from the checked GitHub implementation. Confirm the live configuration before using it as a public demo endpoint.

## Project Scope

This project demonstrates backend API development and database integration.

It is an educational and portfolio implementation, not a production-hardened student information system.

The current code authenticates users but does not implement administrator roles or role-based authorization. Additional improvements are needed for input validation, query-parameter restrictions, automated tests, and operational security before real-world use.

## Planned Improvements

- Strengthen request and query validation
- Add role-based authorization
- Add automated API tests
- Improve refresh-token session management
- Introduce OpenAPI documentation
- Add audit logging and additional academic modules

These features are not presented as implemented functionality.

---

**Repository:** [jeremiahrey/sms-backend](https://github.com/jeremiahrey/sms-backend)
