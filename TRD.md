# Technical Requirements (TRD)

**Status:** 📝 Draft. Settle each 📝 and ❓ with the team; when everything is ✅, this status flips.

The technical requirements every CamFinSight repo builds to. Feature behaviour lives in [`specs/`](specs/README.md), fields in the [data dictionary](data-dictionary/README.md).

**Legend:** ✅ settled · 📝 proposed · ❓ open

## 1. System overview

### 1a. Parts and data ownership

Each table is written by exactly one service. Other services read it through that service's API or events.

| Part | Repo | Does | Owns tables | Status |
|---|---|---|---|---|
| Identity service + API Gateway | `ledgervision-identity-service` | Auth, users, businesses, branches, managers, invitations, notifications (email/Telegram), audit log; also the shared infra and the API Gateway (:8080) | Schema `identity` (exact tables: data dictionary) | 📝 |
| Sales service | `ledgervision-sales-service` | Items, categories, units, stock, sales, customers, invoices, sales overview | Schema `sales` | 📝 |
| Finance service | `ledgervision-finance-service` | Money accounts, transaction ledger, bank statements, reconciliation, supplier bills, payments, dashboard figures, monthly AI report, Shopify/WeBill365 integrations | Schema `finance` | 📝 |
| AI & Import service | `ledgervision-ai-service` | AI advisor, forecasting, data import | Schema `ai` | 📝 |
| Web UI | `ledgervision-web-ui` | The web front end; its Next.js server is the BFF (holds the session, calls the gateway) | None | 📝 |
| Android app | `ledgervision-android-ui` | Mobile app; calls the API Gateway with the user's token | None | 📝 |
| iOS app | `ledgervision-ios-ui` | Mobile app (iOS) | None | ❓ |

## 2. Technology stack

| Area | Choice | Status |
|---|---|---|
| Web | Next.js 16, React 19, TypeScript, Tailwind v4, shadcn on base-ui, pnpm | 📝 |
| Android | Kotlin, Jetpack Compose, Hilt, Retrofit, Clean Architecture + MVVM | 📝 |
| Backend services | Spring Boot 3.5, Java 21, Spring Cloud Gateway, OpenFeign | 📝 |
| AI service | Python, FastAPI | 📝 |
| Database | PostgreSQL, one database `camfinsight_db`, one schema per service, Flyway migrations | 📝 |
| Identity provider | Keycloak (realm `camfinsight`), Google sign-in | 📝 |
| Messaging | RabbitMQ | 📝 |
| Cache | Redis | 📝 |
| Config and discovery | Consul (KV config, service discovery) | 📝 |
| File storage | S3-compatible (RustFS locally) | 📝 |
| Tracing | Zipkin | 📝 |

## 3. API

| Requirement | Status |
|---|---|
| REST with JSON bodies; every path starts with `/api/v1` | 📝 |
| Field names in payloads are the dictionary names (snake_case) | 📝 |
| One response envelope for every endpoint: `{ success, code, message, data, timestamp }` (see [`api/README.md`](api/README.md)) | 📝 |
| Every request is scoped on the server to what the caller may see; scope is never trusted from the client | 📝 |
| Pagination, filter and sort parameters | ❓ |

## 4. Security

| Requirement | Status |
|---|---|
| Sign-in method and identity provider | ❓ |
| Role and scope come from the user's record on the server, never from token claims or the request | 📝 |
| Secrets only in environment variables, listed in `.env.example`; never committed | 📝 |

## 5. Data

| Requirement | Status |
|---|---|
| Database and how schemas are split | ❓ |
| Primary keys `uuid`; timestamps stored in UTC; money as `numeric` (never float) | 📝 |
| Schema changes only through migrations; an applied migration is never edited | 📝 |

## 6. Background jobs

| Job | Schedule | Owner | Status |
|---|---|---|---|

## 7. Non-functional requirements

| Requirement | Status |
|---|---|
| Supported browsers / devices | ❓ |
| Accessibility target | ❓ |
| Languages | ❓ |

## 8. Delivery and environments

| Requirement | Status |
|---|---|
| CI runs the build and all tests on every pull request; a failing build can't merge | 📝 |
| Hosting | ❓ |

## 9. Open questions

Numbered; answered ones keep their number.
