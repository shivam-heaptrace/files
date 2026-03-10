# TRD -- EWX Power Platform

> **Purpose:** Define HOW the EWX Power platform is built technically. This document captures the current-state architecture as discovered during the Phase 1 codebase audit. Every technical detail traces to evidence found in the `ewx-core` (backend) and `ewx-next` (frontend) repositories.

---

## Cover Page

| Field | Value |
|-------|-------|
| Project Name | EWX Power -- Platform Rebuild |
| Client | EWX Power (Martin Janda) |
| Document Author | DevSavant - Product Engineering |
| Date | 2026-03-10 |
| Version | 1.0 |
| Status | Draft |
| PRD Reference | Pending (Phase 1 Discovery) |
| Baseline Inventory | [baseline-technical-inventory.md](baseline-technical-inventory.md) |
| Codebase Audit | [rails-api-codebase-audit.md](rails-api-codebase-audit.md) |

---

## Introduction

### Document Purpose

This TRD documents the **current technical state** of the EWX Power platform as discovered during the Phase 1 Discovery audit. It serves three functions:

1. **Baseline reference** -- Captures the existing architecture, technology choices, data design, integrations, and infrastructure so the team has a single authoritative source of truth.
2. **Rebuild input** -- Provides the technical foundation for keep-vs-rebuild decisions and future rebuild TRD iterations.
3. **Risk register** -- Maps critical findings from the codebase audit into each relevant TRD section so that the rebuild addresses them structurally.

### Source Repositories

| Repository | Type | Host |
|------------|------|------|
| `software-ewxfs-com-ewx-core` | Backend (Rails API) | Bitbucket |
| `software-ewxfs-com-ewx-next` | Frontend (Next.js) | Bitbucket |

### Audience

- DevSavant Product Engineering team (rebuild leads)
- DevSavant DevOps / Infrastructure
- QA Engineering
- EWX Power technical stakeholders

---

## Technical Architecture

### Architecture Pattern

The platform follows a **two-tier client-server** pattern with an API-only Rails backend and a Next.js frontend, both deployed as separate ECS Fargate services behind Application Load Balancers.

However, the architecture is **not a clean API-first separation**. The Next.js frontend maintains its own direct database connections (BetterAuth via MySQL) and hosts 49 server-side API route handlers, creating a hybrid where business logic and data access are split across both tiers.

### Request Flow

**Backend (Rails API):**

```
Request
  → Controller (+ Authorization, Pagination, ExceptionHandling concerns)
    → Service (BaseService → Context with call/succeed/fail!)
      → Input (parameter validation + sanitization)
      → Model (ActiveRecord)
        → Database
    → Blueprint (JSON serialization)
  ← Response
```

**Frontend (Next.js):**

```
Browser
  → Next.js App Router (41 pages)
    → Middleware (auth check, subdomain routing, CORS)
      ├── Server Components → API client modules (xior) → Rails API
      ├── API Route Handlers (49 routes) → Direct logic / Rails proxy
      └── BetterAuth (auth.ts) → MySQL (direct connection)
  ← Response
```

### System Diagram

```
                         ┌─────────────────────┐
                         │      Route 53        │
                         │  *.ewx.msadvisors    │
                         │      .services       │
                         └──────────┬───────────┘
                                    │
                         ┌──────────┴───────────┐
                         │   ACM Certificate     │
                         │    (eu-central-1)     │
                         └──────────┬───────────┘
                                    │
                ┌───────────────────┴───────────────────┐
                │                                       │
       ┌────────┴────────┐                     ┌────────┴────────┐
       │  ALB (Backend)  │                     │ ALB (Frontend)  │
       │ staging.ewx...  │                     │ staging.app...  │
       └────────┬────────┘                     └────────┬────────┘
                │                                       │
       ┌────────┴────────┐                     ┌────────┴────────┐
       │  ECS Fargate    │◀── REST API ───────▶│  ECS Fargate    │
       │  Rails 8 API    │    (JSON)           │  Next.js 15     │
       │  (ruby:3.2.2)   │                     │  (node:18)      │
       └────────┬────────┘                     └────────┬────────┘
                │                                       │
       ┌────────┴──────────────────┐           ┌────────┴────────┐
       │  RDS MySQL 8              │◀──────────│ BetterAuth      │
       │  ┌─────────────────────┐  │  Direct   │ (mysql2 npm)    │
       │  │ primary (52 tables) │  │  DB conn  └─────────────────┘
       │  │ better_auth (8)     │  │
       │  │ queue (11)          │  │
       │  └─────────────────────┘  │
       └────────┬──────────────────┘
                │
       ┌────────┴────────┐
       │  S3 (Active     │
       │  Storage)       │
       └─────────────────┘
```

### Audit Findings (Architecture)

| ID | Finding | Severity |
|----|---------|----------|
| -- | Frontend bypasses API for auth (direct DB access) | Critical concern |
| -- | 49 API route handlers in Next.js duplicate/proxy Rails logic | Architectural debt |
| -- | Dual auth systems (BetterAuth frontend + JWT backend) | Critical concern |
| -- | CDK infrastructure code mixed with `lib/` (Rails autoload path) | Medium concern |

---

## Technology Stack

### Current State

| Layer | Technology | Version | How It's Used | Audit Finding |
|-------|-----------|---------|---------------|---------------|
| Frontend | Next.js | 15.4.2-canary.34 | App Router, 41 pages, standalone output | **Canary pre-release in production** |
| Frontend | React | 19.1.1 | UI rendering with React 19 features | -- |
| Frontend | TypeScript | ^5 | Strict mode, path aliases | -- |
| Frontend | Tailwind CSS | ^4 | Styling via `@tailwindcss/postcss` (v4) | -- |
| Frontend UI | shadcn/ui + Radix | various | 14 Radix primitives, `new-york` style | -- |
| Backend | Ruby | 3.2.2 | `.ruby-version` pinned | -- |
| Backend | Rails | 8.0.4 (API-only) | `config.api_only = true` | **`load_defaults 6.1` -- 4 major versions behind** |
| Backend | Puma | 6.6.1 | Application server | -- |
| Database | MySQL | 8.x | 3 databases via `mysql2` gem (0.5.7) | -- |
| Auth (FE) | BetterAuth | ^1.2.4 | Direct MySQL from Next.js | **Bypasses API boundary** |
| Auth (BE) | JWT + bcrypt | custom `TokenManager` | Legacy path | **Password check disabled; tokens never expire** |
| Jobs | Solid Queue | 1.2.4 | Background processing + scheduling | **`perform_now` blocks scheduler** |
| Serialization | Blueprinter | 1.2.1 | 74 JSON serializers | -- |
| Error Tracking | Sentry | 6.2.0 | `sentry-ruby` + `sentry-rails` | -- |
| File Storage | Active Storage | Built-in | S3 adapter in staging/production | -- |
| Package Manager (FE) | pnpm | -- | `pnpm-lock.yaml` | -- |
| Package Manager (BE) | Bundler | 2.6.9 | `Gemfile.lock` | -- |
| IaC | AWS CDK | TypeScript | Both repos | -- |
| CI/CD | Bitbucket Pipelines | -- | Backend only (3 branches) | **Frontend has no pipeline** |
| Hosting | AWS ECS Fargate | -- | Both services | -- |

### Package Managers

| Repo | Manager | Lock File |
|------|---------|-----------|
| Backend | Bundler 2.6.9 + pnpm (for CDK) | `Gemfile.lock` + `pnpm-lock.yaml` |
| Frontend | pnpm | `pnpm-lock.yaml` |

---

## Technical Requirements Mapping

> **Note:** Business requirements (BRs) have not yet been formally documented in a PRD. The following maps **observed capabilities** in the current codebase to the domain areas they serve. This section will be updated with formal BR → TR traceability once the PRD is produced.

### Observed Domain Capabilities

| Domain Area | Backend Coverage | Frontend Coverage | Audit Status |
|-------------|-----------------|-------------------|--------------|
| **Tenant management** | Full CRUD, tenant settings, data units | Organization switcher, config pages | Tenant isolation **broken** (S-6) |
| **User management** | CRUD, roles, bulk actions, activity logs | Admin user pages, invite flow | Auth **disabled** (S-1), RBAC **unenforced** (S-7) |
| **Plant management** | CRUD, attachments, components, metrics, maintenance | Plant list, detail, add pages | Functional |
| **Facility management** | CRUD, plants (with bulk) | Facility list, detail, monitoring | Functional |
| **Component management** | CRUD, attachments, connections, metrics, specs | Component list, detail, add, type config | Functional |
| **Data ingestion (ETL)** | 9 jobs, scheduled via Solid Queue | -- | **12 critical/high findings** (E-1 through E-12) |
| **Data visualization** | Headline metrics, overview tiles, data tables, indicators | Dashboard, charts (Recharts), flow diagrams (XYFlow) | Partially implemented |
| **Alerting** | Full CRUD, alert groups, notification jobs | Alert pages, alert group config | Race conditions (E-6) |
| **Authentication** | Dual-path (JWT + BetterAuth) | BetterAuth + middleware | **Systemically broken** |
| **Internationalization** | Not implemented | 4 locales (en, nl, es, fr) via `next-intl` | Partial |
| **Maps / geographics** | Not implemented | Leaflet + Google Places | Frontend-only |
| **Push notifications** | -- | Service worker + `web-push` | Frontend-only |

---

## Non-Functional Requirements

### Security (Current State)

| Requirement | Current Implementation | Audit Rating |
|-------------|----------------------|--------------|
| Authentication | Password check **disabled**; any email grants access | **F** |
| Token management | JWT with no expiration, no revocation, signing key defaults to `""` | **F** |
| Authorization | RBAC permissions defined but **never enforced** | **F** |
| Multi-tenant isolation | `member?` does not scope to tenant -- cross-tenant access possible | **F** |
| Input validation | `params.permit!` in 16+ locations bypasses Strong Parameters | **D** |
| CORS | Entire `rack-cors` initializer commented out | **F** |
| Credential storage | OAuth/API credentials stored as plaintext in `data_sources` table | **D** |
| Rate limiting | None on any endpoint (login, register, password reset) | **D** |
| Secrets management | AWS Secrets Manager for infra; hardcoded `admin/password123` in app | **D** |

### Performance (Current State)

| Metric | Current State | Observation |
|--------|--------------|-------------|
| API response time | Unknown (no APM configured beyond Sentry) | Sentry captures errors, not latency |
| ETL throughput | Unknown; `perform_now` blocks scheduler | Pipeline cannot sustain overlapping cycles |
| Database queries | No query profiling; missing indexes on 10+ FK columns | Degraded JOIN/CASCADE performance expected at scale |
| Frontend bundle | Unknown; `aws-cdk-lib` (~200MB) in production deps | Build may be unnecessarily large |
| Background jobs | 3 threads shared across all job types | No queue separation; priority inversion possible |

### Scalability (Current State)

| Component | Configuration | Notes |
|-----------|--------------|-------|
| Backend ECS | Managed by CDK | Fargate auto-scaling not explicitly configured in backend CDK |
| Frontend ECS | 256 CPU / 512 MiB, min 1 / max 5 | Auto-scales on 50% CPU utilization |
| Database | Single RDS instance, 3 logical databases | No read replicas detected |
| ETL | Single scheduler, `perform_now` execution | Cannot parallelize; blocked by synchronous execution |

---

## Data Design

### Multi-Database Architecture

| Database | Purpose | Tables | Physical Separation |
|----------|---------|--------|---------------------|
| `primary` | Core business data + Solid Queue | 52 domain + 11 queue | `ewx_development` / `ewx_production` |
| `better_auth` | BetterAuth JS session/account data | 8 | Separate database |
| `queue` | Solid Queue (configured) | -- | Shares primary physical DB |

**Frontend DB connection:** The Next.js app connects directly to the `better_auth` MySQL database via the `mysql2` npm package, bypassing the Rails API entirely.

### Schema Summary

- **63 business tables** in the primary database
- **8 tables** in the Better Auth database
- **MySQL 8** with `utf8mb4_0900_ai_ci` collation
- **190 migration files** spanning Oct 2024 -- Feb 2026

### Key Domain Entities

| Entity Group | Tables | Key Models |
|-------------|--------|------------|
| Tenants | tenants, tenant_locations, tenant_settings | `Core::Tenant`, `Tenants::User`, `Tenants::Role` |
| Plants | plants, plant_types | `Core::Plant` |
| Facilities | facilities, facility_plants | `Core::Facility` |
| Components | components, component_types, component_connection_points | `Core::Component`, `Core::ComponentType` |
| Data Pipeline | data_sources, data_end_points, data_end_point_raws, data_metrics, data_metric_points, data_indicators, data_indicator_points | `Tenants::DataSource`, `Tenants::DataEndPoint`, `Tenants::DataMetric` |
| Alerts | alerts, alert_notifications, alert_groups | `Core::Alert`, `Core::AlertNotification` |
| Users | users, roles, tokens, user_configurations | `Core::User`, `Tenants::Role` |
| Files | file_uploads, active_storage_* | ActiveStorage integration |

### Schema Findings

| ID | Finding | Severity | Impact |
|----|---------|----------|--------|
| D-1 | Missing NOT NULL constraints on ~20 core columns (FKs, user identity, tenant scoping) | HIGH | Invalid database states allowed |
| D-2 | Missing indexes on 10+ FK/polymorphic columns | HIGH | Degraded query performance at scale |
| D-3 | Plaintext credential storage in `data_sources` (client_secret, password, tokens) | HIGH | Credential exposure risk |
| D-4 | Missing foreign key constraints on 8+ columns | HIGH | Referential integrity not enforced |
| D-6 | 8+ data migrations mixed with schema migrations | MEDIUM | Fragile rollbacks |
| D-8 | Numeric values stored as VARCHAR in some columns | MEDIUM | Type coercion issues |
| D-13 | Duplicate index on `tenants.identifier` (both unique and non-unique) | MEDIUM | Wasted storage, confusing schema |
| D-14 | Migration version class mismatch (6.1 vs 8.0) | LOW | Cosmetic but indicates upgrade debt |

### Migrations Strategy (Current)

- **Tool:** Rails ActiveRecord migrations
- **Versioning:** Timestamped migration files, 3 separate migration paths (`primary/`, `better_auth/`, `queue/`)
- **Rollback:** Some migrations are irreversible (`remove_column` without type); data migrations in `change` blocks prevent clean rollback
- **Execution:** Migrations run via ECS Fargate task (`bundle exec rails db:migrate`) triggered from Bitbucket Pipelines before service deployment

---

## Integrations & APIs

### Third-Party Services

| Service | Purpose | Authentication | Status |
|---------|---------|---------------|--------|
| External Data APIs | ETL ingestion (energy monitoring data) | OAuth2 (`rest-client`) | Functional (GET only; POST/Execute **not implemented**) |
| Mailtrap | Transactional email (development) | API key | Configured |
| SendGrid | Transactional email (production) | API key | Configured |
| Sentry | Error tracking | DSN | Configured for backend |
| Google Maps | Geocoding, Places autocomplete | API key (frontend) | Frontend-only |
| Nodemailer | Email sending (frontend) | SMTP config | Frontend-only |
| Web Push | Browser push notifications | VAPID keys | Frontend-only |

### API Structure (Backend)

All endpoints namespaced under `/v1/`:

| Namespace | Resources | Custom Endpoints |
|-----------|-----------|------------------|
| `/v1/users` | CRUD, register, authenticate | activity_logs, alert_groups, uploads, bulk actions |
| `/v1/tenants` | CRUD | alert_notifications, headline_metrics, overview_tiles, ticker_config, data_visualizations |
| `/v1/tenants/:id/...` | 24+ nested resources | indicators, summary, visualizations, feed_status, api_data, data_table |
| `/v1/plants/:id/...` | attachments, components, connections, metrics, maintenance | -- |
| `/v1/facilities/:id/...` | plants (with bulk) | -- |
| `/v1/components/:id/...` | attachments, connections, metrics, specs | -- |
| `/v1/alerts` | Full CRUD | -- |
| `/v1/activities` | Read-only | -- |
| `/health`, `/version` | Health checks | -- |
| `/jobs` | Mission Control dashboard | HTTP Basic Auth (`admin`/`password123` -- **hardcoded**) |

### Frontend API Routes (Next.js)

49 server-side API route handlers under `src/app/api/`. These include:

- BetterAuth session management
- Proxy endpoints to the Rails API
- Direct business logic handlers

**Concern:** The frontend API layer creates ambiguity about which tier owns specific business logic. A full comparison against the Rails API is recommended (see [Baseline Inventory, Section 6](baseline-technical-inventory.md#6-initial-audit-observations)).

### Missing Integrations

| Integration | Requirement Source | Status |
|------------|-------------------|--------|
| Weather forecast | Dashboard requirements | Not implemented |
| Solar production forecast | Dashboard requirements | Not implemented |
| Google Maps / geocoding (backend) | Dashboard requirements | Not implemented (frontend-only) |
| Real-time subscriptions / WebSockets | NFRs | Not implemented |
| Screen card configuration (show/hide/ordering) | Dashboard spec | Not implemented |

---

## Infrastructure

### Environments

| Environment | Purpose | Backend URL | Frontend URL | Branch |
|-------------|---------|------------|--------------|--------|
| Staging | Pre-production testing | `staging.ewx.msadvisors.services` | `staging.app.ewx.msadvisors.services` | `develop` |
| QA | Quality assurance | `ewx.msadvisors.services` | -- | `qa` |
| Production | Live application | `ewx.msadvisors.services` | `app.ewx.msadvisors.services` | `main` |

### AWS Region & Account

| Parameter | Value |
|-----------|-------|
| Region | `eu-central-1` (Frankfurt) |
| Account | `493777069702` |
| VPC | Existing shared VPC (looked up by CDK, not created) |
| Domain | `ewx.msadvisors.services` |

### AWS Services in Use

| AWS Service | Backend | Frontend | Evidence |
|-------------|---------|----------|----------|
| **ECS Fargate** | Yes | Yes | CDK stacks; Bitbucket Pipelines deploys |
| **ECR** | Yes | Yes | Docker image repositories |
| **VPC** | Yes | Yes | Private subnets in both CDK stacks |
| **Application Load Balancer** | Yes | Yes | `ApplicationLoadBalancedFargateService` |
| **S3** | Yes | No | Active Storage for file uploads |
| **RDS (MySQL)** | Yes | No | Dedicated RDS hosts per environment |
| **Secrets Manager** | Yes | Yes | DB creds, auth secrets, API keys |
| **Route 53** | Yes | Yes | DNS hosted zones |
| **ACM (Certificates)** | Yes | Yes | TLS termination at ALB |
| **IAM** | Yes | Yes | ECS task execution roles |
| **Lambda** | Yes | No | Auth database setup (`create-auth-db.ts`) |
| **EC2** | Yes | No | Legacy Capistrano targets + CDK EC2 DB stacks |
| **KMS** | Yes | No | `aws-sdk-kms` gem |
| **CloudWatch** | Implicit | Implicit | ECS Fargate default logging |

**Not detected:** DynamoDB, SQS, SNS, Cognito, CloudFront, Parameter Store, EventBridge, Step Functions, API Gateway.

### CI/CD Pipeline

**Backend (Bitbucket Pipelines):**

| Stage | Branch | Actions |
|-------|--------|---------|
| Build | `develop`, `qa`, `main` | Docker build → Push to ECR |
| Deploy | `develop`, `qa`, `main` | CDK deploy → Run migrations (ECS task) → Update ECS service |

Pipeline image: `python:3.9.18-slim-bullseye` with Node 18 + AWS CLI + CDK installed at runtime.

**Frontend:** No CI/CD pipeline detected in the repository. Deployment mechanism is unclear -- likely manual or orchestrated externally.

### Docker Configuration

| Repo | Base Image | Exposed Port | Build Cmd |
|------|-----------|-------------|-----------|
| Backend | `ruby:3.2.2-slim-bullseye` | -- | `rails db:migrate && puma` |
| Frontend | `node:18-bullseye-slim` | 3000 | `pnpm install → pnpm build → pnpm start` |

### Monitoring & Alerting

| Tool | Purpose | Coverage |
|------|---------|----------|
| Sentry | Error tracking | Backend only (`sentry-ruby`, `sentry-rails`) |
| CloudWatch Logs | Container logging | Both (ECS default) |
| Mission Control Jobs | Job dashboard | Backend (`/jobs` endpoint) |

**Gaps:** No APM, no frontend error tracking, no structured logging, no uptime monitoring, no alerting thresholds configured.

---

## Testing & QA

### Current Test Coverage

**Backend (RSpec):**

| Layer | App Files | Test Files | Effective Coverage |
|-------|-----------|------------|--------------------|
| Models | 79 | 0 | **0%** |
| Controllers | 59 | 1 | ~2% |
| Services | 119 | 3 | ~3% |
| Jobs | 9 | 2 (empty placeholders) | **0%** |
| Inputs | 103 | 4 | ~4% |
| Acceptance/Integration | -- | 7 | ~6 of 47+ endpoints |
| **Total** | **350+** | **17** | **~3-5%** |

**Frontend (Jest):**

| Layer | App Files | Test Files | Effective Coverage |
|-------|-----------|------------|--------------------|
| Components | ~230 | 3 | **~1%** |
| API routes | 49 | 0 | **0%** |
| Hooks | 5 | 0 | **0%** |
| **Total** | **280+** | **3** | **~1%** |

### Test Tooling

| Repo | Framework | Config | Runner |
|------|-----------|--------|--------|
| Backend | RSpec + FactoryBot | `spec/rails_helper.rb`, SimpleCov configured | `rspec` |
| Frontend | Jest + Testing Library | `jest.config.js`, `jest.setup.js` | `jest` (via `pnpm test`) |

### Critical Gaps

- **Zero model specs** -- `spec/models/` directory does not exist
- **Zero API route tests** -- Frontend's 49 route handlers are untested
- **Empty job specs** -- Both existing job spec files are auto-generated placeholders
- **No E2E testing** -- No Playwright, Cypress, or browser-based tests
- **No integration/request specs** -- API contract is unverified
- **SimpleCov never run to completion** -- No `coverage/` directory exists

### Audit Rating

| Repo | Rating | Assessment |
|------|--------|------------|
| Backend | **F** | ~3-5% coverage; zero safety net |
| Frontend | **F** | ~1% coverage; 3 tests for 280+ files |

---

## Assumptions & Limitations

### Technical Constraints (Discovered)

| Constraint | Impact | Source |
|-----------|--------|--------|
| `config.load_defaults 6.1` on Rails 8.0.4 | 4 major versions of framework behavior changes are disabled | `config/application.rb` |
| Next.js canary build (15.4.2-canary.34) | Pre-release runtime in production; may have undocumented bugs | `package.json` |
| No `.env.example` in either repo | Environment setup is undocumented; onboarding requires tribal knowledge | Both repos |
| No Node.js version pinning | Potential inconsistency between local dev, CI, and production | Frontend repo |
| MySQL-only architecture | No read replicas, no connection pooling layer detected | `config/database.yml` |
| Bitbucket Pipelines backend-only | Frontend deployment is manual or externally managed | Repo root |
| `aws-cdk-lib` as production dependency | ~200MB library included in frontend production build | `package.json` |

### Dependencies on External Systems

| System | Dependency Type | Risk |
|--------|---------------|------|
| External data APIs (energy monitoring) | ETL source | API downtime = silent data loss (no retry logic) |
| AWS Secrets Manager | Runtime configuration | Service fails to start if secrets unavailable |
| RDS MySQL | Primary data store | Single point of failure (no replicas detected) |
| Google Maps API | Frontend geocoding | Requires API key; client-side only |
| Mailtrap / SendGrid | Email delivery | Dual provider configuration; unclear routing logic |

---

## Cost Analysis

### Current Infrastructure Costs (Estimated Monthly)

| Service | Configuration | Estimated Cost |
|---------|--------------|----------------|
| ECS Fargate (Backend) | Variable task size | ~$30-60 |
| ECS Fargate (Frontend) | 256 CPU / 512 MiB, 1-5 tasks | ~$15-40 |
| RDS MySQL | Single instance (size TBD) | ~$30-100 |
| S3 (Active Storage) | Low volume (file uploads) | ~$1-5 |
| ALB (x2) | One per service | ~$35 |
| ECR | Docker image storage | ~$1-5 |
| Secrets Manager | ~10 secrets | ~$4 |
| Route 53 | Hosted zone + records | ~$1 |
| ACM | TLS certificates | Free |
| **Estimated Total** | | **~$120-250/month** |

> **Note:** Actual costs depend on RDS instance class, ECS task scaling behavior, and data transfer volumes. A detailed cost analysis requires access to the AWS Cost Explorer for account `493777069702`.

---

## Appendices

### Glossary

| Term | Definition |
|------|-----------|
| BetterAuth | JavaScript-based authentication library that manages sessions, accounts, and 2FA directly via a database connection |
| Blueprinter | Ruby gem for fast, declarative JSON serialization |
| CDK | AWS Cloud Development Kit -- infrastructure-as-code using TypeScript |
| ECS Fargate | Serverless container orchestration service (no EC2 instance management) |
| ETL | Extract, Transform, Load -- the data ingestion pipeline from external energy monitoring APIs |
| Solid Queue | Rails 8 background job framework backed by a database (replaces Redis-based alternatives) |
| STI | Single Table Inheritance -- Rails pattern where multiple model types share one database table |
| AASM | Acts As State Machine -- Ruby gem for defining state transitions on models |
| Paranoia | Ruby gem implementing soft deletes (marks records as deleted without removing them) |

### Key File Locations

| File | Repo | Purpose |
|------|------|---------|
| `config/application.rb` | Backend | Rails configuration (API-only, `load_defaults 6.1`) |
| `config/database.yml` | Backend | Multi-database configuration (primary, better_auth, queue) |
| `config/routes.rb` | Backend | API route definitions (v1 namespace) |
| `config/recurring.yml` | Backend | Solid Queue scheduled job definitions |
| `config/storage.yml` | Backend | Active Storage configuration (S3) |
| `bitbucket-pipelines.yml` | Backend | CI/CD pipeline (develop, qa, main) |
| `lib/revisedCDK/` | Backend | AWS CDK infrastructure stacks |
| `auth.ts` | Frontend | BetterAuth server configuration |
| `src/middleware.ts` | Frontend | Auth check, subdomain routing, CORS |
| `next.config.ts` | Frontend | Next.js configuration (standalone, i18n, security headers) |
| `lib/application-stack.ts` | Frontend | ECS Fargate CDK stack |
| `bin/cdk-deploy.ts` | Frontend | CDK deployment entry point |

### Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-10 | DevSavant Product Engineering | Initial draft from Phase 1 Discovery scan |

---

## Related Documents

- [Baseline Technical Inventory](baseline-technical-inventory.md) -- Full dependency and structure inventory
- [Rails API Codebase Audit](rails-api-codebase-audit.md) -- Detailed backend audit with 44 findings
- [Platform Rebuild README](../README.md) -- Project context and engagement timeline
- [Project Evaluation](../ewx-power-project-evaluation.md) -- Risk score and hour estimates
- [ADR Template](../../../processes/adr-template.md) -- For future architecture decisions
- [PRD Template](../../../processes/prd-template.md) -- For formal requirements documentation
- Full TRD template: `internal-docs/processes/trd-template.md`

---

_DevSavant - Product Engineering_
_EWX Power Platform Rebuild -- Phase 1 Discovery_
