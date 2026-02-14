# Scalehose Standards

A production-ready, full-stack serverless application built on AWS with a contract-first API design. The project follows domain-driven architecture patterns with comprehensive infrastructure-as-code and CI/CD pipelines.

---

## Tech Stack

| Layer              | Technology                                                             |
| ------------------ | ---------------------------------------------------------------------- |
| **Backend**        | Node.js 22.x, TypeScript (strict), Serverless Framework v3, AWS Lambda |
| **Database**       | PostgreSQL 16 (AWS RDS), TypeORM ^0.3.20                               |
| **Cache**          | Redis 7 (AWS ElastiCache)                                              |
| **Queue**          | AWS SQS                                                                |
| **Storage**        | AWS S3 + CloudFront CDN                                                |
| **Auth**           | Firebase Auth (JWT via Lambda Authorizer)                              |
| **Frontend**       | Next.js (App Router), React, Tailwind CSS v4                           |
| **API Client**     | `openapi-fetch` with auto-generated types                              |
| **API Spec**       | OpenAPI 3.0.3 (contract-first)                                         |
| **Infrastructure** | Terraform (modular), AWS                                               |
| **CI/CD**          | GitHub Actions (OIDC auth)                                             |

---

## Project Structure

```
scalehose-standards/
├── docs/                          # Architecture & operations documentation
│   ├── coding-standards.md        # Code patterns and conventions
│   ├── infrastructure-standards.md # Terraform and AWS standards
│   └── deployment-cheatsheet.md   # Quick reference for operations
├── backend/                       # Serverless backend
│   ├── serverless.yml             # Lambda function definitions
│   ├── docker-compose.yml         # Local Postgres, Redis, LocalStack
│   └── src/
│       ├── config/                # DB, cache, and AWS client setup
│       ├── domains/               # Domain modules (handler → service → mapper → entity)
│       ├── migrations/            # TypeORM migrations
│       └── shared/                # Response helpers, auth middleware, utilities
├── frontend/                      # Next.js application
│   └── src/
│       ├── app/                   # App Router pages and layouts
│       ├── components/            # Reusable UI components
│       └── lib/                   # API client, auth, and utilities
└── infra/                         # Infrastructure as Code
    ├── openapi.yaml               # API contract (single source of truth)
    └── terraform/
        ├── modules/               # Reusable Terraform modules
        └── environments/          # Per-environment variable files
```

---

## Architecture

### Contract-First API Design

`openapi.yaml` is the single source of truth. Types are auto-generated for both backend and frontend — never edit generated type files manually.

### Domain-Driven Backend

Each business domain is a self-contained module:

- **Handler** — Thin controller: parse, validate, delegate, respond
- **Service** — Business logic and data access
- **Mapper** — Pure transform functions (entity <-> API schema)
- **Entity** — TypeORM schema definition

### Response Envelope

All API responses follow a consistent shape:

```typescript
// Success
{ "data": T }
{ "data": T[], "pagination": { "page": 1, "limit": 20, "total": 150 } }

// Error
{ "error": "message", "details?": unknown }
```

---

## Prerequisites

- Node.js 22.x
- Docker and Docker Compose
- AWS CLI v2
- Terraform

---

## Local Development

### First-Time Setup

```bash
# 1. Clone the repository
git clone <repo-url>
cd scalehose-standards

# 2. Start local infrastructure (Postgres, Redis, LocalStack)
cd backend
docker-compose up -d

# 3. Install backend dependencies
npm ci

# 4. Configure environment
cp .env.example .env
# Edit .env with local values (see Environment Variables below)

# 5. Generate API types from OpenAPI spec
npm run openapi:generate

# 6. Run database migrations
npm run migration:run

# 7. Start backend (serverless-offline on port 3001)
npm run dev

# 8. In a new terminal — set up frontend
cd frontend
npm ci
cp .env.example .env.local
npm run openapi:generate

# 9. Start frontend (Next.js on port 3000)
npm run dev
```

### Daily Startup

```bash
# Terminal 1 — Backend
cd backend && docker-compose up -d && npm run dev

# Terminal 2 — Frontend
cd frontend && npm run dev
```

---

## Environment Variables

### Backend (`.env`)

| Variable              | Local Value              | Description             |
| --------------------- | ------------------------ | ----------------------- |
| `DB_HOST`             | `localhost`              | Database host           |
| `DB_PORT`             | `5432`                   | Database port           |
| `DB_USERNAME`         | `postgres`               | Database user           |
| `DB_PASSWORD`         | `postgres`               | Database password       |
| `DB_NAME`             | `scalehose_standards_db` | Database name           |
| `NODE_ENV`            | `local`                  | Environment             |
| `AWS_REGION`          | `us-east-1`              | AWS region              |
| `REDIS_HOST`          | `localhost`              | Redis host              |
| `REDIS_PORT`          | `6379`                   | Redis port              |
| `S3_ENDPOINT`         | `http://localhost:4566`  | LocalStack S3 endpoint  |
| `SQS_ENDPOINT`        | `http://localhost:4566`  | LocalStack SQS endpoint |
| `FIREBASE_PROJECT_ID` | —                        | Firebase project ID     |

### Frontend (`.env.local`)

| Variable                           | Local Value                 | Description          |
| ---------------------------------- | --------------------------- | -------------------- |
| `NEXT_PUBLIC_API_URL`              | `http://localhost:3001/dev` | Backend API URL      |
| `NEXT_PUBLIC_FIREBASE_API_KEY`     | —                           | Firebase API key     |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | —                           | Firebase auth domain |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID`  | —                           | Firebase project ID  |

> Production secrets are stored in AWS Secrets Manager and SSM Parameter Store — never in code or config files.

---

## Key Workflows

### OpenAPI Type Generation

```bash
# After editing infra/openapi.yaml
cd backend && npm run openapi:generate
cd frontend && npm run openapi:generate
```

### Database Migrations

```bash
cd backend

npm run migration:show              # Show migration status
npm run migration:run               # Run pending migrations
npm run migration:revert            # Revert last migration
npm run migration:generate -- src/migrations/MigrationName   # Generate from entity changes
npm run migration:create -- src/migrations/MigrationName     # Create empty migration
```

### Terraform

```bash
cd infra/terraform

./scripts/init.sh dev               # Initialize for environment
./scripts/validate.sh               # Validate configuration
./scripts/plan.sh dev               # Plan changes
./scripts/apply.sh dev              # Apply changes
```

### Deployment

```bash
# Backend
cd backend
npm run deploy                      # Deploy to dev
npm run deploy:prod                 # Deploy to prod
npx serverless deploy function --function getThings --stage dev  # Single function

# Frontend — deployed via CI/CD pipeline
```

---

## CI/CD

| Workflow                          | Trigger           | What It Does                                       |
| --------------------------------- | ----------------- | -------------------------------------------------- |
| **CI** (`ci.yml`)                 | Push/PR to `main` | Typecheck, lint, test (backend); build (frontend)  |
| **Deploy Dev**                    | Push to `main`    | Terraform apply, Serverless deploy, run migrations |
| **Deploy Prod**                   | Manual dispatch   | Same as dev with production config                 |
| **DB Migrate** (`db-migrate.yml`) | Manual dispatch   | Run, show, or revert migrations per environment    |

All pipelines authenticate via OIDC — no long-lived AWS credentials in GitHub.

---

## Remote Database Access

Connect to RDS through an SSM tunnel (no public endpoint, no SSH):

```bash
# Get connection details
cd infra/terraform
terraform output ssm_tunnel_instance_id
terraform output rds_instance_endpoint

# Start tunnel
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["<rds-endpoint>"],
    "portNumber": ["5432"],
    "localPortNumber": ["5432"]
  }'

# Connect
psql -h localhost -p 5432 -U <db_username> -d <db_name>
```

---

## Documentation

| Document                                                               | Description                                                 |
| ---------------------------------------------------------------------- | ----------------------------------------------------------- |
| [`docs/coding-standards.md`](docs/coding-standards.md)                 | Architecture patterns, code conventions, and do/don't rules |
| [`docs/infrastructure-standards.md`](docs/infrastructure-standards.md) | Terraform modules, AWS stack, and security standards        |
| [`docs/deployment-cheatsheet.md`](docs/deployment-cheatsheet.md)       | Quick reference for daily operations and commands           |

---

## Rules

- **Migrations only** — never `synchronize: true` in production
- **Contract-first** — update `openapi.yaml` before changing API contracts; never edit generated types
- **Handlers are thin** — business logic belongs in services
- **Always use mappers** — never return entities directly from handlers
- **No secrets in code** — use AWS Secrets Manager / SSM Parameter Store
- **Scoped IAM** — no wildcard `*` resource policies
