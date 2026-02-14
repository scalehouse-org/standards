# Infrastructure & Deployment Standards

This document defines how we provision, configure, and deploy infrastructure. Everything is Infrastructure as Code. No manual console clicks in production. No exceptions.

---

## Table of Contents

1. [Core Principles](#core-principles)
2. [AWS Stack](#aws-stack)
3. [Terraform Architecture](#terraform-architecture)
4. [Module Standards](#module-standards)
5. [Environment Configuration](#environment-configuration)
6. [Networking Module](#networking-module)
7. [Security Module](#security-module)
8. [RDS Module](#rds-module)
9. [ElastiCache Module](#elasticache-module)
10. [SQS Module](#sqs-module)
11. [S3 + CloudFront Module](#s3--cloudfront-module)
12. [IAM Module](#iam-module)
13. [SSM Tunnel Module](#ssm-tunnel-module)
14. [Serverless Framework (Lambda)](#serverless-framework-lambda)
15. [CI/CD Pipelines](#cicd-pipelines)
16. [Local Development](#local-development)
17. [Secrets Management](#secrets-management)
18. [Security Standards](#security-standards)

---

## Core Principles

- **Everything is code**: Terraform for infrastructure, Serverless Framework for Lambda, GitHub Actions for CI/CD. Nothing is provisioned manually.
- **Modular Terraform**: Each AWS concern is its own module with `main.tf` / `variables.tf` / `outputs.tf`.
- **Environment parity**: Dev and prod use the same modules, different `tfvars`.
- **Remote state**: Terraform state lives in S3 with per-environment keys.
- **Tagged everything**: All resources tagged with `Project` and `Environment`.
- **Least privilege**: IAM roles scoped per module. No `*` policies.
- **No secrets in code**: All sensitive values in AWS Secrets Manager.

---

## AWS Stack

| Service | Purpose | Module |
|---|---|---|
| VPC | Isolated network | `networking` |
| Public Subnets (2+ AZs) | NAT Gateways, Internet Gateway | `networking` |
| Private Subnets (2+ AZs) | RDS, Lambda, ElastiCache, SSM | `networking` |
| NAT Gateway | Internet access for private subnets | `networking` |
| S3 VPC Endpoint | Cost-optimized S3 access from VPC | `networking` |
| RDS PostgreSQL 16 | Primary database | `rds` |
| ElastiCache Redis 7 | Cache layer | `elasticache` |
| S3 Bucket | File uploads | `s3-cloudfront` |
| CloudFront Distribution | CDN for S3 files (OAC) | `s3-cloudfront` |
| SQS Standard Queue | Async message processing | `sqs` |
| SQS Dead Letter Queue | Failed message retry | `sqs` |
| EC2 (t3.micro) | SSM tunnel for secure RDS access | `ssm-tunnel` |
| Lambda Functions | API handlers | Serverless Framework |
| API Gateway | HTTP routing + Lambda Authorizer | Serverless Framework |
| Secrets Manager | DB credentials, API keys | `rds` + manual |
| IAM Roles | Scoped per-module permissions | `iam` |

---

## Terraform Architecture

### Directory Structure

```
infra/
├── openapi.yaml                      # API contract (source of truth)
└── terraform/
    ├── main.tf                       # Root: orchestrates all modules
    ├── variables.tf                  # Root-level variables
    ├── outputs.tf                    # Root-level outputs
    ├── versions.tf                   # Provider versions, S3 backend config
    ├── backend-dev.hcl              # Backend config for dev state
    ├── backend-prod.hcl            # Backend config for prod state
    ├── environments/
    │   ├── dev.tfvars               # Dev environment values
    │   └── prod.tfvars              # Prod environment values
    ├── modules/
    │   ├── networking/              # VPC, subnets, NAT, endpoints
    │   ├── security/                # Security groups
    │   ├── rds/                     # PostgreSQL RDS
    │   ├── elasticache/             # Redis ElastiCache
    │   ├── sqs/                     # SQS queues
    │   ├── s3-cloudfront/           # S3 bucket + CloudFront CDN
    │   ├── iam/                     # IAM roles and policies
    │   └── ssm-tunnel/             # EC2 for secure DB access
    └── scripts/
        ├── init.sh                  # terraform init per environment
        ├── plan.sh                  # terraform plan per environment
        ├── apply.sh                 # terraform apply per environment
        ├── validate.sh              # fmt check + validate
        ├── ssh-tunnel.sh            # SSM port forwarding to RDS
        └── setup-state-bucket.sh    # Create S3 state bucket
```

### Root `main.tf` Pattern

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
    }
  }
}

module "networking" {
  source       = "./modules/networking"
  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
  common_tags  = local.common_tags
  # ...
}

module "security" {
  source       = "./modules/security"
  vpc_id       = module.networking.vpc_id
  common_tags  = local.common_tags
  # ...
}

module "rds" {
  source              = "./modules/rds"
  subnet_ids          = module.networking.private_subnet_ids
  security_group_id   = module.security.rds_security_group_id
  common_tags         = local.common_tags
  # ...
}

# ... remaining modules follow the same pattern
```

### Remote State Configuration

```hcl
# versions.tf
terraform {
  backend "s3" {
    bucket = "{project}-terraform-state"
    region = "us-east-1"
    # key is set per-environment via backend-{env}.hcl
  }
}
```

```hcl
# backend-dev.hcl
key = "dev/terraform.tfstate"
```

```hcl
# backend-prod.hcl
key = "prod/terraform.tfstate"
```

---

## Module Standards

Every Terraform module follows this structure:

```
modules/{module-name}/
├── main.tf           # Resource definitions
├── variables.tf      # Input variables
└── outputs.tf        # Output values
```

### Variable Rules

- Every variable **must** have a `description`.
- Every variable **must** have a `type`.
- Provide sensible `default` values where appropriate.
- Sensitive values (passwords, keys) **must** be marked `sensitive = true`.
- Modules receive `common_tags` explicitly as a variable.

```hcl
variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Deployment environment (dev, prod)"
  type        = string
}

variable "rds_db_password" {
  description = "Master password for RDS instance"
  type        = string
  sensitive   = true
  default     = null  # Auto-generated if null
}

variable "common_tags" {
  description = "Common tags applied to all resources"
  type        = map(string)
  default     = {}
}
```

### Naming Convention

All resources follow: `{project_name}-{environment}-{resource_descriptor}`

```hcl
resource "aws_s3_bucket" "files" {
  bucket = "${var.project_name}-${var.environment}-files"
}
```

### Tagging

Two layers of tagging:

1. **Provider `default_tags`** (root level): `Project` and `Environment` — applied to everything automatically.
2. **`common_tags` variable** (module level): Passed explicitly to modules for additional tags.

---

## Environment Configuration

### Dev (`environments/dev.tfvars`)

```hcl
environment         = "dev"
vpc_cidr            = "10.0.0.0/16"
single_nat_gateway  = true           # Cost optimization
rds_instance_class  = "db.t3.micro"
rds_allocated_storage = 20
rds_multi_az        = false
rds_skip_final_snapshot = true
elasticache_node_type   = "cache.t3.micro"
elasticache_num_nodes   = 1
```

### Prod (`environments/prod.tfvars`)

```hcl
environment         = "prod"
vpc_cidr            = "10.1.0.0/16"
single_nat_gateway  = false          # HA: NAT per AZ
rds_instance_class  = "db.t3.small"
rds_allocated_storage = 100
rds_multi_az        = true
rds_skip_final_snapshot = false
rds_backup_retention_period = 30
rds_deletion_protection = true
elasticache_node_type   = "cache.t3.small"
elasticache_num_nodes   = 2          # With replication
```

---

## Networking Module

Creates the VPC foundation:

- VPC with DNS support and DNS hostnames enabled.
- Public subnets (2+ AZs) with Internet Gateway.
- Private subnets (2+ AZs) for RDS, Lambda, ElastiCache.
- NAT Gateway(s) — single in dev (cost), multi-AZ in prod (HA).
- Route tables for public and private subnets.
- S3 VPC Gateway Endpoint (avoids NAT costs for S3 traffic).

### Key Outputs

```hcl
output "vpc_id" {}
output "public_subnet_ids" {}
output "private_subnet_ids" {}
```

---

## Security Module

Creates all security groups with least-privilege rules:

| Security Group | Ingress | Egress |
|---|---|---|
| **Lambda SG** | None | All (outbound to internet, RDS, Redis) |
| **RDS SG** | PostgreSQL (5432) from Lambda SG + SSM Tunnel SG | None |
| **ElastiCache SG** | Redis (6379) from Lambda SG | None |
| **SSM Tunnel SG** | None (SSM only, no SSH) | All |
| **VPC Endpoint SG** | HTTPS (443) from Lambda SG + SSM Tunnel SG | None |

### Key Outputs

```hcl
output "lambda_security_group_id" {}
output "rds_security_group_id" {}
output "elasticache_security_group_id" {}
output "ssm_tunnel_security_group_id" {}
```

---

## RDS Module

PostgreSQL 16 in private subnets:

- Not publicly accessible.
- Encrypted storage.
- Auto-generated password stored in Secrets Manager.
- Parameter group for PostgreSQL 16 tuning.
- Storage autoscaling enabled.
- CloudWatch logging enabled.
- Multi-AZ in production.
- Deletion protection in production.

### Key Outputs

```hcl
output "rds_instance_endpoint" {}
output "rds_db_credentials_secret_arn" {}
output "rds_db_name" {}
```

---

## ElastiCache Module

Redis 7 in private subnets:

- Single node in dev, replicated in prod.
- Accessible only from Lambda security group.
- Encryption in transit.

### Key Outputs

```hcl
output "redis_endpoint" {}
output "redis_port" {}
```

---

## SQS Module

Standard queues with dead letter queues:

- Main queue: 30-second visibility timeout, configurable retention.
- Dead letter queue: `maxReceiveCount: 3` — messages retried 3 times before going to DLQ.
- Both queues follow naming: `{project}-{env}-{queue-name}`.

### Key Outputs

```hcl
output "queue_url" {}
output "queue_arn" {}
output "dlq_url" {}
output "dlq_arn" {}
```

---

## S3 + CloudFront Module

File storage with CDN:

- S3 bucket: All public access blocked. Accessed only through CloudFront.
- CloudFront: Origin Access Control (OAC) — not legacy OAI.
- CORS configured for browser-based presigned URL uploads.
- Bucket policy restricts access to CloudFront distribution only.
- Optional Route 53 hosted zone for custom domain.

### Key Outputs

```hcl
output "files_bucket_name" {}
output "cloudfront_url" {}
output "cloudfront_distribution_id" {}
```

---

## IAM Module

Scoped IAM roles:

- SSM tunnel instance profile with `AmazonSSMManagedInstanceCore`.
- Lambda execution role with scoped policies (S3, SQS, Secrets Manager, VPC, CloudWatch Logs).
- No wildcard `*` resource policies.

---

## SSM Tunnel Module

Secure database access for developers:

- EC2 instance (t3.micro) in private subnet.
- Amazon Linux 2 AMI.
- No SSH — access exclusively via AWS Systems Manager Session Manager.
- Port forwarding: local 5432 -> RDS 5432.
- Toggled via `enable_ssm_tunnel` variable.

### Connection Command

```bash
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["<rds-endpoint>"],
    "portNumber": ["5432"],
    "localPortNumber": ["5432"]
  }'
```

---

## Serverless Framework (Lambda)

Backend API is deployed via Serverless Framework v3, **not** Terraform. Terraform provisions the infrastructure that Lambda runs on.

### Configuration Pattern

```yaml
# serverless.yml
service: {project}-backend
frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs22.x
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  memorySize: 256
  timeout: 30
  environment:
    DB_HOST: ${ssm:/path/to/db-host}
    # ... all env vars from SSM or Secrets Manager
  vpc:
    securityGroupIds:
      - ${param:lambdaSecurityGroupId}
    subnetIds: ${param:privateSubnetIds}
  iam:
    role:
      statements:
        - Effect: Allow
          Action: ['s3:PutObject', 's3:DeleteObject']
          Resource: 'arn:aws:s3:::${param:filesBucket}/*'

plugins:
  - serverless-dotenv-plugin
  - serverless-esbuild
  - serverless-offline
  - serverless-prune-plugin

functions:
  - ${file(serverless/functions/auth.yml)}
  - ${file(serverless/functions/health.yml)}
  - ${file(serverless/functions/{domain}.yml)}
  - ${file(serverless/functions/migrations.yml)}
```

### Function Definition Pattern

```yaml
# serverless/functions/{domain}.yml
getThings:
  handler: src/domains/{domain}/handlers/index.getAll
  description: Get all things
  events:
    - http:
        path: /things
        method: get
        cors: true
        authorizer:
          name: authorize
          resultTtlInSeconds: 300

createThing:
  handler: src/domains/{domain}/handlers/index.create
  description: Create a thing
  events:
    - http:
        path: /things
        method: post
        cors: true
        authorizer:
          name: authorize
          resultTtlInSeconds: 300
```

### Lambda Authorizer

Defined in `serverless/functions/auth.yml`. Validates Firebase ID tokens via JWKS. Caches authorization results for 300 seconds.

---

## CI/CD Pipelines

Three GitHub Actions workflows handle the full lifecycle:

### 1. `ci.yml` — Continuous Integration

**Trigger**: Push or PR to `main`.

**Jobs** (parallel):
- **Backend**: `npm ci` -> `openapi:generate` -> `typecheck` -> `lint` -> `test`
- **Frontend**: `npm ci` -> `openapi:generate` -> `build`

### 2. `deploy-{env}.yml` — Environment Deployment

**Trigger**: Push to `main` (dev), manual dispatch (prod).

**Jobs** (sequential):

```
terraform-validate -> terraform-plan -> terraform-apply
                                              |
                                     deploy-backend (Serverless)
                                              |
                                     run-migrations (Lambda invoke)
                                              |
                                     deploy-frontend
                                              |
                                     deployment-summary
```

Key details:
- **Authentication**: OIDC via IAM role (no long-lived AWS keys).
- **Terraform plan**: Uploads artifact, comments on PR with plan output.
- **Terraform apply**: Only on push to `main` (not on PR).
- **Backend deploy**: Retrieves DB credentials from Secrets Manager, sets VPC config from Terraform outputs.
- **Migrations**: Invokes migration Lambda, validates success.
- **Frontend deploy**: Builds and deploys (Amplify, Vercel, or S3+CloudFront — depends on project).

### 3. `db-migrate.yml` — Database Migrations

**Trigger**: Manual dispatch or workflow call.

**Inputs**:
- `action`: `run`, `show`, or `revert`
- `environment`: `dev` or `prod`

**Action**: Invokes the migration Lambda with the specified action.

### OIDC Authentication

```yaml
permissions:
  id-token: write
  contents: read

- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::{account-id}:role/{GithubOIDCRole}
    aws-region: us-east-1
```

No long-lived AWS access keys stored in GitHub secrets.

---

## Local Development

### Docker Compose

```yaml
# backend/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    ports: ['5432:5432']
    environment:
      POSTGRES_DB: {project_db}
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

  localstack:
    image: localstack/localstack:latest
    ports: ['4566:4566']
    environment:
      SERVICES: s3,sqs
    volumes:
      - localstack_data:/var/lib/localstack

  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  localstack_data:
  redis_data:
```

### Local Environment Variables

```bash
# backend/.env
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=postgres
DB_NAME={project_db}
NODE_ENV=local

AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=local
AWS_SECRET_ACCESS_KEY=local

S3_ENDPOINT=http://localhost:4566
SQS_ENDPOINT=http://localhost:4566
UPLOAD_FILES_BUCKET_NAME={project}-local-files
FILES_CDN_URL=http://localhost:4566/{project}-local-files

REDIS_HOST=localhost
REDIS_PORT=6379

FIREBASE_PROJECT_ID={firebase-project-id}
```

### Running Locally

```bash
# Start infrastructure
cd backend && docker-compose up -d

# Run migrations
npm run migration:run

# Start backend (serverless-offline on port 3001)
npm run dev

# Start frontend (Next.js on port 3000)
cd frontend && npm run dev
```

---

## Secrets Management

### What Goes in Secrets Manager

- Database credentials (auto-generated by Terraform RDS module).
- Firebase configuration (project ID, etc.).
- Stripe API keys.
- Any third-party API keys.
- Webhook secrets.

### What Goes in SSM Parameter Store

- Non-sensitive configuration that changes per environment.
- Feature flags.
- CDN URLs.

### What Goes in Environment Variables (Serverless)

- References to SSM or Secrets Manager paths.
- `NODE_ENV`.
- Non-sensitive, non-changing values.

### Rule: No secrets in code, config files, or version control. Ever.

---

## Security Standards

| Concern | Standard |
|---|---|
| Network | RDS and ElastiCache in private subnets only |
| Database access | SSM tunnel only — no public endpoint, no SSH |
| Lambda networking | Runs in private subnets with NAT for internet |
| S3 | All public access blocked, CloudFront OAC only |
| IAM | Scoped per-module, no wildcard resources |
| Secrets | AWS Secrets Manager, never in code |
| CI/CD auth | OIDC — no long-lived AWS keys in GitHub |
| API auth | Firebase JWT via Lambda Authorizer |
| Encryption | RDS storage encrypted, S3 encrypted, Redis in-transit |
| Backup | RDS automated backups (30-day retention in prod) |
| Deletion protection | Enabled in prod for RDS |
