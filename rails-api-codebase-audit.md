# EWX Power -- Rails API Codebase Audit

**Client:** EWX Power (Martin Janda)
**Date:** 2026-03-10
**Auditor:** DevSavant Product Engineering
**Deliverable:** Phase 1 Discovery -- Codebase & Architecture Audit
**Codebase:** `software-ewxfs-com-ewx-core` (Rails 8.0.0, Ruby 3.2.2, MySQL 8)
**Cross-references:** [Project Evaluation](../ewx-power-project-evaluation.md) | [Phase 1 Execution Plan](../phase1-option4-execution-plan.md)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Codebase Inventory](#2-codebase-inventory)
3. [Code Structure and Organization](#3-code-structure-and-organization)
4. [Test Coverage and Quality](#4-test-coverage-and-quality)
5. [Database Schema and Migrations](#5-database-schema-and-migrations)
6. [API Endpoint Structure](#6-api-endpoint-structure)
7. [Authentication and Authorization](#7-authentication-and-authorization)
8. [Multi-Tenant Isolation](#8-multi-tenant-isolation)
9. [ETL and Background Jobs](#9-etl-and-background-jobs)
10. [Consolidated Finding Register](#10-consolidated-finding-register)
11. [Root Cause Analysis](#11-root-cause-analysis)
12. [Keep vs. Rebuild Inputs](#12-keep-vs-rebuild-inputs)
13. [Recommendations](#13-recommendations)

---

## 1. Executive Summary

### Overview

The EWX Core API is a Rails 8 API-only application with **350+ application files** spanning models, controllers, services, blueprints, and input validators. At a structural level, the codebase demonstrates familiarity with Rails conventions -- the service object pattern, Blueprinter serialization, and multi-database configuration are reasonable architectural choices.

However, the implementation contains **critical security vulnerabilities**, **near-zero test coverage**, an **unreliable ETL pipeline**, and **broken multi-tenant isolation** that collectively represent a systemic quality failure.

### Audit Scorecard

| Area | Rating | Summary |
|------|--------|---------|
| Code Structure | **C+** | Good directory layout; poor implementation patterns |
| Test Coverage | **F** | ~3-5% effective coverage; 0% model coverage |
| Database Schema | **C** | Sound domain modeling; weak constraints and security |
| API Endpoints | **B-** | Clean REST structure; 3 of 5 integrations missing |
| Authentication | **F** | Password check disabled; tokens never expire |
| Multi-Tenancy | **F** | Tenant isolation check does not scope to tenant |
| ETL Pipeline | **D-** | Core flow exists; no retry, no transactions, silent failures |
| Overall | **D** | **Not production-ready** |

### Key Numbers

| Metric | Value | Assessment |
|--------|-------|------------|
| Application files | 350+ | Broad domain coverage |
| Test files | 17 | **Catastrophically low** |
| Effective test coverage | ~3-5% | **Unacceptable** |
| Database tables | 63 + 8 (auth) | Adequate |
| Migrations | 190 | Heavy churn |
| **Critical findings** | **10** | **Blocks production use** |
| **High findings** | **13** | Significant remediation needed |
| **Medium findings** | **16** | Ongoing quality concerns |
| Low findings | 5 | Minor cleanup |
| **Total findings** | **44** | -- |

### Verdict

**The codebase is not production-ready.** The combination of a disabled password check, broken tenant isolation, near-zero test coverage, and an unreliable ETL pipeline means this application cannot safely serve real users or ingest real data without significant remediation.

---

## 2. Codebase Inventory

### Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Ruby | 3.2.2 |
| Framework | Rails (API-only) | 8.0.0 |
| Database | MySQL | 8.x |
| Background Jobs | Solid Queue | Latest |
| Job Dashboard | Mission Control Jobs | Latest |
| Web Server | Puma | ~6.0 |
| Auth (Legacy) | JWT via `bcrypt` + custom `TokenManager` | -- |
| Auth (New) | Better Auth (JS library integration) | -- |
| Serialization | Blueprinter | Latest |
| Error Tracking | Sentry | Latest |
| File Storage | Active Storage (S3) | Built-in |
| Deployment | Capistrano + Docker + Bitbucket Pipelines | -- |

### File Distribution

| Directory | Files | Purpose |
|-----------|-------|---------|
| `app/models/` | 79 | Domain models with 14 concerns |
| `app/controllers/` | 59 | V1-namespaced API controllers with 10 concerns |
| `app/services/` | 119 | Service objects (Context pattern) |
| `app/blueprints/` | 73 | JSON serialization |
| `app/inputs/` | 103 | Request parameter validation |
| `app/jobs/` | 9 | Background jobs (ETL + notifications) |
| `app/decorators/` | 10 | Notification decorators |
| `lib/` | 36 | Auth engine, encryption, notifications |
| `db/migrate/` | 190 | Database migrations |
| `spec/` | 17 | Test suite |

### Multi-Database Architecture

| Database | Purpose | Physical Separation |
|----------|---------|---------------------|
| `primary` | Core business data (52 tables) + Solid Queue (11 tables) | `ewx_development` / `ewx_production` |
| `better_auth` | Better Auth JS session/account data (8 tables) | Separate DB |
| `queue` | Solid Queue (configured) | Shares primary DB |

---

## 3. Code Structure and Organization

### Architecture Pattern

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

### What Works

- **Service objects** encapsulate business logic via a consistent `BaseService` / `Context` pattern with `call`, `succeed`, `fail!` semantics
- **Input objects** validate and sanitize request parameters before they reach services
- **Blueprinter** provides a clean, consistent JSON serialization layer across 73 files
- **Namespaced structure** mirrors the domain cleanly: `Core::`, `Tenants::`, `Plants::`, `Components::`, `Sensors::`
- **Multi-database setup** separates authentication from business data

### What Doesn't

| Issue | Severity | Impact |
|-------|----------|--------|
| `params.permit!` used in **16+ locations** | HIGH | Completely bypasses Rails' Strong Parameters -- the primary defense against mass assignment attacks |
| `default_scope` on 5+ models | MEDIUM | Implicit query filtering causes unexpected behavior in joins and subqueries |
| `constantize` on derived strings | MEDIUM | Fragile class resolution; potential remote code execution if user input reaches it |
| `status_id` getter methods on `DataSource` and `DataMetric` trigger DB writes and external API calls | HIGH | Reading a model attribute causes side effects -- breaks serialization, caching, and query expectations |
| 119 services are mechanical wrappers | LOW | Most services delegate directly to `Queries::Engine` or `Core::Global::Contexts::Mutation` with no domain logic |

### Rating

| Aspect | Grade |
|--------|-------|
| Directory structure | **B+** |
| Separation of concerns | **B** |
| Implementation patterns | **C-** |
| Rails convention adherence | **C** |

---

## 4. Test Coverage and Quality

### Coverage by Layer

| Layer | App Files | Test Files | Effective Coverage |
|-------|-----------|------------|--------------------|
| Models | 79 | 0 | **0%** |
| Controllers | 59 | 1 | ~2% |
| Services | 119 | 3 | ~3% |
| Jobs | 9 | 2 (both empty placeholders) | **0%** |
| Inputs | 103 | 4 | ~4% |
| Acceptance/Integration | -- | 7 | ~6 of 47+ endpoints |
| **Total** | **350+** | **17** | **~3-5%** |

### Critical Gaps

**Zero model specs.** The `spec/models/` directory does not exist. All 79 models -- validations, associations, scopes, callbacks, and 14 concerns -- are completely untested.

**Empty job specs.** Both existing job spec files contain only a `pending` placeholder. The remaining 7 jobs have no spec at all. The ETL pipeline that ingests external data has zero test coverage.

**No request or integration specs.** Neither `spec/requests/` nor `spec/integration/` directories exist. The API contract is unverified.

**Minimal factories.** Only ~24 factories for 79+ models. Most define no attributes ("bare factories"), making them fragile and unusable for rigorous testing.

**SimpleCov configured but no output.** The coverage tool is set up in `rails_helper.rb` but no `coverage/` directory exists -- the suite has likely never been run to completion.

### Quality of Existing Tests

| File | Quality | Notes |
|------|---------|-------|
| `spec/inputs/base_input_spec.rb` | Good | Tests behavior, edge cases, error messages |
| `spec/services/users/contexts/authentication_spec.rb` | Good | Tests success/failure paths |
| `spec/acceptance/v1/tenants/users/component_types_spec.rb` | Good | Best acceptance test -- verifies actual mutation |
| `spec/acceptance/v1/users_spec.rb` | Moderate | Tests 401, search, user scoping |
| Other acceptance tests (5 files) | Weak | Status code checks only; updates never verify the value changed |
| `spec/controllers/users_controller_spec.rb` | Weak | Stubs all service calls; tests routing, not behavior |
| All job specs (2 files) | None | Auto-generated placeholders |

### Rating: F (Critical)

The test suite provides no safety net. Any modification carries high regression risk. There is no validation that business rules are enforced at the data layer, and no verification that the API contract holds.

---

## 5. Database Schema and Migrations

### Schema Summary

- **63 business tables** in the primary database (52 domain + 11 Solid Queue)
- **8 tables** in the Better Auth database
- **MySQL 8** with `utf8mb4_0900_ai_ci` collation
- **190 migration files** spanning Oct 2024 -- Feb 2026

### Issues Identified

#### Missing NOT NULL Constraints (HIGH)

Core columns that should never be null lack constraints, allowing invalid database states:

| Category | Affected Columns |
|----------|-----------------|
| **Join table FKs** | `facility_plants.facility_id`, `facility_plants.plant_id`, `tenant_locations.tenant_id`, `tenant_locations.location_id`, `component_connection_point_component_types` (both FKs) |
| **User identity** | `users.email`, `users.first_name`, `users.last_name` |
| **Tenant scoping** | `plants.tenant_id`, `component_types.tenant_id`, `data_metrics.tenant_id`, `data_sources.tenant_id`, `data_end_points.tenant_id`, `roles.tenant_id` |
| **Core identity** | `statuses.name`, `severities.name`, `locations.name` |

#### Missing Indexes (HIGH)

10+ foreign key columns lack indexes, degrading JOIN and CASCADE performance:

| Table | Missing Index On |
|-------|-----------------|
| `file_uploads` | `resource_type` + `resource_id` (polymorphic -- zero indexes) |
| `notification_notifications` | `context_type` + `context_id` (polymorphic -- zero indexes) |
| `notification_notifications` | `recipient_type` + `recipient_id` (polymorphic -- zero indexes) |
| `tokens` | `owner_type` + `owner_id` (polymorphic -- zero indexes) |
| `data_metrics` | `from_data_metric_id` (self-referential) |
| `component_connection_points` | `to_network_id` |
| `tenants` | `settings_data_unit_id` |

#### Missing Foreign Key Constraints (HIGH)

8+ columns reference other tables with no database-level FK enforcement:

- `data_metric_points.data_metric_id` (part of composite PK but no FK constraint)
- `data_metrics.from_data_metric_id` (self-referential)
- `user_configurations.user_id`
- `tenants.settings_data_unit_id`

#### Plaintext Credential Storage (HIGH)

The `data_sources` table stores OAuth/API credentials as plaintext strings:

```
client_id       :string(255)
client_secret   :string(255)
username        :string(255)
password        :string(255)
access_token    :text(16777215)
refresh_token   :text(16777215)
```

These should use Rails encrypted attributes or `attr_encrypted` at minimum.

#### Migration Quality Issues (MEDIUM)

- **8+ data migrations** mixed with schema migrations (model-dependent `create` calls in `change` blocks)
- **3+ irreversible migrations** use `remove_column` without specifying types
- **Duplicate index** on `tenants.identifier` (both unique and non-unique)

#### Better Auth Database

The Better Auth database uses camelCase columns and singular table names (standard for the JS library). No FK constraints or indexes on `userId` columns in most tables.

### Rating

| Aspect | Grade |
|--------|-------|
| Domain modeling | **B** |
| NOT NULL constraints | **D** |
| Foreign key constraints | **C+** |
| Index coverage | **C** |
| Migration discipline | **C-** |
| Credential security | **D** |

---

## 6. API Endpoint Structure

### Route Inventory

All endpoints are namespaced under `/v1/`:

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
| `/v1/data_intervals` | Read-only | -- |
| `/v1/statuses` | Read-only | -- |
| `/v1/locations` | Full CRUD | -- |
| `/v1/sensors/monitor_networks` | Full CRUD | -- |
| `/health`, `/version` | Health checks | -- |
| `/jobs` | Mission Control dashboard | HTTP Basic Auth |

### Gap Analysis vs. Dashboard Requirements

**Present:** Tenant management, facilities, plants, components, component types, data metrics, data visualizations, data indicators, alert groups, alert notifications, user management, roles, locations, data sources, data endpoints.

**Missing or incomplete:**

| Gap | Status | Impact |
|-----|--------|--------|
| Weather forecast integration | Not implemented | 1 of 5 required integrations |
| Solar production forecast | Not implemented | 1 of 5 required integrations |
| Google Maps / geocoding | Not implemented | 1 of 5 required integrations |
| Real-time subscriptions / WebSockets | Not implemented | Required per NFRs |
| i18n configuration | Not implemented | Required per NFRs |
| Screen card configuration (show/hide/ordering) | Not implemented | Required per Dashboard spec |
| Deep link resolution | Not implemented | Required per NFRs |
| Complex dashboard aggregations | Partially implemented | Data visualization endpoints exist but completeness unclear |

### Rating

| Aspect | Grade |
|--------|-------|
| Route organization | **B+** |
| Endpoint coverage | **B-** |
| Response format consistency | **B** |
| Error handling | **C** |

---

## 7. Authentication and Authorization

### Architecture

Two parallel authentication paths exist:

| Path | Header | Signing Key | Token Lifetime |
|------|--------|-------------|----------------|
| Legacy JWT | `Authorization: Bearer <token>` | `SIGNING_SECRET` env var | **Never expires** |
| Better Auth | `x-access-token` | `TOKEN_SIGNING_KEY` env var | Session-based |

`current_user` tries the Better Auth session first, then falls back to legacy JWT.

### Findings

#### S-1: Password Verification Disabled (CRITICAL)

The password authentication check is **commented out** in production code. The replacement only checks whether a user record exists for the supplied email -- any valid email grants full access.

```ruby
# File: app/services/users/contexts/authentication.rb
# Developer comment: "the authentication check is only working on development
#   - is it possibly password length?"

# user.authenticate(input.password) ? succeed(...) : fail!(...)   ← DISABLED
user.present? ? succeed(authentication_token) : fail!(...)         ← ACTIVE
```

#### S-2: Hardcoded Credentials on `/jobs` Dashboard (CRITICAL)

Mission Control Jobs is protected by `admin` / `password123`, committed in plaintext to the repository.

```ruby
# File: config/initializers/mission_control_jobs.rb
MissionControl::Jobs.http_basic_auth_user = 'admin'
MissionControl::Jobs.http_basic_auth_password = 'password123'
```

#### S-3: CORS Configuration Disabled (CRITICAL)

The entire `rack-cors` initializer is commented out. No CORS headers are returned.

#### S-4: JWT Tokens Never Expire (HIGH)

No `exp`, `iat`, or `nbf` claims in JWT payloads. Tokens are valid indefinitely with no revocation mechanism.

#### S-5: Signing Key Defaults to Empty String (HIGH)

If `SIGNING_SECRET` is missing from the environment, the signing key falls back to `""`, making tokens effectively unsigned and forgeable.

```ruby
# File: lib/encryption/token_manager.rb
def signing_key
  ENV.fetch(SIGNING_KEY, "")   # Empty string fallback
end
```

#### S-7: RBAC Permissions Defined but Never Enforced (HIGH)

`Tenants::Role` defines 6 granular boolean permissions (`can_dashboard_view`, `can_dashboard_create`, `can_dashboard_export`, `can_metric_view`, `can_metric_create`, `can_metric_export`). No controller, service, or middleware ever checks them. No policy framework (Pundit, CanCanCan) is installed.

#### S-8: `params.permit!` Bypasses Strong Parameters (HIGH)

Used in 16+ locations across controllers and concerns. Attackers can inject arbitrary fields in update requests (e.g., set `is_ewx_administrator: true` on their user record).

#### Additional Findings

| ID | Finding | Severity |
|----|---------|----------|
| S-9 | No JWT algorithm restriction on decode (potential `alg: none` bypass) | MEDIUM |
| S-10 | No rate limiting on any endpoint (login, register, password reset all unprotected) | MEDIUM |
| S-11 | No password complexity requirements | MEDIUM |
| S-12 | `SECRET_KEY_BASE` (Rails master key) dual-purposed as Better Auth API key | MEDIUM |

### Rating: F (Critical)

The authentication layer has multiple vulnerabilities that independently block production deployment. Combined, they represent a complete security failure.

---

## 8. Multi-Tenant Isolation

### How It Works

All tenant-scoped controllers call `tenants_authorize(using: :tenant_id)`, which delegates to `Core::Tenant#member?` to verify the current user has access to the requested tenant.

### S-6: `member?` Does Not Scope to Tenant (CRITICAL)

The membership check queries whether a user belongs to **any** tenant, not the **specific** tenant being accessed:

```ruby
# File: app/models/concerns/tenants/role_concerns.rb
def member?(user_id, role_names = [])
  relation = ::Tenants::User.includes(:role).where(user_id:)
  # ^^^ Missing: tenant_id: id
  relation = relation.where(role: { name: role_names }) if role_names.present?
  relation.exists?
end
```

**Exploit scenario:** User belongs only to Tenant A. User sends a request with `tenant_id=<Tenant B>`. The system resolves Tenant B, calls `tenant_b.member?(user.id)`, finds the user's membership in Tenant A, and returns `true`. The request proceeds with Tenant B's data.

The same issue exists in `administrator?` -- it checks if the user is an admin in **any** tenant.

### Additional Tenant Isolation Concerns

| ID | Finding | Severity |
|----|---------|----------|
| S-6a | Data scoping relies on `tenant_id` in request params, not server-side enforcement | MEDIUM |
| S-6b | `/v1/tenants` index returns all tenants to any authenticated user | LOW |

### Positive Note

All 24+ tenant-scoped controllers consistently wrap their actions in `tenants_authorize`. The pattern is correctly applied -- the bug is in the underlying implementation, not in its usage.

---

## 9. ETL and Background Jobs

### Pipeline Architecture

```
                         ┌─────────────────────────────────────────┐
                         │          Solid Queue Scheduler           │
                         │  (config/recurring.yml, runs in Puma)   │
                         └────────────────┬────────────────────────┘
                                          │
                    ┌─────────────────────┼──────────────────────┐
                    ▼                     ▼                      ▼
            5-min Ingestion      Hourly Ingestion     Hourly/Daily/Monthly
           (interval 17520)     (interval 1460)        Aggregation
                    │                     │                      │
                    ▼                     ▼                      ▼
         DataEndPointScheduledJob    (same)     DataMetricAggregationScheduledJob
                    │                                            │
                    ▼                                            ▼
         DataEndPoint.ingestion(interval)         DataMetricAggregation.aggregate()
                    │
      ┌─────── by identifier type ──────┐
      ▼                                 ▼
  lat/long → iterate locations    serial → iterate components
      │                                 │
      └──────────┬──────────────────────┘
                 ▼
    DataEndPoint#ingest_for_params()
         │
         ├── connect()  ──→  External API (RestClient GET)
         ├── DataEndPointRaw.create()  ──→  Store raw response
         ├── flatten_hash() + apply mappings  ──→  Transform
         ├── DataMetricPoint.upsert_all()  ──→  Load
         ├── alert_out_of_range()  ──→  AlertNotification
         └── load_indicator_points()  ──→  DataIndicatorPoint
```

### Findings

#### E-1: Zero Retry Logic (CRITICAL)

Both `retry_on` and `discard_on` are commented out in `ApplicationJob`. Any transient failure (network glitch, DB deadlock, API rate limit) permanently loses data for that interval with no recovery path.

```ruby
# File: app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # retry_on ActiveRecord::Deadlocked        ← COMMENTED OUT
  # discard_on ActiveJob::DeserializationError  ← COMMENTED OUT
end
```

#### E-2: `perform_now` Blocks the Scheduler Thread (CRITICAL)

All 7 scheduled entries use `command: "Job.perform_now(...)"` instead of the Solid Queue `class:` + `args:` format. Jobs execute **synchronously** inside the scheduler process. If a 5-minute feed takes longer than 5 minutes, the scheduler blocks and all subsequent tasks are delayed.

```yaml
# File: config/recurring.yml
ingest_5_minute_feeds:
  command: "DataEndPointScheduledJob.perform_now(17520)"    # ← Blocks scheduler
  schedule: every 5 minutes
```

#### E-3: Missing HTTP Methods (CRITICAL)

Only `call_via_get` is implemented. `call_via_post` and `call_via_execute` are referenced in the `connect()` routing but **do not exist** anywhere in the codebase. Any data endpoint configured for POST will crash at runtime with `NoMethodError`.

#### E-4: Silent Failure on API Errors (CRITICAL)

When an external API call times out or returns an error, the method returns `'{}'` (empty JSON). The pipeline then: creates a raw data record with an empty body, silently skips data insertion, and logs an activity as if ingestion succeeded.

```ruby
# File: app/models/tenants/data_end_point.rb
rescue RestClient::RequestTimeout => err
  feed_alert_notification(4, err)
  Rails.logger.debug 'TIMEOUT'
  '{}'    # ← Returns empty JSON; caller continues as if successful
```

#### E-5: No Transaction Wrapping (CRITICAL)

The ingestion pipeline writes to 5+ tables (`DataEndPointRaw`, `DataMetricPoint`, `AlertNotification`, `DataIndicatorPoint`, `ActivityLog`) without transaction wrapping. A failure midway leaves the database in an inconsistent state.

#### Additional ETL Findings

| ID | Finding | Severity | Impact |
|----|---------|----------|--------|
| E-6 | Race conditions in alert creation (check-then-act without locking) | HIGH | Duplicate alerts |
| E-7 | Instance variable bleed across loop iterations (`@latitude`, `@longitude`, `@serial_number`) | HIGH | Corrupted data records |
| E-8 | `alert_id` column stores boolean (`true`/`false` → `1`/`0`), not actual alert FK | HIGH | Foreign key corruption |
| E-9 | HTTP timeout read from `Core::Alert.find_by_id(4).range_upper.to_i` on every call | HIGH | Runtime crash if record missing; DB hit per request |
| E-10 | Raw data trimming job exists but is **never scheduled** | MEDIUM | Unbounded table growth (LONGBLOB body column) |
| E-11 | Aggregation method accepts `since` param but never uses it | MEDIUM | Full-history scans on every aggregation run |
| E-12 | `load_indicator_points` recalculates all indicators for all facilities and plants on every ingestion | MEDIUM | O(I x (1+F+P)) queries per ingestion cycle |

### Pipeline Reliability Scorecard

| Aspect | Score | Notes |
|--------|-------|-------|
| Retry and resilience | **1/10** | No retry logic; no dead letter queue |
| Error handling | **2/10** | Errors swallowed; empty JSON treated as success |
| Concurrency safety | **2/10** | Race conditions; instance variable bleed |
| Data integrity | **3/10** | `upsert_all` provides idempotency for metric points; everything else is unprotected |
| Observability | **2/10** | `Rails.logger.debug` only; no structured logging; no metrics |
| Scheduling | **3/10** | Schedule exists but uses `perform_now`; mislabeled entries |
| Performance | **3/10** | No batching; expensive recalculations; unbounded queries |
| Completeness | **4/10** | Core GET flow works; POST broken; raw cleanup absent |

### Rating: D- (Critical)

The pipeline functions under ideal conditions (all APIs responding, low volume, GET-only, single tenant) but will degrade under any real-world failure mode with no self-healing capability.

---

## 10. Consolidated Finding Register

### By Severity

#### Critical (10 findings) -- Blocks Production Deployment

| ID | Finding | Area | Impact |
|----|---------|------|--------|
| S-1 | Password verification disabled | Auth | Any email grants full access |
| S-2 | Hardcoded `admin`/`password123` on `/jobs` | Auth | Job dashboard exposed |
| S-3 | CORS configuration disabled | Config | Frontend connectivity broken or wide open |
| S-6 | `member?` not scoped to tenant | Tenancy | Cross-tenant data access |
| E-1 | Zero retry logic on all jobs | ETL | Data loss on transient failures |
| E-2 | `perform_now` blocks scheduler | ETL | Missed scheduled ingestions |
| E-3 | `call_via_post` / `call_via_execute` not implemented | ETL | Runtime crash for non-GET endpoints |
| E-4 | API errors silently return empty JSON | ETL | Invisible data loss |
| E-5 | No transaction wrapping on ingestion | ETL | Inconsistent database state |
| T-1 | ~3% effective test coverage | Quality | Zero safety net |

#### High (13 findings) -- Significant Risk

| ID | Finding | Area | Impact |
|----|---------|------|--------|
| S-4 | JWT tokens never expire | Auth | Permanent access on token theft |
| S-5 | Signing key defaults to empty string | Auth | Forgeable tokens on misconfiguration |
| S-7 | RBAC permissions never enforced | Auth | All members have equal access |
| S-8 | Pervasive `params.permit!` | Security | Mass assignment vulnerability |
| E-6 | Race conditions in alert creation | ETL | Duplicate alerts |
| E-7 | Instance variable bleed across iterations | ETL | Corrupted data records |
| E-8 | `alert_id` stores boolean, not FK | Data | Foreign key corruption |
| E-9 | HTTP timeout read from Alert table | ETL | Runtime crash if record absent |
| D-1 | Missing NOT NULL on ~20 tables | Schema | Invalid states allowed |
| D-2 | Missing indexes on polymorphic FKs | Schema | Slow queries at scale |
| D-3 | Plaintext credential storage | Schema | Credential exposure risk |
| D-4 | `status_id` getter triggers writes | Code | Side effects on read |
| D-5 | `rescue StandardError` in data source auth | Code | All errors silently swallowed |

#### Medium (16 findings) -- Quality Concerns

| ID | Finding | Area |
|----|---------|------|
| S-9 | No JWT algorithm restriction | Auth |
| S-10 | No rate limiting on any endpoint | Security |
| S-11 | No password complexity requirements | Auth |
| S-12 | `SECRET_KEY_BASE` used as API key | Auth |
| E-10 | Raw data trimming never scheduled | ETL |
| E-11 | Aggregation ignores `since` parameter | ETL |
| E-12 | Expensive indicator recalculation per ingestion | ETL |
| D-6 | Data migrations mixed with schema migrations | Schema |
| D-7 | `default_scope` overuse (5+ models) | Code |
| D-8 | Numeric values stored as VARCHAR | Schema |
| D-9 | Inconsistent time usage (`Time.now` vs `Time.zone.now` vs `DateTime.now`) | Code |
| D-10 | Bare factories with no attributes | Tests |
| D-11 | No queue separation (all jobs share 3 threads) | ETL |
| D-12 | Dual auth paths with separate signing keys | Auth |
| D-13 | Duplicate index on `tenants.identifier` | Schema |
| S-6a | Tenant scoping via params only | Tenancy |

#### Low (5 findings) -- Minor Cleanup

| ID | Finding | Area |
|----|---------|------|
| D-14 | Migration version mismatch (6.1 vs 8.0) | Schema |
| D-15 | Queue DB shares primary physical DB | Infra |
| D-16 | Dead code in `need_to_expand_arrays?` | Code |
| D-17 | Hardcoded `"2025-01-01"` default start date | Code |
| D-18 | `DataEndPointRaw.persistence` hardcodes `tenant_id: 1` | Code |

### By Category

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| Security / Auth | 3 | 4 | 4 | 0 | **11** |
| ETL / Jobs | 5 | 4 | 3 | 0 | **12** |
| Database / Schema | 0 | 3 | 4 | 2 | **9** |
| Code Quality | 1 | 2 | 4 | 3 | **10** |
| Testing | 1 | 0 | 1 | 0 | **2** |
| **Total** | **10** | **13** | **16** | **5** | **44** |

---

## 11. Root Cause Analysis

This section addresses [Project Evaluation Red Flag #1](../ewx-power-project-evaluation.md): *"Before committing to the build phase, we MUST understand what went wrong."*

### 11.1 Technical Competence

The evidence points to gaps in the previous team's Rails expertise:

| Evidence | What It Indicates |
|----------|-------------------|
| Disabled password verification with a confused developer comment (*"is it possibly password length?"*) | Inability to debug fundamental `bcrypt` / `has_secure_password` issues |
| Missing `call_via_post` / `call_via_execute` that would crash at runtime | Features declared but never completed or tested |
| `member?` not scoping to tenant | Misunderstanding of the multi-tenant security model they were building |
| `status_id` getters that write to the database | Anti-pattern that experienced Rails developers avoid |
| `params.permit!` in 16+ places | Unfamiliarity with Strong Parameters, a core Rails security feature |
| `perform_now` in recurring schedule | Misunderstanding of Solid Queue's scheduling model |

### 11.2 Process Discipline

| Evidence | What It Indicates |
|----------|-------------------|
| 17 test files for 350+ app files | No testing culture or test-first methodology |
| Both job specs are auto-generated placeholders | Tests were scaffolded but never implemented |
| Data migrations mixed with schema migrations | No migration discipline or review process |
| Disabled password check committed to main branch | No code review process -- this would be caught in any review |
| Hardcoded `admin`/`password123` in initializer | No security review or secrets management |

### 11.3 Delivery Pattern

The codebase has **broad coverage** -- 350+ files spanning the full domain model. This indicates the team prioritized **surface-level feature delivery** (building endpoints and screens) over **correctness** (testing, security, reliability). The result is a platform that appears complete at first glance but is structurally unsound.

### Conclusion

The previous team's failure was primarily a **technical competence problem** compounded by **absent process discipline**. This is relevant for the keep-vs-rebuild decision: the patterns and decisions of the original team are embedded throughout the codebase, making incremental remediation unreliable.

---

## 12. Keep vs. Rebuild Inputs

### What Has Value

| Asset | Reuse Value | Notes |
|-------|-------------|-------|
| Domain model (79 models, associations, structure) | **High** | Reasonable modeling of the EWX domain; portable to any framework |
| Database schema (63 tables) | **Medium-High** | Sound table design; needs constraint hardening |
| Blueprint serialization (73 files) | **Medium** | Clean pattern; Rails-specific |
| Route structure | **Medium** | RESTful, well-organized; Rails-specific |
| Service pattern (structural) | **Low-Medium** | Architecture is fine; most implementations are trivial wrappers |

### What Must Be Rebuilt or Remediated

| Component | Rebuild Effort | Justification |
|-----------|---------------|---------------|
| Authentication | High | Broken at multiple levels; needs full redesign |
| Multi-tenant authorization | High | Core isolation logic is wrong |
| ETL pipeline | High | 12 findings across reliability, integrity, and performance |
| Test suite | High | Must be written from scratch for ~350 files |
| Security layer | High | `params.permit!`, CORS, rate limiting, JWT expiration |
| Background job infrastructure | Medium | Retry logic, queue separation, scheduler configuration |
| Database constraints | Medium | NOT NULL, FKs, indexes across 30+ tables |

### Cost Comparison

| Approach | Estimated Hours | Risk Profile |
|----------|----------------|--------------|
| **Remediate existing codebase** | 270-370h | Higher -- preserves architectural decisions that produced 44 findings |
| **Full rebuild** (per project evaluation) | 640-800h | Lower -- clean foundation with proper testing, security, reliability |
| **Hybrid** (keep CRUD, rebuild ETL) | 400-500h | Medium -- partial reuse, new reliability for data pipeline |

#### Remediation Breakdown

| Work Package | Hours | Priority |
|--------------|-------|----------|
| Security remediation (S-1 through S-12) | 40-60h | Immediate |
| ETL pipeline overhaul (E-1 through E-12) | 60-80h | Immediate |
| Database constraint hardening (D-1 through D-3) | 20-30h | Sprint 1 |
| Test suite creation (~350 files) | 120-160h | Ongoing |
| Code quality fixes (D-4 through D-13) | 30-40h | Sprint 2-3 |
| **Total** | **270-370h** | -- |

### Decision Factors

**Favoring rebuild:**

1. Remediation cost (270-370h) is **35-45% of a full rebuild** (640-800h), but carries higher risk -- it preserves the architectural decisions that produced the problems
2. Security foundation is **systemically compromised**, not a single-bug fix
3. Test coverage is effectively zero -- building confidence in existing code may cost more than rewriting with tests
4. ETL pipeline needs fundamental reliability work, not patches
5. The domain model (the most valuable asset) can be preserved regardless of framework

**Favoring keep + remediate:**

1. Broad feature coverage already exists -- 350+ files spanning the full domain
2. Domain model and table structure are reasonable
3. Client's tech stack (Rails + NextJS) is established -- switching adds migration risk
4. Some patterns are solid -- service objects, blueprints, input objects
5. Lower total hours if remediation succeeds

---

## 13. Recommendations

### Immediate Actions (If Any Traffic Is Planned)

These 5 fixes address the most dangerous vulnerabilities and should be applied regardless of the keep-vs-rebuild decision:

1. **Re-enable password verification** in `app/services/users/contexts/authentication.rb`
2. **Fix `member?` tenant scoping** in `app/models/concerns/tenants/role_concerns.rb` -- add `tenant_id: id` to the query
3. **Remove hardcoded credentials** from `config/initializers/mission_control_jobs.rb`
4. **Add JWT expiration** claims in `lib/encryption/token_manager.rb`
5. **Configure CORS** with explicit allowed origins in `config/initializers/cors.rb`

### Strategic Recommendation

**Lean toward rebuild with domain model preservation.**

The remediation cost (270-370h) is high enough relative to a full rebuild (640-800h) that starting fresh -- with proper testing, security, and reliability built in from the foundation -- is likely more cost-effective and lower risk than attempting to harden an application with 44 findings across every layer.

The previous team's patterns are embedded throughout the codebase. Piecemeal fixes address individual symptoms but don't resolve the systemic absence of testing, security review, and reliability engineering that produced them.

The **domain model** (table structure, associations, entity relationships) is the most valuable asset and should be preserved and migrated regardless of the framework decision. This is covered in detail in the [Keep vs. Rebuild ADR](./platform-rebuild-adr-keep-vs-rebuild.md) (forthcoming).

---

*DevSavant Product Engineering -- Phase 1 Discovery*
*EWX Power Platform Rebuild*
