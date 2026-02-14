# Dev Shop Engagement & Communication Standards

This document defines how we work together. Every item here is a hard requirement, not a suggestion. Read it before writing your first line of code.

---

## Table of Contents

1. [Roles and Responsibilities](#roles-and-responsibilities)
2. [Tech Stack (Non-Negotiable)](#tech-stack-non-negotiable)
3. [Git Workflow](#git-workflow)
4. [Pull Request Standards](#pull-request-standards)
5. [Code Review Process](#code-review-process)
6. [Communication Channels](#communication-channels)
7. [Jira / Project Management](#jira--project-management)
8. [Definition of Done](#definition-of-done)
9. [Escalation Protocol](#escalation-protocol)
10. [Onboarding Checklist](#onboarding-checklist)
11. [What We Will Not Accept](#what-we-will-not-accept)

---

## Roles and Responsibilities

### Lead Engineers (Us)

We are two lead engineers who own the project end-to-end. We handle:

- **Architecture decisions**: Tech choices, module design, data modeling, API contract design.
- **Infrastructure provisioning**: Terraform modules, AWS account management, environment setup.
- **Production deployments**: We approve and trigger all production deploys.
- **Code review**: Every PR is reviewed by at least one of us before merge.
- **Client communication**: All client-facing communication goes through us.
- **Product decisions**: Feature scope, prioritization, and trade-offs.
- **OpenAPI spec design**: We design the API contracts. You implement them.

### Dev Shop (You)

You are responsible for **implementation** within the boundaries we define:

- **Feature implementation**: Build what the Jira ticket describes, following the coding standards exactly.
- **Writing tests**: Unit tests for services, integration tests for critical paths.
- **Writing migrations**: With proper `down()` methods, tested locally.
- **Updating OpenAPI spec**: If your work changes an API contract, update the spec first and get approval before implementing.
- **PR descriptions**: Clear, detailed, linked to Jira.
- **Jira updates**: Keep ticket status current. Move cards. Log time if requested.
- **Following standards**: Coding standards, infrastructure standards, naming conventions, file structure — all of it.
- **Asking questions early**: If something is unclear, ask in Slack immediately. Do not guess.

### What the Dev Shop Does NOT Do

- Make architecture decisions or introduce new libraries/tools.
- Modify Terraform infrastructure.
- Deploy to production.
- Communicate with the client.
- Change the tech stack or deviate from established patterns.
- Merge your own PRs.

---

## Tech Stack (Non-Negotiable)

The following technologies are mandated. Do not substitute, add to, or deviate from this list without explicit written approval from a lead engineer.

| Layer | Required Technology |
|---|---|
| Backend Runtime | Node.js 22.x, TypeScript (strict mode) |
| Backend Framework | Serverless Framework v3, AWS Lambda |
| ORM | TypeORM ^0.3.x |
| Database | PostgreSQL 16 (RDS) |
| Cache | Redis 7 (ElastiCache) |
| Queue | AWS SQS |
| Storage | AWS S3 + CloudFront |
| Auth | Firebase Auth (JWKS via `jose`) |
| API Specification | OpenAPI 3.0.3 |
| Type Generation | `openapi-typescript` |
| Frontend Framework | Next.js (App Router), React, TypeScript (strict) |
| Frontend API Client | `openapi-fetch` with generated types |
| CSS Framework | Tailwind CSS v4 |
| Linting | ESLint (`next/core-web-vitals`), Prettier (`prettier-plugin-tailwindcss`) |
| Infrastructure | Terraform, AWS |
| CI/CD | GitHub Actions |
| Version Control | Git, GitHub |

If you believe a library addition is necessary, raise it in Slack with a justification. We decide.

---

## Git Workflow

### Branching

- **`main`** is the protected trunk branch. No direct commits.
- All work happens on **feature branches** off `main`.
- Branch naming: `feature/{jira-ticket-id}-short-description` or `fix/{jira-ticket-id}-short-description`.

```
main
 └── feature/PROJ-42-add-thing-endpoint
 └── fix/PROJ-57-fix-mapper-null-check
```

### Commit Messages

Use descriptive commit messages. Reference the Jira ticket.

```
PROJ-42: Add getAll handler for things domain

- Create thing.entity.ts with TypeORM decorators
- Add thing.service.ts with findAll and findById
- Add thing.mapper.ts for entity -> API transform
- Wire up handler in serverless/functions/things.yml
```

### Branch Lifecycle

1. Create branch from `main`.
2. Implement changes.
3. Push branch to origin.
4. Open PR (see PR Standards below).
5. Address review feedback.
6. Lead engineer merges after approval.
7. Delete branch after merge.

---

## Pull Request Standards

Every PR must meet these requirements before it will be reviewed:

### PR Title

Format: `[PROJ-XX] Short descriptive title`

Example: `[PROJ-42] Add things domain with CRUD endpoints`

### PR Description

Every PR description must include:

```markdown
## Jira Ticket
[PROJ-42](https://your-org.atlassian.net/browse/PROJ-42)

## What Changed
- Added `things` domain with entity, service, mapper, handler
- Created migration for `things` table
- Updated openapi.yaml with /things endpoints
- Generated new API types

## How to Test
1. Run `npm run migration:run`
2. Start backend: `npm run dev`
3. POST /things with body: `{"name": "test"}`
4. GET /things — should return the created thing

## Screenshots (if UI changes)
[Attach screenshots here]

## Checklist
- [ ] OpenAPI spec updated (if API change)
- [ ] Types regenerated (`npm run openapi:generate`)
- [ ] Migration has `down()` method
- [ ] Migration tested locally (run + revert)
- [ ] No TypeScript errors (`npm run typecheck`)
- [ ] Linter passes (`npm run lint`)
- [ ] Tests pass (`npm run test`)
```

### PR Rules

- **One concern per PR**: Don't mix unrelated changes. One feature or fix per PR.
- **Keep PRs small**: Under 400 lines of real code changes when possible. Break large features into sequential PRs.
- **CI must pass**: All GitHub Actions checks must be green before requesting review.
- **No commented-out code**: Remove it, don't leave it.
- **No `console.log` debugging**: Clean up before pushing.
- **No `any` types without justification**: If you must use `any`, add a comment explaining why.

---

## Code Review Process

### Turnaround

- We aim to review PRs within **1 business day**.
- If a PR is urgent, flag it in Slack.

### What We Review For

- **Adherence to standards**: Domain structure, naming, patterns (handler/service/mapper/entity).
- **Correct use of OpenAPI types**: Generated types used, no manual type duplication.
- **Migration quality**: `down()` works, indexes created, naming correct.
- **Business logic in the right place**: Services, not handlers.
- **Error handling**: Custom error classes, proper response helpers.
- **Security**: No secrets, proper auth checks, input validation.
- **Test coverage**: Critical paths covered.

### Review Outcomes

- **Approved**: Merge happens. We merge, not you.
- **Changes Requested**: Address feedback and push new commits. Do not force-push or squash during review.
- **Questions**: Respond in the PR thread. If complex, discuss in Slack, then document the decision in the PR.

---

## Communication Channels

### Slack

- **Primary channel for real-time communication.**
- Use for: blockers, quick questions, status updates, standups, deployment coordination.
- Expect responses within a few hours during business hours.
- **Tag us** if something is blocking your progress.

### When to Use Slack

- You're blocked and need a decision.
- You have a question about a ticket's requirements.
- You found a bug or unexpected behavior.
- You want to propose a deviation from the standard approach.
- Deployment coordination.
- Daily standup updates (async is fine).

### When NOT to Use Slack

- Detailed technical discussions that should be in a PR review.
- Anything that should be documented (put it in Jira or the PR).

### Jira

- **Primary channel for task tracking and async project management.**
- All work must have a Jira ticket.
- If there's no ticket, ask for one before starting work.

---

## Jira / Project Management

### Board Type

**Kanban** — continuous flow, no sprints.

### Columns

| Column | Meaning |
|---|---|
| **Backlog** | Defined but not yet started. Prioritized top-to-bottom. |
| **In Progress** | Actively being worked on. Limit: 2 per developer. |
| **In Review** | PR opened, waiting for code review. |
| **Done** | Merged to `main`. Deployed or ready to deploy. |

### Ticket Requirements

Every ticket must have before you start working on it:

- **Title**: Clear, concise description of the work.
- **Description**: What needs to be done, with context.
- **Acceptance Criteria**: Checkable list of "done when..." items.
- **Priority**: Set by leads.
- **Assignee**: You.

### Your Jira Responsibilities

- **Move tickets** when status changes. Do not leave tickets in Backlog while actively coding.
- **Add comments** when you have questions, encounter blockers, or make decisions.
- **Link PRs** to tickets (GitHub integration should do this automatically if you use ticket IDs in branch names).
- **Log time** if requested.
- **Do not create tickets** unless asked. If you find work that needs a ticket, flag it in Slack and we'll create it.

### Priority Order

Work on tickets in the order they appear in the Backlog column (top = highest priority). If you're unsure, ask.

---

## Definition of Done

A ticket is "Done" when ALL of the following are true:

- [ ] Code implements the acceptance criteria in the ticket.
- [ ] Code follows all patterns in `coding-standards.md`.
- [ ] `openapi.yaml` updated if any API contract changed.
- [ ] Types regenerated (`npm run openapi:generate`) if OpenAPI changed.
- [ ] Migration written with working `down()` method (if schema changed).
- [ ] Migration tested locally: `run` then `revert` then `run` again.
- [ ] No TypeScript errors: `npm run typecheck` passes.
- [ ] Linter passes: `npm run lint` passes.
- [ ] Tests pass: `npm run test` passes.
- [ ] PR opened with proper description, linked to Jira ticket.
- [ ] All CI checks green.
- [ ] PR approved by a lead engineer.
- [ ] Jira ticket moved to "Done" after merge.

If any of these are missing, the PR will be sent back.

---

## Escalation Protocol

### Blockers

If you are blocked:

1. **Immediately** post in Slack with:
   - What you're working on (ticket ID).
   - What's blocking you.
   - What you've already tried.
2. Do NOT sit on a blocker for more than **2 hours** without escalating.
3. While waiting, pick up the next ticket in the Backlog.

### Disagreements

If you disagree with a technical decision:

1. Raise it in Slack or the PR with your reasoning.
2. We'll discuss and decide.
3. Once decided, implement the decision. The discussion is over.

### Scope Changes

If you discover that a ticket requires more work than described:

1. Comment on the Jira ticket with what you found.
2. Flag it in Slack.
3. **Do not expand scope on your own.** We'll re-scope the ticket or create a follow-up.

---

## Onboarding Checklist

Before writing any code, ensure you have:

- [ ] GitHub repo access (read + write on feature branches).
- [ ] Jira board access.
- [ ] Slack channel access.
- [ ] Read `coding-standards.md` completely.
- [ ] Read `infrastructure-standards.md` completely.
- [ ] Read `deployment-cheatsheet.md` completely.
- [ ] Read this document completely.
- [ ] Local environment set up and running (backend + frontend + Docker).
- [ ] Can run migrations locally.
- [ ] Can run type generation locally.
- [ ] Understand the OpenAPI contract-first workflow.
- [ ] AWS credentials configured (for SSM tunnel if needed).

---

## What We Will Not Accept

To be crystal clear:

| Will Not Accept | Explanation |
|---|---|
| PRs without linked Jira tickets | All work is tracked. No rogue commits. |
| Deviations from the tech stack | No new libraries, frameworks, or tools without approval. |
| Business logic in handlers | Handlers parse/respond. Services own logic. Period. |
| Hand-written API types | Generated from OpenAPI. Always. |
| Migrations without `down()` | Non-negotiable. Rollbacks must work. |
| `synchronize: true` anywhere near production | This is a firing offense. |
| Secrets in code or config files | Use Secrets Manager. |
| PRs with failing CI | Fix it before requesting review. |
| Self-merged PRs | We merge after approval. |
| Untracked work (no Jira ticket) | If there's no ticket, the work doesn't happen. |
| Direct commits to `main` | Feature branches and PRs only. |
| Expanding ticket scope without approval | Flag it, don't build it. |
| Architecture changes without discussion | Raise it, don't implement it. |
