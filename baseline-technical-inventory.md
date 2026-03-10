# EWX Power -- Baseline Technical Inventory

**Client:** EWX Power (Martin Janda)
**Date:** 2026-03-10
**Auditor:** DevSavant Product Engineering
**Deliverable:** Phase 1 Discovery -- Project Scan & Baseline Inventory
**Repositories:** `software-ewxfs-com-ewx-core` (Backend) | `software-ewxfs-com-ewx-next` (Frontend)

---

## Table of Contents

1. [Build Status](#1-build-status)
2. [Tech Stack Inventory](#2-tech-stack-inventory)
3. [Project Structure](#3-project-structure)
4. [Dependency Overview](#4-dependency-overview)
5. [AWS Services Catalog](#5-aws-services-catalog)
6. [Initial Audit Observations](#6-initial-audit-observations)

---

## 1. Build Status

### Backend (`ewx-core`)

| Check | Status | Notes |
|-------|--------|-------|
| Repository access | Pass | Bitbucket repo cloned successfully |
| `.ruby-version` present | Pass | `ruby-3.2.2` |
| `Gemfile.lock` present | Pass | Bundled with Bundler 2.6.9 |
| Dockerfile present | Pass | `ruby:3.2.2-slim-bullseye` base image |
| CI/CD pipeline | Pass | `bitbucket-pipelines.yml` -- 3 branches: `develop`, `qa`, `main` |
| `.env.example` | **Missing** | No `.env.example` or `.env.sample` present |
| `Procfile` | **Missing** | No Procfile; Puma started via Dockerfile CMD |
| Database config | Pass | `config/database.yml` -- MySQL 3-database setup (primary, better_auth, queue) |
| Credentials | Encrypted | `config/credentials.yml.enc` -- Rails credentials with master key |

### Frontend (`ewx-next`)

| Check | Status | Notes |
|-------|--------|-------|
| Repository access | Pass | Bitbucket repo cloned successfully |
| `package.json` present | Pass | `ewx-next@0.1.0` |
| Lock file present | Pass | `pnpm-lock.yaml` |
| Dockerfile present | Pass | `node:18-bullseye-slim` base image |
| CI/CD pipeline | **Not found** | No `bitbucket-pipelines.yml` in frontend repo |
| `.env.example` | **Missing** | No `.env.example`, `.env.sample`, or `.env.local.example` |
| `.nvmrc` / `.node-version` | **Missing** | No explicit Node version pinned |
| `engines` field | **Missing** | Not set in `package.json` |
| TypeScript config | Pass | `tsconfig.json` with `strict: true`, path alias `@/*` -> `./src/*` |

---

## 2. Tech Stack Inventory

### Core Framework Versions

| Technology | Version | Source | Notes |
|------------|---------|--------|-------|
| **Ruby** | 3.2.2 | `.ruby-version`, `Gemfile`, `Gemfile.lock` | `ruby 3.2.2p53` |
| **Rails** | 8.0.4 | `Gemfile.lock` (constraint: `~> 8.0.0, >= 8.0.0.1`) | API-only mode; `load_defaults 6.1` |
| **Node.js** | 18.x | `Dockerfile` (frontend) | Not pinned via `.nvmrc`; pipeline installs `nodesource 18.x` |
| **Next.js** | 15.4.2-canary.34 | `package.json` | **Canary pre-release build** |
| **React** | 19.1.1 | `package.json` | Latest React 19 |
| **TypeScript** | ^5 | `package.json` devDependencies | |

### Package Managers

| Repo | Manager | Lock File |
|------|---------|-----------|
| Backend | Bundler 2.6.9 + pnpm (for CDK) | `Gemfile.lock` + `pnpm-lock.yaml` |
| Frontend | pnpm | `pnpm-lock.yaml` |

### Databases

| Database | Usage | Adapter |
|----------|-------|---------|
| **MySQL 8** | Primary application data | `mysql2` gem (0.5.7) |
| **MySQL 8** | BetterAuth authentication DB | `mysql2` gem (separate DB) |
| **MySQL 8** | Solid Queue jobs | `mysql2` gem (separate DB) |
| **MySQL** (frontend) | BetterAuth sessions | `mysql2` npm package (^3.14.0) |

**Database architecture notes:**

- **Primary (backend):** The main Rails database holding all core domain models -- plants, facilities, components, tenants, users, alerts, activities, etc. Connected via the `mysql2` Ruby gem (0.5.7). Mapped to the `primary` entry in `config/database.yml`.
- **BetterAuth (backend):** A second, dedicated MySQL database exclusively for BetterAuth authentication (sessions, accounts, verifications). Uses the same `mysql2` gem but connects to a completely separate database. Has its own migration path at `db/migrate/better_auth/`.
- **Solid Queue (backend):** A third MySQL database dedicated to background job processing. Job metadata (enqueued, running, completed) lives here rather than in the primary database. Mapped to the `queue` entry in `config/database.yml` with its own schema at `db/queue_schema.rb`.
- **BetterAuth (frontend):** The Next.js frontend also maintains a direct MySQL connection via the `mysql2` npm package. BetterAuth runs server-side in Next.js (configured in the root `auth.ts`) and connects to MySQL to manage sessions and authentication -- completely bypassing the Rails API. This means the frontend holds its own database credentials and writes directly to the auth database, breaking the API-only architecture boundary. This is flagged as a critical finding in [Section 6](#6-initial-audit-observations).

### Deployment & CI/CD

| Component | Technology |
|-----------|-----------|
| CI/CD | Bitbucket Pipelines (backend only) |
| IaC | AWS CDK (TypeScript) -- both repos |
| Container Registry | AWS ECR |
| Container Orchestration | AWS ECS Fargate |
| Legacy Deployment | Capistrano to EC2 (still in Gemfile) |
| Docker base (backend) | `ruby:3.2.2-slim-bullseye` |
| Docker base (frontend) | `node:18-bullseye-slim` |

---

## 3. Project Structure

### Backend -- `ewx-core`

```
ewx-core/
├── app/
│   ├── blueprints/          # 74 Blueprinter JSON serializers
│   ├── controllers/         # 59 controllers (API v1 namespace + concerns)
│   ├── jobs/                # 9 background jobs (Solid Queue)
│   ├── models/              # 79 models (concerns, STI namespaces)
│   ├── mailers/
│   └── services/            # Service objects
├── bin/
│   └── deploy.sh            # AWS deploy script
├── config/
│   ├── deploy/              # Capistrano configs (staging, test, templates)
│   ├── environments/        # Rails env configs
│   ├── initializers/        # App initializers
│   ├── recurring.yml        # Solid Queue scheduled jobs
│   ├── database.yml         # Multi-database (primary, better_auth, queue)
│   ├── routes.rb            # API v1 routes
│   └── storage.yml          # Active Storage (local + S3)
├── db/
│   ├── migrate/
│   │   ├── primary/         # ~150+ main DB migrations
│   │   ├── better_auth/     # Auth DB migrations
│   │   └── queue/           # Solid Queue migrations
│   ├── schema.rb
│   ├── queue_schema.rb
│   └── seeds.rb
├── lib/
│   ├── revisedCDK/          # AWS CDK stacks (TypeScript)
│   │   └── stacks/          # Foundation, EC2DB, Secrets, etc.
│   ├── rails-cdk-stack.ts
│   ├── rails-environment-stack.ts
│   ├── service-stack.ts
│   ├── foundation-stack.ts
│   ├── lambda/              # Lambda handler for auth DB creation
│   └── notifications/
├── spec/                    # 17 test files (RSpec)
│   ├── acceptance/
│   ├── controllers/
│   ├── jobs/
│   ├── services/
│   └── inputs/
├── doc/
├── public/
├── Dockerfile
├── Gemfile / Gemfile.lock
├── bitbucket-pipelines.yml
├── package.json             # CDK dependencies (pnpm)
└── .rubocop.yml
```

**Code Statistics (Backend)**

| Category | Count |
|----------|-------|
| Models | 79 |
| Controllers | 59 |
| Blueprinter serializers | 74 |
| Background jobs | 9 |
| Database migrations | ~190 (across 3 DBs) |
| Test files | 17 |
| Total application files | ~350+ |

### Frontend -- `ewx-next`

```
ewx-next/
├── bin/
│   └── cdk-deploy.ts               # CDK entry (staging + production)
├── lib/
│   ├── application-stack.ts         # ECS Fargate stack
│   └── shared-resources.ts          # VPC + ECR shared stack
├── public/
│   ├── images/                      # SVGs, assets
│   ├── svgs/
│   └── sw.js                        # Service worker (push notifications)
├── src/
│   ├── app/                         # Next.js App Router
│   │   ├── (auth)/                  # sign-in, forgot-password, reset-password
│   │   ├── (action-required)/       # 2fa-setup, account-setup, no-client
│   │   ├── (authorized)/
│   │   │   ├── (main-pages)/        # dashboard, plants, facilities, components, etc.
│   │   │   └── (subpages)/          # user/client/data-management detail pages
│   │   ├── (link-redirects)/        # accept-invite
│   │   ├── api/                     # 49 API route handlers
│   │   └── lib/                     # Organization helpers
│   ├── apis/                        # 9 API client modules
│   ├── components/                  # ~230 components
│   │   ├── atoms/                   # Small reusable components
│   │   ├── ui/                      # shadcn/ui primitives
│   │   └── features/                # Feature-specific components
│   ├── hooks/                       # 5 custom React hooks
│   ├── lib/                         # Utilities, config, auth helpers
│   ├── messages/                    # i18n (en, nl, es, fr)
│   ├── middleware.ts                # Auth + subdomain + CORS
│   └── models/                      # TypeScript models/types
├── auth.ts                          # BetterAuth config (root)
├── auth-client.ts                   # Auth client config
├── components.json                  # shadcn/ui config
├── Dockerfile
├── next.config.ts
├── jest.config.js
├── tsconfig.json
├── package.json
└── pnpm-lock.yaml
```

**Code Statistics (Frontend)**

| Category | Count |
|----------|-------|
| Page/route components | 41 |
| Shared components | ~230 |
| API route handlers | 49 |
| Custom hooks | 5 |
| API client modules | 9 |
| Test files | 3 |
| i18n locales | 4 (en, nl, es, fr) |

---

## 4. Dependency Overview

### Key Ruby Gems (Backend)

| Category | Gem | Locked Version | Purpose |
|----------|-----|----------------|---------|
| **Framework** | `rails` | 8.0.4 | Web framework (API-only) |
| **Framework** | `puma` | 6.6.1 | Application server |
| **Framework** | `propshaft` | -- | Asset pipeline (Rails 8 default) |
| **Framework** | `rack-cors` | -- | CORS handling |
| **Database** | `mysql2` | 0.5.7 | MySQL adapter |
| **Auth** | `bcrypt` | 3.1.20 | Password hashing |
| **Auth** | `jwt` | 3.1.2 | JSON Web Tokens |
| **Auth** | `oauth2` | 2.0.18 | OAuth 2.0 client |
| **AWS** | `aws-sdk-s3` | 1.208.0 | S3 file storage |
| **AWS** | `aws-sdk-core` | 3.240.0 | AWS core SDK |
| **AWS** | `aws-sdk-kms` | 1.118.0 | Key Management Service |
| **Jobs** | `solid_queue` | 1.2.4 | Background job processing |
| **Jobs** | `mission_control-jobs` | 1.1.0 | Job dashboard UI |
| **Jobs** | `whenever` | 1.1.1 | Cron scheduling |
| **Serialization** | `blueprinter` | 1.2.1 | JSON serialization |
| **State Machine** | `aasm` | 5.5.2 | State machine DSL |
| **Soft Deletes** | `paranoia` | 3.1.0 | Soft delete support |
| **HTTP** | `httparty` | -- | HTTP client |
| **HTTP** | `rest-client` | -- | REST API client |
| **Email** | `mailtrap` | -- | Transactional email |
| **Email** | `sendgrid-ruby` | -- | SendGrid integration |
| **Monitoring** | `sentry-ruby` | 6.2.0 | Error tracking |
| **Monitoring** | `sentry-rails` | 6.2.0 | Rails Sentry integration |
| **Config** | `figaro` | -- | ENV variable management |
| **Dev** | `rspec-rails` | 8.0.2 | Test framework |
| **Dev** | `factory_bot_rails` | -- | Test factories |
| **Dev** | `rubocop` + plugins | -- | Linting (Shopify style) |
| **Dev** | `capistrano` + plugins | -- | Legacy deployment tool |

### Key Node Packages (Frontend)

| Category | Package | Version | Purpose |
|----------|---------|---------|---------|
| **Framework** | `next` | 15.4.2-canary.34 | React framework |
| **Framework** | `react` / `react-dom` | 19.1.1 | UI library |
| **UI Components** | `@radix-ui/*` (14 packages) | various | Headless UI primitives |
| **UI Components** | `lucide-react` | ^0.511.0 | Icon library |
| **UI Components** | `cmdk` | ^1.1.1 | Command palette |
| **UI Components** | `vaul` | ^1.1.2 | Drawer component |
| **UI Components** | `sonner` | ^2.0.1 | Toast notifications |
| **Styling** | `tailwindcss` | ^4 | CSS framework |
| **Styling** | `class-variance-authority` | ^0.7.1 | Variant styling |
| **Styling** | `tailwind-merge` | ^3.0.2 | Class merging |
| **Styling** | `framer-motion` | ^12.7.3 | Animations |
| **State/Data** | `@tanstack/react-query` | ^5.69.0 | Server state management |
| **Forms** | `@tanstack/react-form` | ^1.14.1 | Form management |
| **Validation** | `zod` | ^3.24.2 | Schema validation |
| **Auth** | `better-auth` | ^1.2.4 | Authentication framework |
| **Auth** | `jsonwebtoken` | ^9.0.2 | JWT handling |
| **Database** | `mysql2` | ^3.14.0 | MySQL client (BetterAuth) |
| **Database** | `pg` | ^8.14.1 | PostgreSQL client |
| **i18n** | `next-intl` | ^4.0.2 | Internationalization |
| **Maps** | `leaflet` / `react-leaflet` | ^1.9.4 / ^5.0.0 | Map rendering |
| **Maps** | `use-places-autocomplete` | ^4.0.1 | Google Places |
| **Charts** | `recharts` | 2.15.4 | Data visualization |
| **Flow** | `@xyflow/react` | ^12.6.4 | Node flow diagrams |
| **DnD** | `@dnd-kit/*` | various | Drag and drop |
| **Dates** | `date-fns` | ^4.1.0 | Date utilities |
| **Themes** | `next-themes` | ^0.4.6 | Theme switching |
| **Notifications** | `web-push` | ^3.6.7 | Web push notifications |
| **Email** | `nodemailer` | ^7.0.5 | Email sending |
| **HTTP** | `xior` | ^0.7.7 | HTTP client |
| **AWS/IaC** | `aws-cdk-lib` | ^2.38.0 | AWS CDK |
| **Testing** | `jest` | ^29.7.0 | Test runner |
| **Testing** | `@testing-library/react` | ^14.2.1 | React testing |
| **Build** | `babel-plugin-react-compiler` | 19.0.0-beta | React Compiler (experimental) |

---

## 5. AWS Services Catalog

### Combined AWS Service Usage

| AWS Service | Backend | Frontend | Evidence |
|-------------|---------|----------|----------|
| **ECS Fargate** | Yes | Yes | CDK stacks in both repos; `bitbucket-pipelines.yml` deploys to ECS clusters |
| **ECR** | Yes | Yes | Docker images pushed to ECR in both pipelines |
| **VPC** | Yes | Yes | Both CDK stacks reference existing VPC with private subnets |
| **Application Load Balancer** | Yes | Yes | `ApplicationLoadBalancedFargateService` pattern in both CDK stacks |
| **S3** | Yes | No | `config/storage.yml` (Active Storage); `aws-sdk-s3` gem |
| **RDS (MySQL)** | Yes | No | `config/deploy/test.rb`, `staging.rb` -- dedicated RDS hosts |
| **Secrets Manager** | Yes | Yes | Backend: `lib/revisedCDK/stacks/Secrets.ts`; Frontend: `application-stack.ts` |
| **Route 53** | Yes | Yes | Hosted zones for `ewx.msadvisors.services` in CDK stacks |
| **ACM (Certificates)** | Yes | Yes | TLS certificate ARNs in CDK configs (`eu-central-1`) |
| **IAM** | Yes | Yes | Task execution roles for ECS, EC2 principals |
| **Lambda** | Yes | No | `lib/lambda/create-auth-db.ts` -- auth database setup |
| **EC2** | Yes | No | Capistrano deploy targets (legacy); CDK EC2 DB stacks |
| **KMS** | Yes | No | `aws-sdk-kms` gem in `Gemfile.lock` |
| **CloudWatch** | Implicit | Implicit | ECS Fargate logging (default) |

### AWS Services NOT Detected

DynamoDB, SQS, SNS, Cognito, CloudFront, Parameter Store, EventBridge, Step Functions, API Gateway

### Infrastructure Topology

```
                     ┌─────────────────┐
                     │    Route 53      │
                     │  *.ewx.msadvisors│
                     │    .services     │
                     └────────┬────────┘
                              │
                     ┌────────┴────────┐
                     │  ACM Certificate │
                     │   (eu-central-1) │
                     └────────┬────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
     ┌────────┴────────┐             ┌────────┴────────┐
     │  ALB (Backend)  │             │  ALB (Frontend)  │
     │  staging.ewx... │             │  staging.app...  │
     └────────┬────────┘             └────────┬────────┘
              │                               │
     ┌────────┴────────┐             ┌────────┴────────┐
     │  ECS Fargate    │             │  ECS Fargate     │
     │  Rails API      │             │  Next.js App     │
     │  (ruby:3.2.2)   │             │  (node:18)       │
     └────────┬────────┘             └────────┬────────┘
              │                               │
     ┌────────┴────────┐             ┌────────┴────────┐
     │  RDS MySQL 8    │             │  Secrets Manager │
     │  (3 databases)  │             │  (env config)    │
     ├─────────────────┤             └──────────────────┘
     │  primary        │
     │  better_auth    │
     │  queue          │
     └─────────────────┘
              │
     ┌────────┴────────┐
     │  S3 (Active     │
     │  Storage)       │
     └─────────────────┘
```

### Deployment Environments

| Environment | Branch | Backend Domain | Frontend Domain |
|-------------|--------|----------------|-----------------|
| Staging | `develop` | `staging.ewx.msadvisors.services` | `staging.app.ewx.msadvisors.services` |
| QA | `qa` | `ewx.msadvisors.services` | -- |
| Production | `main` | `ewx.msadvisors.services` | `app.ewx.msadvisors.services` |

### AWS Region & Account

- **Region:** `eu-central-1` (Frankfurt)
- **Account:** `493777069702`
- **VPC:** Existing shared VPC (looked up by CDK, not created)

---

## 6. Initial Audit Observations

### Critical Findings (Require Immediate Investigation)

- **`config.load_defaults 6.1` on Rails 8.0.4** -- The application was upgraded to Rails 8 but still loads Rails 6.1 defaults. This means critical security, cookie, and framework behavior changes from Rails 7.0, 7.1, 7.2, and 8.0 are all disabled. This is a significant configuration debt and potential security gap.
- **Next.js canary build in production** -- The frontend runs `next@15.4.2-canary.34`, a pre-release canary version. This is inherently unstable and unsuitable for production workloads.
- **Near-zero test coverage** -- Backend has 17 spec files for 350+ application files (~3-5% coverage). Frontend has 3 test files for 230+ components (~1% coverage).
- **No `.env.example` in either repo** -- Neither codebase documents required environment variables, making onboarding and environment replication fragile and error-prone.
- **No Node.js version pinning (frontend)** -- Dockerfile uses `node:18` but no `.nvmrc`, `.node-version`, or `engines` field ensures consistency between local dev and CI.
- **Frontend has direct DB access** -- The Next.js app includes `mysql2` and `pg` as dependencies and has BetterAuth connected directly to MySQL. This means the frontend has database credentials and bypasses the API layer for authentication.

### Architectural Concerns

- **Dual authentication systems** -- BetterAuth runs in the Next.js frontend with direct MySQL access while the Rails backend has its own JWT-based authentication. This creates auth fragmentation and unclear session ownership.
- **Multi-database complexity** -- Three MySQL databases (primary, better_auth, queue) in the backend plus a separate BetterAuth MySQL connection from the frontend. No connection pooling layer detected.
- **Legacy Capistrano still present** -- The Gemfile includes `capistrano` and related gems despite the migration to ECS Fargate/CDK. This is dead code that adds confusion.
- **Mixed CDK locations** -- Backend CDK stacks live under `lib/` (shared with Rails autoload path), creating a collision between TypeScript infrastructure code and Ruby application code.
- **Frontend CI/CD gap** -- No `bitbucket-pipelines.yml` found in the frontend repo, suggesting manual or external deployment orchestration.
- **`config.api_only = true` with `config.assets.enabled = true`** -- Contradictory configuration; API-only apps should not need the asset pipeline.
- **49 API route handlers in Next.js** -- The frontend has a significant server-side API layer, suggesting business logic duplication with the Rails backend.
- **React Compiler beta in production** -- `babel-plugin-react-compiler` at `19.0.0-beta` is experimental and may introduce runtime bugs.

### Dependency Risks

- **`aws-cdk-lib` ^2.38.0 in frontend** -- This is a runtime production dependency rather than a devDependency, adding ~200MB to the build.
- **`@libsql/kysely-libsql` present** -- SQLite/Turso dependency present despite all databases being MySQL, suggesting abandoned experimentation.
- **Two HTTP clients in backend** -- Both `httparty` and `rest-client` gems are present, indicating inconsistent API consumption patterns.
- **`rspec_api_documentation` pinned to GitHub ref** -- Depends on a specific commit hash from a fork (`SchoolKeep/rspec_api_documentation`), creating a supply chain risk.
- **`pg` in frontend** -- PostgreSQL adapter present alongside MySQL, suggesting unclear database strategy or unused dependency.
- **Redundant email gems** -- Backend has both `mailtrap` and `sendgrid-ruby`, while frontend has `nodemailer`, splitting email concerns across three libraries.

### Code Quality Signals

- **190 migrations across 3 databases** -- High migration count suggests rapid iteration without consolidation. Needs schema review.
- **79 models with AASM + Paranoia** -- State machines and soft deletes increase query complexity. Multi-tenant scoping must be verified for both.
- **5 custom hooks vs 230 components** -- Low hook extraction ratio suggests component-level logic duplication.
- **Missing documentation** -- No API documentation (Swagger/OpenAPI), no architecture decision records (ADRs), no README in either codebase root with setup instructions.
- **4 i18n locales** -- Internationalization is partially implemented (en, nl, es, fr) but needs validation for completeness.

### Items for Deeper Audit Phases

| Priority | Area | Investigation Needed |
|----------|------|---------------------|
| P0 | Auth architecture | Map the full auth flow across BetterAuth (frontend) + JWT (backend); identify session overlap and security gaps |
| P0 | Multi-tenant isolation | Verify tenant scoping on all 79 models and 59 controllers; test cross-tenant data leakage |
| P0 | `load_defaults 6.1` impact | Catalog all behavioral changes between Rails 6.1 and 8.0 defaults; assess security implications |
| P1 | ETL pipeline | Audit the 9 background jobs for retry logic, error handling, and data integrity |
| P1 | API duplication | Compare the 49 Next.js API routes against the Rails API to identify overlap and divergence |
| P1 | Database schema | Review 190 migrations for constraint gaps, index coverage, and normalization issues |
| P2 | Frontend architecture | Assess component structure, state management patterns, and bundle size |
| P2 | Infrastructure | Evaluate ECS Fargate sizing (256 CPU / 512 MiB for frontend), auto-scaling config, and cost |
| P2 | CI/CD | Establish frontend pipeline; audit backend pipeline for security scanning and test gates |
| P3 | Dependency audit | Run security scan on all gems and npm packages; identify EOL or vulnerable versions |

---

_Baseline inventory produced by DevSavant Product Engineering -- Phase 1 Discovery_
