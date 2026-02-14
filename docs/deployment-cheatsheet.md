# Deployment & Operations Cheat Sheet

Quick-reference for every command and workflow you need day-to-day. Bookmark this.

---

## Table of Contents

1. [Local Development Setup](#local-development-setup)
2. [OpenAPI Workflow](#openapi-workflow)
3. [Database Operations](#database-operations)
4. [Terraform Operations](#terraform-operations)
5. [Backend Deployment (Serverless)](#backend-deployment-serverless)
6. [Frontend Deployment](#frontend-deployment)
7. [SSM Tunnel (Database Access)](#ssm-tunnel-database-access)
8. [Full Deployment Flow](#full-deployment-flow)
9. [Rollback Procedures](#rollback-procedures)
10. [Debugging](#debugging)
11. [Useful AWS CLI Commands](#useful-aws-cli-commands)

---

## Local Development Setup

### First-Time Setup

```bash
# 1. Clone the repo
git clone <repo-url>
cd <project>

# 2. Start local infrastructure
cd backend
docker-compose up -d

# 3. Install backend dependencies
npm ci

# 4. Copy environment file
cp .env.example .env
# Edit .env with local values (see infrastructure-standards.md)

# 5. Generate API types from OpenAPI spec
npm run openapi:generate

# 6. Run database migrations
npm run migration:run

# 7. Start backend (serverless-offline, port 3001)
npm run dev

# 8. In a separate terminal — install frontend dependencies
cd frontend
npm ci

# 9. Copy frontend environment file
cp .env.example .env.local
# Edit with local values

# 10. Generate frontend API types
npm run openapi:generate

# 11. Start frontend (Next.js, port 3000)
npm run dev
```

### Daily Startup

```bash
# Start infra (if stopped)
cd backend && docker-compose up -d

# Start backend
npm run dev

# Start frontend (separate terminal)
cd frontend && npm run dev
```

### Stopping

```bash
# Stop local infra
cd backend && docker-compose down

# Stop with data cleanup
cd backend && docker-compose down -v
```

---

## OpenAPI Workflow

The OpenAPI spec is the source of truth. Every API change starts here.

### Edit -> Generate -> Implement -> Test

```bash
# 1. Edit the spec
# Open infra/openapi.yaml (or backend/openapi.yaml depending on project)
# Add/modify endpoints and schemas

# 2. Generate backend types
cd backend
npm run openapi:generate
# Creates/updates src/types/api.ts

# 3. Generate frontend types
cd frontend
npm run openapi:generate
# Creates/updates src/lib/api-types.ts

# 4. Implement changes in backend handlers/services/mappers

# 5. Update frontend to use new endpoints/types

# 6. Test locally
```

### Type Generation Commands

```bash
# Backend
cd backend && npm run openapi:generate

# Frontend
cd frontend && npm run openapi:generate
```

### Rule: Never edit `api-types.ts` or `src/types/api.ts` manually.

---

## Database Operations

### Migrations

```bash
# Show migration status (which have run, which are pending)
npm run migration:show

# Run all pending migrations
npm run migration:run

# Revert the last migration
npm run migration:revert

# Generate a migration from entity changes (diff-based)
npm run migration:generate -- src/migrations/DescriptiveName

# Create an empty migration (for manual SQL)
npm run migration:create -- src/migrations/DescriptiveName
```

### Migration Workflow

```bash
# 1. Write/modify entity
# 2. Generate or create migration
npm run migration:generate -- src/migrations/AddStatusToThings

# 3. Review the generated migration file
# 4. Verify down() method is correct

# 5. Test forward
npm run migration:run

# 6. Test backward
npm run migration:revert

# 7. Run forward again (leaves DB in correct state)
npm run migration:run

# 8. Commit
```

### Direct Database Access (Local)

```bash
# Connect to local Postgres
psql -h localhost -p 5432 -U postgres -d {project_db}

# Or using Docker
docker exec -it <postgres-container> psql -U postgres -d {project_db}
```

### Direct Database Access (Remote — via SSM Tunnel)

See [SSM Tunnel](#ssm-tunnel-database-access) section below.

---

## Terraform Operations

All Terraform commands are run from `infra/terraform/`.

### Using Scripts (Recommended)

```bash
cd infra/terraform

# First-time: create state bucket
./scripts/setup-state-bucket.sh

# Initialize for an environment
./scripts/init.sh dev       # or: ./scripts/init.sh prod

# Validate configuration
./scripts/validate.sh

# Plan changes
./scripts/plan.sh dev       # or: ./scripts/plan.sh prod

# Apply changes
./scripts/apply.sh dev      # or: ./scripts/apply.sh prod
```

### Manual Commands (If Scripts Don't Exist Yet)

```bash
cd infra/terraform

# Initialize
terraform init -backend-config=backend-dev.hcl

# Plan
terraform plan -var-file=environments/dev.tfvars -out=tfplan

# Apply
terraform apply tfplan

# Destroy (CAREFUL — dev only)
terraform destroy -var-file=environments/dev.tfvars
```

### Viewing Outputs

```bash
# Show all outputs
terraform output

# Show specific output
terraform output rds_instance_endpoint
terraform output files_bucket_name
terraform output cloudfront_url
terraform output ssm_tunnel_instance_id
```

### State Commands

```bash
# List all resources in state
terraform state list

# Show a specific resource
terraform state show module.rds.aws_db_instance.main

# Import an existing resource (rare)
terraform import module.networking.aws_vpc.main vpc-abc123
```

---

## Backend Deployment (Serverless)

### Deploy to Dev

```bash
cd backend
npm run deploy
# or explicitly:
npx serverless deploy --stage dev
```

### Deploy to Prod

```bash
cd backend
npm run deploy:prod
# or explicitly:
npx serverless deploy --stage prod
```

### Deploy a Single Function (Fast)

```bash
npx serverless deploy function --function getThings --stage dev
```

### View Logs

```bash
# Tail logs for a specific function
npx serverless logs --function getThings --stage dev --tail

# Last 30 minutes
npx serverless logs --function getThings --stage dev --startTime 30m
```

### Remove Deployment

```bash
npx serverless remove --stage dev
```

### Invoke Function Directly

```bash
# Invoke with no payload
npx serverless invoke --function healthCheck --stage dev

# Invoke with JSON payload
npx serverless invoke --function createThing --stage dev --data '{"body": "{\"name\":\"test\"}"}'
```

---

## Frontend Deployment

### Build

```bash
cd frontend

# Production build
npm run build

# Check for type errors
npm run typecheck

# Lint
npm run lint
```

### Deploy

Depends on hosting (Amplify, Vercel, or S3+CloudFront). CI/CD handles production deploys — see the GitHub Actions workflow.

```bash
# If using Vercel
vercel --prod

# If using Amplify — triggered by CI/CD pipeline, not manual
```

---

## SSM Tunnel (Database Access)

Connect to RDS from your local machine via AWS Systems Manager. No SSH keys, no public database endpoint.

### Prerequisites

```bash
# Install AWS CLI v2
# Install Session Manager plugin
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html
```

### Connect

```bash
# Using the script (if available)
cd infra/terraform
./scripts/ssh-tunnel.sh dev

# Or manually:
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["<rds-endpoint>"],
    "portNumber": ["5432"],
    "localPortNumber": ["5432"]
  }' \
  --region us-east-1
```

### Get Instance ID and RDS Endpoint

```bash
cd infra/terraform

# Get SSM tunnel instance ID
terraform output ssm_tunnel_instance_id

# Get RDS endpoint
terraform output rds_instance_endpoint

# Get connection command (if output exists)
terraform output ssm_tunnel_connection_command
```

### Connect with psql Through Tunnel

While the SSM tunnel is running in one terminal:

```bash
# In another terminal
psql -h localhost -p 5432 -U <db_username> -d <db_name>

# Get credentials from Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id {project}-{env}-db-credentials \
  --query SecretString --output text | jq .
```

---

## Full Deployment Flow

This is what CI/CD does on push to `main`. Understand it even if you never run it manually.

```
1. CI Checks (parallel)
   ├── Backend: npm ci -> openapi:generate -> typecheck -> lint -> test
   └── Frontend: npm ci -> openapi:generate -> build

2. Terraform
   ├── terraform fmt -check
   ├── terraform validate
   ├── terraform plan -var-file=environments/{env}.tfvars
   └── terraform apply (push to main only)

3. Backend Deploy
   ├── Retrieve DB credentials from Secrets Manager
   ├── Set VPC config from Terraform outputs
   └── serverless deploy --stage {env}

4. Database Migrations
   └── Invoke migration Lambda (action: run)

5. Frontend Deploy
   └── Build and deploy to hosting platform

6. Summary
   └── Output deployment details (URLs, versions)
```

---

## Rollback Procedures

### Backend Rollback (Serverless)

```bash
# Roll back to previous deployment
npx serverless rollback --stage dev --timestamp <timestamp>

# Find available timestamps
npx serverless deploy list --stage dev
```

### Database Rollback

```bash
# Revert last migration (remote via Lambda)
# Use the db-migrate.yml GitHub Action with action: revert

# Or locally via SSM tunnel:
# 1. Start SSM tunnel
# 2. Set DB_HOST=localhost in .env
# 3. npm run migration:revert
```

### Terraform Rollback

Terraform doesn't have a built-in rollback. Instead:

```bash
# 1. Revert the offending commit
git revert <commit-sha>

# 2. Push to main — CI/CD will plan and apply the reverted state
git push origin main

# Or manually:
terraform plan -var-file=environments/{env}.tfvars
terraform apply
```

### Frontend Rollback

Depends on hosting platform:

```bash
# Vercel: roll back to previous deployment in Vercel dashboard

# Amplify: redeploy previous build in Amplify console

# Or revert the commit and push
git revert <commit-sha>
git push origin main
```

---

## Debugging

### CloudWatch Logs

```bash
# View Lambda logs (via Serverless)
npx serverless logs --function getThings --stage dev --tail

# Via AWS CLI
aws logs tail /aws/lambda/{project}-backend-{env}-getThings --follow
```

### LocalStack (Local S3/SQS)

```bash
# List S3 buckets
aws --endpoint-url=http://localhost:4566 s3 ls

# List objects in bucket
aws --endpoint-url=http://localhost:4566 s3 ls s3://{project}-local-files/

# Create local bucket (if init script doesn't exist)
aws --endpoint-url=http://localhost:4566 s3 mb s3://{project}-local-files

# List SQS queues
aws --endpoint-url=http://localhost:4566 sqs list-queues

# Send test message to queue
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://localhost:4566/000000000000/{queue-name} \
  --message-body '{"test": true}'
```

### API Gateway (Local)

```bash
# serverless-offline runs on http://localhost:3001
# Test endpoints:
curl http://localhost:3001/dev/health
curl -H "Authorization: Bearer <token>" http://localhost:3001/dev/things
```

---

## Useful AWS CLI Commands

### Secrets Manager

```bash
# Get a secret value
aws secretsmanager get-secret-value \
  --secret-id {project}-{env}-db-credentials \
  --query SecretString --output text

# List secrets
aws secretsmanager list-secrets --query 'SecretList[].Name'
```

### S3

```bash
# List bucket contents
aws s3 ls s3://{project}-{env}-files/ --recursive

# Upload a file
aws s3 cp local-file.jpg s3://{project}-{env}-files/uploads/

# Sync a directory
aws s3 sync ./local-dir s3://{project}-{env}-files/prefix/
```

### CloudFront

```bash
# Create cache invalidation
aws cloudfront create-invalidation \
  --distribution-id <distribution-id> \
  --paths "/*"
```

### Lambda

```bash
# Invoke a function
aws lambda invoke \
  --function-name {project}-backend-{env}-runMigrations \
  --payload '{"action": "show"}' \
  response.json

# View response
cat response.json
```

### SQS

```bash
# Get queue attributes (message count, etc.)
aws sqs get-queue-attributes \
  --queue-url <queue-url> \
  --attribute-names All

# Purge a queue (CAREFUL)
aws sqs purge-queue --queue-url <queue-url>
```
