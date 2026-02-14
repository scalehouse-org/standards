# Coding Standards & Architecture Patterns

This document is the authoritative reference for how we write code. Every contributor — internal or external — must follow these patterns exactly. No exceptions, no "creative alternatives."

---

## Table of Contents

1. [Tech Stack (Non-Negotiable)](#tech-stack-non-negotiable)
2. [Backend Architecture](#backend-architecture)
3. [Domain Structure](#domain-structure)
4. [Handler Pattern](#handler-pattern)
5. [Service Pattern](#service-pattern)
6. [Mapper Pattern](#mapper-pattern)
7. [Entity Pattern](#entity-pattern)
8. [Response Envelope](#response-envelope)
9. [Authentication & Authorization](#authentication--authorization)
10. [TypeORM Migrations](#typeorm-migrations)
11. [OpenAPI Contract-First Workflow](#openapi-contract-first-workflow)
12. [Frontend Architecture](#frontend-architecture)
13. [Naming Conventions](#naming-conventions)
14. [Do NOT List](#do-not-list)

---

## Tech Stack (Non-Negotiable)

| Layer | Technology |
|---|---|
| Backend Runtime | Node.js 22.x, TypeScript (strict) |
| Backend Framework | Serverless Framework v3, AWS Lambda |
| ORM | TypeORM ^0.3.20 |
| Database | PostgreSQL 16 (AWS RDS) |
| Cache | Redis 7 (AWS ElastiCache) |
| Queue | AWS SQS |
| Storage | AWS S3 + CloudFront CDN |
| Auth | Firebase Auth (JWKS verification via `jose`) |
| API Spec | OpenAPI 3.0.3 (`openapi.yaml`) |
| Type Generation | `openapi-typescript` (backend + frontend) |
| Frontend Framework | Next.js (App Router), React, TypeScript (strict) |
| Frontend API Client | `openapi-fetch` with generated types |
| Styling | Tailwind CSS v4, Prettier with `prettier-plugin-tailwindcss` |
| Linting | ESLint (`next/core-web-vitals`) |
| Infrastructure | Terraform (modular), AWS |
| CI/CD | GitHub Actions |

No substitutions without explicit lead engineer approval.

---

## Backend Architecture

### Project Structure

```
backend/
├── serverless.yml                    # Main Serverless config (imports function files)
├── package.json
├── tsconfig.json
├── docker-compose.yml                # Local dev (Postgres + LocalStack + Redis)
├── serverless/
│   ├── functions/                    # One YAML per domain
│   │   ├── auth.yml
│   │   ├── health.yml
│   │   ├── {domain}.yml
│   │   └── migrations.yml
│   └── resources/                    # CloudFormation resources (SQS, etc.)
├── src/
│   ├── config/
│   │   ├── data-source.ts           # TypeORM DataSource (singleton, Lambda-optimized)
│   │   └── firebase-admin.ts        # Firebase JWKS verification
│   ├── domains/                     # One folder per business domain
│   │   └── {domain}/
│   │       ├── entities/
│   │       ├── handlers/
│   │       ├── mappers/
│   │       ├── services/
│   │       └── index.ts             # Barrel export
│   ├── migrations/                  # TypeORM migration files
│   ├── shared/
│   │   ├── middleware/
│   │   │   └── errorHandler.ts      # Custom error classes + handler
│   │   ├── entities/                # Shared base entities (if any)
│   │   └── utils/
│   │       ├── response.ts          # Response envelope helpers
│   │       ├── auth.ts              # Auth context extraction
│   │       ├── image-url.ts         # S3 key -> CloudFront URL
│   │       ├── s3.ts                # S3 client factory
│   │       └── pagination.ts        # Pagination helpers
│   └── types/
│       └── api.ts                   # AUTO-GENERATED from OpenAPI (never edit)
```

### Key Principles

- **Domain-driven structure**: Each business domain gets its own folder under `src/domains/`.
- **Separation of concerns**: Handlers parse/respond, services own logic, mappers transform, entities define schema.
- **Singleton DataSource**: One lazy-initialized TypeORM connection per Lambda container, `max: 1` connection pool for serverless.
- **No business logic in handlers**: Handlers are thin controllers.

---

## Domain Structure

Every domain follows this exact folder layout:

```
src/domains/{domain}/
├── entities/
│   └── {name}.entity.ts          # TypeORM entity
├── handlers/
│   └── index.ts                  # All Lambda handlers for this domain
├── mappers/
│   └── {name}.mapper.ts          # Entity <-> API schema transforms
├── services/
│   └── {name}.service.ts         # Business logic
└── index.ts                      # Barrel export
```

### Barrel Export Pattern

```typescript
// src/domains/{domain}/index.ts
export * from './entities/{name}.entity';
export * from './services/{name}.service';
export * from './mappers/{name}.mapper';
export * as handlers from './handlers';
```

### Adding a New Domain Checklist

1. Create the domain folder structure above.
2. Create `serverless/functions/{domain}.yml` with function definitions.
3. Import in `serverless.yml`: `- ${file(serverless/functions/{domain}.yml)}`.
4. Register the entity in `src/config/data-source.ts`.
5. Write the migration for the new table(s).
6. Update `openapi.yaml` with new endpoints and schemas.
7. Run `npm run openapi:generate` to regenerate types.

---

## Handler Pattern

Handlers live in `src/domains/{domain}/handlers/index.ts`. They are **thin controllers** — their only job is to:

1. Parse the `APIGatewayProxyEvent` (path params, query params, body, auth context).
2. Validate input (basic checks only — complex validation belongs in services).
3. Call the appropriate service method.
4. Map the result using a mapper.
5. Return a standardized response.

### Template

```typescript
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import {
  success,
  created,
  badRequest,
  notFound,
  forbidden,
  serverError,
} from '../../../shared/utils/response';
import { getAuthContext, requireUserId } from '../../../shared/utils/auth';
import { ThingService } from '../services/thing.service';
import { transformToApiThing } from '../mappers/thing.mapper';

const thingService = new ThingService();

export const getAll = async (
  event: APIGatewayProxyEvent,
): Promise<APIGatewayProxyResult> => {
  try {
    const authContext = getAuthContext(event);
    if (!authContext?.userId) {
      return badRequest('User ID is required');
    }

    const dbThings = await thingService.findByUserId(authContext.userId);
    const things = dbThings.map(transformToApiThing);
    return success(things);
  } catch (err) {
    console.error('Error fetching things:', err);
    return serverError(err instanceof Error ? err.message : 'Unknown error');
  }
};

export const getById = async (
  event: APIGatewayProxyEvent,
): Promise<APIGatewayProxyResult> => {
  try {
    const id = event.pathParameters?.id;
    if (!id) return badRequest('ID is required');

    const userId = requireUserId(event);
    const dbThing = await thingService.findById(id);
    if (!dbThing) return notFound('Thing not found');

    return success(transformToApiThing(dbThing));
  } catch (err) {
    console.error('Error fetching thing:', err);
    return serverError(err instanceof Error ? err.message : 'Unknown error');
  }
};

export const create = async (
  event: APIGatewayProxyEvent,
): Promise<APIGatewayProxyResult> => {
  try {
    const userId = requireUserId(event);
    const body = JSON.parse(event.body || '{}');

    // Basic input validation
    if (!body.name) return badRequest('Name is required');

    const dbThing = await thingService.create({ ...body, userId });
    return created(transformToApiThing(dbThing));
  } catch (err) {
    console.error('Error creating thing:', err);
    return serverError(err instanceof Error ? err.message : 'Unknown error');
  }
};
```

### Rules

- **No business logic** in handlers. If you find yourself writing conditional logic beyond input parsing, it belongs in the service.
- Always use the shared response helpers — never construct raw `{ statusCode, body, headers }` manually.
- Always extract auth context via `getAuthContext(event)` or `requireUserId(event)`.
- Always log errors with `console.error` before returning `serverError`.

---

## Service Pattern

Services live in `src/domains/{domain}/services/{name}.service.ts`. They own **all business logic**.

### Template

```typescript
import { Repository } from 'typeorm';
import { Thing } from '../entities/thing.entity';
import { getDataSource } from '../../../config/data-source';
import {
  ForbiddenError,
  NotFoundError,
} from '../../../shared/middleware/errorHandler';

// DTOs as interfaces — no classes
export interface CreateThingDto {
  name: string;
  description?: string;
  userId: string;
}

export interface UpdateThingDto {
  name?: string;
  description?: string;
}

export class ThingService {
  private async getRepository(): Promise<Repository<Thing>> {
    const dataSource = await getDataSource();
    return dataSource.getRepository(Thing);
  }

  async findByUserId(userId: string): Promise<Thing[]> {
    const repository = await this.getRepository();
    return repository.find({
      where: { userId },
      order: { createdAt: 'DESC' },
    });
  }

  async findById(id: string): Promise<Thing | null> {
    const repository = await this.getRepository();
    return repository.findOne({ where: { id } });
  }

  async create(data: CreateThingDto): Promise<Thing> {
    const repository = await this.getRepository();
    const thing = repository.create(data);
    return repository.save(thing);
  }

  async update(
    id: string,
    data: UpdateThingDto,
    userId?: string | null,
  ): Promise<Thing | null> {
    const repository = await this.getRepository();
    const thing = await repository.findOne({ where: { id } });

    if (!thing) return null;

    // Ownership check — business logic lives HERE, not in handler
    if (userId && thing.userId && thing.userId !== userId) {
      throw new ForbiddenError('Access denied');
    }

    Object.assign(thing, data);
    return repository.save(thing);
  }

  async delete(id: string, userId?: string | null): Promise<boolean> {
    const repository = await this.getRepository();
    const thing = await repository.findOne({ where: { id } });

    if (!thing) return false;

    if (userId && thing.userId && thing.userId !== userId) {
      throw new ForbiddenError('Access denied');
    }

    await repository.remove(thing);
    return true;
  }
}
```

### Rules

- Services are **class-based** with a private `getRepository()` method.
- DTOs are defined as **interfaces** (not classes) in the same service file.
- Throw custom error classes (`ForbiddenError`, `NotFoundError`) — handlers catch and map to responses.
- Never import response helpers in services. Services return entities or throw errors.

---

## Mapper Pattern

Mappers live in `src/domains/{domain}/mappers/{name}.mapper.ts`. They are **pure functions** that transform between TypeORM entities and OpenAPI schemas.

### Template

```typescript
import type { components } from '../../../types/api';
import { resolveImageUrl } from '../../../shared/utils/image-url';

type ApiThing = components['schemas']['Thing'];

export function transformToApiThing(dbThing: any): ApiThing {
  return {
    id: dbThing.id,
    name: dbThing.name,
    description: dbThing.description || null,
    imageUrl: dbThing.image ? resolveImageUrl(dbThing.image) : null,
    createdAt:
      dbThing.createdAt instanceof Date
        ? dbThing.createdAt.toISOString()
        : dbThing.createdAt,
    updatedAt:
      dbThing.updatedAt instanceof Date
        ? dbThing.updatedAt.toISOString()
        : dbThing.updatedAt,
  };
}
```

### Rules

- Export **pure functions** only — no classes, no side effects.
- Handle type coercion: Date -> ISO string, JSONB parsing, null safety.
- Use `resolveImageUrl()` for any S3 keys that need CloudFront URL resolution.
- Handlers and services import from mappers — never duplicate transform logic.

---

## Entity Pattern

Entities live in `src/domains/{domain}/entities/{name}.entity.ts`.

### Template

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

@Entity('things')
export class Thing {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255 })
  name: string;

  @Column({ type: 'text', nullable: true })
  description: string | null;

  @Column({ type: 'jsonb', default: [] })
  tags: string[];

  @Column({ type: 'varchar', length: 500, nullable: true })
  image: string | null;

  @Column({ type: 'uuid', nullable: true, name: 'user_id' })
  userId: string | null;

  @Column({ type: 'boolean', name: 'is_active', default: true })
  isActive: boolean;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

### Rules

- **Table names**: snake_case, plural (`things`, `purchase_orders`, `user_roles`).
- **Column names**: snake_case in DB, camelCase in TypeScript. Use the `name` option on decorators to map.
- **Primary keys**: UUID via `@PrimaryGeneratedColumn('uuid')` for most entities. Firebase UID string via `@PrimaryColumn({ type: 'varchar' })` for user entities.
- **Timestamps**: Always include `@CreateDateColumn` and `@UpdateDateColumn`.
- **Soft deletes**: Use a `deleted_at` column where applicable (not TypeORM's `@DeleteDateColumn` — use explicit column).
- **JSONB**: Use for arrays and flexible object columns.

---

## Response Envelope

All API responses use a consistent envelope shape enforced by shared helpers in `src/shared/utils/response.ts`.

### Success Responses

```json
{
  "data": { ... }
}
```

With pagination:

```json
{
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

### Error Responses

```json
{
  "error": "Human-readable error message",
  "details": { ... }
}
```

### Shared Helpers

```typescript
// src/shared/utils/response.ts

export const defaultHeaders = {
  'Content-Type': 'application/json',
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
};

export function success<T>(data: T, statusCode = 200) {
  return { statusCode, headers: defaultHeaders, body: JSON.stringify({ data }) };
}

export function created<T>(data: T) {
  return success(data, 201);
}

export function noContent() {
  return { statusCode: 204, headers: defaultHeaders, body: '' };
}

export function badRequest(message: string, details?: unknown) {
  return { statusCode: 400, headers: defaultHeaders, body: JSON.stringify({ error: message, details }) };
}

export function unauthorized(message = 'Unauthorized') {
  return { statusCode: 401, headers: defaultHeaders, body: JSON.stringify({ error: message }) };
}

export function forbidden(message = 'Forbidden') {
  return { statusCode: 403, headers: defaultHeaders, body: JSON.stringify({ error: message }) };
}

export function notFound(message = 'Not found') {
  return { statusCode: 404, headers: defaultHeaders, body: JSON.stringify({ error: message }) };
}

export function serverError(message = 'Internal server error') {
  return { statusCode: 500, headers: defaultHeaders, body: JSON.stringify({ error: message }) };
}
```

Always use these helpers. Never construct raw API Gateway responses manually.

---

## Authentication & Authorization

### Flow

1. User signs in via Firebase (Google, email link, etc.).
2. Frontend stores Firebase ID token in `__session` httpOnly cookie.
3. API requests include `Authorization: Bearer <token>` header.
4. **Lambda Authorizer** validates the token using Firebase JWKS (public keys from Google, no service account needed). Uses the `jose` library.
5. On success, authorizer generates an IAM policy with user context (`userId`, `email`, `roles`).
6. Handlers extract context via `getAuthContext(event)`.

### Auth Utilities (`src/shared/utils/auth.ts`)

```typescript
interface AuthContext {
  userId: string;
  email: string;
  roles: string[];
}

// Extract auth context from Lambda Authorizer
export function getAuthContext(event: APIGatewayProxyEvent): AuthContext | null { ... }

// Get user ID or throw
export function requireUserId(event: APIGatewayProxyEvent): string { ... }

// Role checks
export function hasRole(event: APIGatewayProxyEvent, role: string): boolean { ... }
export function hasAnyRole(event: APIGatewayProxyEvent, roles: string[]): boolean { ... }
```

### Error Classes (`src/shared/middleware/errorHandler.ts`)

```typescript
export class AppError extends Error {
  statusCode: number;
  isOperational: boolean;
}

export class ValidationError extends AppError { /* 400 */ }
export class NotFoundError extends AppError { /* 404 */ }
export class UnauthorizedError extends AppError { /* 401 */ }
export class ForbiddenError extends AppError { /* 403 */ }
```

---

## TypeORM Migrations

Migrations live in `src/migrations/`. They are the **only** way database schema changes are made. Never use `synchronize: true`.

### Naming Convention

```
{Timestamp}-{DescriptiveName}.ts
```

Example: `1706000001000-CreateThingsTable.ts`

The class name matches: `CreateThingsTable1706000001000`.

### Generic Migration Template

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CreateThingsTable1706000001000 implements MigrationInterface {
  name = 'CreateThingsTable1706000001000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE "things" (
        "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        "name" VARCHAR(255) NOT NULL,
        "slug" VARCHAR(255) NOT NULL UNIQUE,
        "description" TEXT,
        "status" VARCHAR(50) NOT NULL DEFAULT 'active',
        "metadata" JSONB DEFAULT '{}',
        "image" VARCHAR(500),
        "user_id" UUID,
        "is_active" BOOLEAN NOT NULL DEFAULT true,
        "created_at" TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
        "updated_at" TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
      )
    `);

    -- Indexes for common query patterns
    await queryRunner.query(`
      CREATE INDEX "IDX_things_user_id" ON "things" ("user_id")
    `);
    await queryRunner.query(`
      CREATE INDEX "IDX_things_status" ON "things" ("status")
    `);
    await queryRunner.query(`
      CREATE UNIQUE INDEX "IDX_things_slug" ON "things" ("slug")
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    -- Drop indexes FIRST, then table (reverse order of up)
    await queryRunner.query(`DROP INDEX IF EXISTS "IDX_things_slug"`);
    await queryRunner.query(`DROP INDEX IF EXISTS "IDX_things_status"`);
    await queryRunner.query(`DROP INDEX IF EXISTS "IDX_things_user_id"`);
    await queryRunner.query(`DROP TABLE IF EXISTS "things"`);
  }
}
```

### Junction Table Migration Template

For many-to-many relationships:

```typescript
public async up(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`
    CREATE TABLE "thing_categories" (
      "thing_id" UUID NOT NULL REFERENCES "things"("id") ON DELETE CASCADE,
      "category_id" UUID NOT NULL REFERENCES "categories"("id") ON DELETE CASCADE,
      PRIMARY KEY ("thing_id", "category_id")
    )
  `);

  await queryRunner.query(`
    CREATE INDEX "IDX_thing_categories_thing_id" ON "thing_categories" ("thing_id")
  `);
  await queryRunner.query(`
    CREATE INDEX "IDX_thing_categories_category_id" ON "thing_categories" ("category_id")
  `);
}

public async down(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`DROP INDEX IF EXISTS "IDX_thing_categories_category_id"`);
  await queryRunner.query(`DROP INDEX IF EXISTS "IDX_thing_categories_thing_id"`);
  await queryRunner.query(`DROP TABLE IF EXISTS "thing_categories"`);
}
```

### Column Type Quick Reference

| Use Case | SQL Type | TypeORM Decorator |
|---|---|---|
| UUID primary key | `UUID PRIMARY KEY DEFAULT gen_random_uuid()` | `@PrimaryGeneratedColumn('uuid')` |
| String PK (e.g. Firebase UID) | `VARCHAR(255) PRIMARY KEY` | `@PrimaryColumn({ type: 'varchar', length: 255 })` |
| Required string | `VARCHAR(255) NOT NULL` | `@Column({ type: 'varchar', length: 255 })` |
| Optional string | `VARCHAR(255)` | `@Column({ type: 'varchar', length: 255, nullable: true })` |
| Unique string | `VARCHAR(255) NOT NULL UNIQUE` | `@Column({ type: 'varchar', unique: true })` |
| Long text | `TEXT` | `@Column({ type: 'text', nullable: true })` |
| Boolean with default | `BOOLEAN NOT NULL DEFAULT true` | `@Column({ type: 'boolean', default: true })` |
| JSON/Array | `JSONB DEFAULT '[]'` | `@Column({ type: 'jsonb', default: [] })` |
| Integer | `INTEGER NOT NULL DEFAULT 0` | `@Column({ type: 'integer', default: 0 })` |
| Decimal/Money | `DECIMAL(10,2)` | `@Column({ type: 'decimal', precision: 10, scale: 2 })` |
| Timestamp (auto) | `TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP` | `@CreateDateColumn({ name: 'created_at' })` |
| Foreign key | `UUID REFERENCES "other"("id")` | `@ManyToOne(() => Other)` + `@JoinColumn({ name: 'other_id' })` |
| Enum-like | `VARCHAR(50) NOT NULL DEFAULT 'pending'` | `@Column({ type: 'varchar', length: 50, default: 'pending' })` |

### Index Naming Convention

```
IDX_{table}_{column}            -- standard index
IDX_{table}_{column}_unique     -- unique index (or use CREATE UNIQUE INDEX)
```

### Migration Commands

```bash
# Generate migration from entity changes (diff-based)
npm run migration:generate -- src/migrations/MigrationName

# Create empty migration (for manual SQL)
npm run migration:create -- src/migrations/MigrationName

# Run all pending migrations
npm run migration:run

# Revert the last migration
npm run migration:revert

# Show migration status
npm run migration:show
```

### Migration Rules

- **Always write `down()` methods** — no exceptions.
- **Reverse order in `down()`** — drop indexes before tables, drop dependent tables before parent tables.
- **Never use `synchronize: true`** in production or staging.
- **Test migrations locally** before pushing: run `migration:run` then `migration:revert` to verify both directions.
- **Use raw SQL** via `queryRunner.query()` — not the schema builder.
- **Timestamp-based naming** — never manually number migrations.

---

## OpenAPI Contract-First Workflow

The `openapi.yaml` file is the **single source of truth** for all API communication. Both backend and frontend generate their types from it.

### Workflow

1. **Design the API first**: Edit `openapi.yaml` with new endpoints, request/response schemas.
2. **Generate types**: Run `npm run openapi:generate` in both backend and frontend.
3. **Implement backend**: Write handlers/services/mappers using the generated types.
4. **Implement frontend**: Use `openapi-fetch` with the generated `paths` types.
5. **Never hand-edit `api-types.ts`** — it is auto-generated and will be overwritten.

### Using Generated Types

Backend:
```typescript
import type { components } from '../../../types/api';

type Thing = components['schemas']['Thing'];
type CreateThingRequest = components['schemas']['CreateThingRequest'];
```

Frontend:
```typescript
import type { components } from '@/lib/api-types';

export type Thing = components['schemas']['Thing'];
```

### Type Generation Command

```bash
# Backend
cd backend && npm run openapi:generate

# Frontend
cd frontend && npm run openapi:generate
```

---

## Frontend Architecture

### Project Structure

```
frontend/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── (admin)/                  # Route group: authenticated pages
│   │   │   ├── layout.tsx            # Sidebar + header layout
│   │   │   ├── page.tsx              # Dashboard
│   │   │   └── {domain}/            # Domain pages
│   │   ├── (full-width-pages)/       # Route group: auth, public pages
│   │   │   ├── (auth)/
│   │   │   │   ├── signin/
│   │   │   │   └── signup/
│   │   │   └── layout.tsx
│   │   ├── api/                      # Next.js API routes
│   │   │   ├── auth/session/         # Session cookie management
│   │   │   └── upload/presigned-url/ # S3 upload proxy
│   │   └── layout.tsx                # Root layout (providers)
│   ├── components/
│   │   ├── ui/                       # Generic UI (button, modal, table, etc.)
│   │   ├── common/                   # Shared business components
│   │   ├── {domain}/                 # Domain-specific components
│   │   └── form/                     # Form elements
│   ├── context/
│   │   ├── auth-context.tsx          # Firebase Auth provider + hook
│   │   ├── theme-context.tsx         # Theme provider
│   │   └── sidebar-context.tsx       # Sidebar state
│   ├── hooks/                        # Custom React hooks
│   ├── icons/                        # SVG icon components
│   ├── layout/                       # Layout components (header, sidebar)
│   ├── lib/
│   │   ├── api-client.ts            # openapi-fetch client + auth helper
│   │   ├── api-types.ts             # AUTO-GENERATED (never edit)
│   │   ├── types.ts                 # Re-exported OpenAPI types + form types
│   │   ├── config.ts                # Environment config
│   │   ├── firebase.ts              # Firebase SDK init (client-only)
│   │   ├── auth-utils.ts            # Server-side auth helpers
│   │   └── utils.ts                 # General utilities (cn(), etc.)
│   └── types/                        # Additional TypeScript types
├── .eslintrc.json
├── next.config.ts
├── prettier.config.js
├── tailwind.config.ts (or globals.css @theme for v4)
├── tsconfig.json
└── package.json
```

### API Client Pattern

```typescript
// src/lib/api-client.ts
import createClient from 'openapi-fetch';
import type { paths } from './api-types';

const BACKEND_URL = process.env.NEXT_PUBLIC_API_URL;

export const apiClient = createClient<paths>({ baseUrl: BACKEND_URL });

export function createAuthenticatedClient(token: string) {
  return createClient<paths>({
    baseUrl: BACKEND_URL,
    headers: { Authorization: `Bearer ${token}` },
  });
}
```

Usage is fully type-safe — request params and response shapes are inferred from the OpenAPI spec.

### Auth Context Pattern

```typescript
// src/context/auth-context.tsx
'use client';

export function AuthProvider({ children }: { children: React.ReactNode }) {
  // Firebase onAuthStateChanged listener
  // Sets __session cookie via /api/auth/session on login
  // Clears cookie on logout
  // Exposes: user, loading, signIn, signOut
}

export const useAuth = () => useContext(AuthContext);
```

### Middleware Route Protection

```typescript
// src/middleware.ts
// 1. Public routes (signin, signup, api/auth) — pass through
// 2. Protected API routes — check __session cookie, return 401 if missing
// 3. Protected pages — redirect to /signin if no token
```

### Styling Rules

- **Tailwind CSS v4** with CSS-based `@theme` configuration in `globals.css`.
- **Prettier** with `prettier-plugin-tailwindcss` for automatic class sorting.
- **ESLint** with `next/core-web-vitals` rules.
- **SVGs** imported as React components via `@svgr/webpack`.
- **Path aliases**: `@/*` maps to `./src/*`.

### Frontend Config Files

**`.eslintrc.json`**:
```json
{
  "extends": "next/core-web-vitals"
}
```

**`prettier.config.js`**:
```javascript
module.exports = {
  plugins: ['prettier-plugin-tailwindcss'],
};
```

**`tsconfig.json`** (relevant paths):
```json
{
  "compilerOptions": {
    "strict": true,
    "paths": { "@/*": ["./src/*"] }
  }
}
```

---

## Naming Conventions

| Context | Convention | Example |
|---|---|---|
| Database table names | snake_case, plural | `purchase_orders`, `user_roles` |
| Database column names | snake_case | `created_at`, `user_id`, `is_active` |
| TypeScript properties | camelCase | `createdAt`, `userId`, `isActive` |
| Entity files | `{name}.entity.ts` | `thing.entity.ts` |
| Service files | `{name}.service.ts` | `thing.service.ts` |
| Mapper files | `{name}.mapper.ts` | `thing.mapper.ts` |
| Handler files | `index.ts` (per domain) | `handlers/index.ts` |
| Serverless function files | `{domain}.yml` | `properties.yml` |
| Migration files | `{Timestamp}-{Name}.ts` | `1706000001000-CreateThingsTable.ts` |
| Frontend components | PascalCase | `ThingCard.tsx`, `ThingList.tsx` |
| Frontend hooks | camelCase with `use` prefix | `useAuth.ts`, `useThing.ts` |
| Environment variables | UPPER_SNAKE_CASE | `DB_HOST`, `NEXT_PUBLIC_API_URL` |

---

## Do NOT List

These are hard rules. Violations will be rejected in code review.

| Do NOT | Why |
|---|---|
| Use `synchronize: true` in production | Destroys data. Migrations are the only way. |
| Store secrets in code or `serverless.yml` | Use AWS Secrets Manager or env vars. |
| Manually edit `src/types/api.ts` or `api-types.ts` | Auto-generated from OpenAPI. Will be overwritten. |
| Skip writing migration `down()` methods | Rollbacks must work. No exceptions. |
| Skip updating `openapi.yaml` when changing API contracts | The spec is the source of truth. Code follows spec. |
| Put business logic in handlers | Handlers parse and respond. Services own logic. |
| Return database entities directly from handlers | Always transform through mappers to OpenAPI schemas. |
| Create raw `{ statusCode, body }` responses | Use the shared response helpers. |
| Construct fetch calls manually in frontend | Use `openapi-fetch` with generated types. |
| Commit without running linter and type checks | CI will catch it. Save everyone's time. |
| Deploy migrations without testing locally | Run `migration:run` + `migration:revert` locally first. |
| Use `any` types without justification | TypeScript strict mode is on for a reason. |
