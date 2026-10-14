# Backend Documentation

This document describes the backend architecture, database layer, API routes, middleware, authentication, and data-fetching patterns for the application.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Database Layer](#2-database-layer)
   - [Connection](#connection)
   - [Schema](#schema)
   - [Types](#types)
3. [API Routes & Endpoints](#3-api-routes--endpoints)
4. [Middleware & Authentication](#4-middleware--authentication)
   - [Middleware](#middleware)
   - [Authentication](#authentication)
5. [Data Fetching & Forms](#5-data-fetching--forms)
   - [Data Fetching](#data-fetching)
   - [Forms](#forms)

---

## 1. Architecture Overview

This is a **Next.js 14.2.0** application built on **React 18.0.0** that uses the **App Router** paradigm, with routes located in the `src/app` directory. The entire codebase is written in **TypeScript**, providing end-to-end type safety across both the frontend and backend.

The backend logic is co-located within the Next.js App Router API routes (`src/app/api/`). Rather than maintaining a separate backend service, the server-side logic lives inside the same Next.js application as the frontend. The API routes under `src/app/api/` act as the server-side backend, handling HTTP requests, business logic, and database interactions.

The frontend is styled with **Tailwind CSS** and composed from **shadcn/ui** components.

### Co-located Architecture Pattern

In this co-located architecture, the frontend and backend share a single Next.js application and repository. This approach offers several benefits:

- **Single deployment unit** — The frontend and API are deployed together, simplifying CI/CD and infrastructure.
- **Shared types** — TypeScript interfaces can be imported directly by both the client and server code, reducing drift between contracts.
- **Reduced latency** — API route handlers run in the same process as the rendered pages, avoiding an extra network hop between a separate API server.
- **Unified developer experience** — One toolchain, one configuration, and one set of dependencies for the full stack.

The App Router API routes in `src/app/api/` expose route handlers (using the Web `Request`/`Response` APIs) that serve as the backend endpoints consumed by the frontend.

---

## 2. Database Layer

The database layer is responsible for connecting to PostgreSQL, defining the schema, and providing TypeScript types that map database rows to application objects.

### Connection

- The database client is configured in `src/lib/db.ts`.
- It uses the `pg` package (node-postgres) for PostgreSQL connectivity.
- The connection reads the `DATABASE_URL` environment variable to obtain the connection string.

A typical configuration creates a connection `Pool` from the `DATABASE_URL`:

```typescript
import { Pool } from "pg";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export default pool;
```

### Schema

- The SQL schema is defined in `database/schema.sql`.
- It defines a `users` table to support the user management features seen in the UI.
- The schema file should be run against the PostgreSQL database during setup, for example:

  ```bash
  psql "$DATABASE_URL" -f database/schema.sql
  ```

### Types

- TypeScript interfaces for database entities are defined in `src/types/user.ts`.
- These interfaces provide type safety when mapping database rows to application objects, ensuring that the shape of the data returned from queries is known at compile time.

---

## 3. API Routes & Endpoints

The RESTful API routes are located in `src/app/api/`. The table below summarizes all available endpoints.

| Method   | Endpoint          | File                              | Description                                              |
|----------|-------------------|-----------------------------------|----------------------------------------------------------|
| `GET`    | `/api/health`     | `src/app/api/health/route.ts`     | Health check to verify server and database are running   |
| `GET`    | `/api/users`      | `src/app/api/users/route.ts`      | Fetches a list of users from the database                |
| `POST`   | `/api/users`      | `src/app/api/users/route.ts`      | Creates a new user                                       |
| `GET`    | `/api/users/[id]` | `src/app/api/users/[id]/route.ts` | Fetches a single user by ID                              |
| `PUT`    | `/api/users/[id]` | `src/app/api/users/[id]/route.ts` | Updates a user by ID                                     |
| `DELETE` | `/api/users/[id]` | `src/app/api/users/[id]/route.ts` | Deletes a user by ID                                     |

### `GET /api/health`

- **Method:** `GET`
- **Route:** `/api/health`
- **Source file:** `src/app/api/health/route.ts`
- **Purpose:** Health check endpoint used to verify that the server is running and that the database connection is healthy. Typically used by monitoring tools, load balancers, or deployment platforms to confirm service availability.

### `GET /api/users`

- **Method:** `GET`
- **Route:** `/api/users`
- **Source file:** `src/app/api/users/route.ts`
- **Purpose:** Fetches the list of users from the database. Returns a collection of user records for display and management in the UI.

### `POST /api/users`

- **Method:** `POST`
- **Route:** `/api/users`
- **Source file:** `src/app/api/users/route.ts`
- **Purpose:** Creates a new user. Accepts the required user attributes in the request body, validates the input, persists the new record to the database, and returns the created user.

### `GET /api/users/[id]`

- **Method:** `GET`
- **Route:** `/api/users/[id]`
- **Source file:** `src/app/api/users/[id]/route.ts`
- **Purpose:** Fetches a single user by ID. The `[id]` dynamic segment is read from the URL and used to look up the corresponding record in the database.

### `PUT /api/users/[id]`

- **Method:** `PUT`
- **Route:** `/api/users/[id]`
- **Source file:** `src/app/api/users/[id]/route.ts`
- **Purpose:** Updates a user by ID. Accepts the fields to update in the request body, applies the changes to the matching record, and returns the updated user.

### `DELETE /api/users/[id]`

- **Method:** `DELETE`
- **Route:** `/api/users/[id]`
- **Source file:** `src/app/api/users/[id]/route.ts`
- **Purpose:** Deletes a user by ID. Removes the matching record from the database and returns a confirmation of the deletion.

---

## 4. Middleware & Authentication

### Middleware

- The Next.js Edge Middleware is located at `src/app/api/middleware.ts`.
- It protects API routes by intercepting incoming requests and validating authentication tokens before they reach the route handlers. Requests that fail validation are rejected before any business logic executes.
- Next.js middleware runs on the **Edge runtime**, which executes close to the user for low-latency request handling. Because of this runtime, middleware is restricted to a subset of the Node.js APIs and should remain lightweight.

### Authentication

- Authentication is implemented using `next-auth` (NextAuth.js) for session management and authentication flows.
- Password hashing is handled by `bcryptjs`.

The following environment variables are required in `.env.local`:

| Variable          | Description                                                                                  |
|-------------------|----------------------------------------------------------------------------------------------|
| `NEXTAUTH_SECRET` | Secret key used to encrypt JWT tokens and sessions.                                          |
| `NEXTAUTH_URL`    | The canonical URL of the deployed application (e.g., `http://localhost:3000` for local dev). |
| `DATABASE_URL`    | The PostgreSQL connection string used by the database client in `src/lib/db.ts`.             |

---

## 5. Data Fetching & Forms

### Data Fetching

- The frontend uses `@tanstack/react-query` for data fetching, caching, and state management.
- The React Query provider is configured in `src/components/ReactQueryProvider.tsx`, which wraps the application and provides the query client to all components.
- The API client base URL is configured in `src/lib/api.ts`.

### Forms

- Forms utilize `react-hook-form` for form state management and submission handling.
- Schema validation is powered by `zod`, with `@hookform/resolvers` bridging `react-hook-form` and `zod` so that validation schemas are applied directly to the form fields.
- Form-related utilities are located in `src/lib/form-utils.ts`.
