# Kuwenta – Tech Stack

| Field | Value |
|---|---|
| Project | Kuwenta – personal finance and expense tracker |
| Document | Tech stack: languages, frameworks, libraries, infrastructure and tooling |
| Version | 1.0 (Draft) |
| Date | 11 October 2026 |
| Status | Draft – for review |
| Related | kuwenta-requirements.md, kuwenta-external-apis.md, kuwenta-aws-prd.drawio.svg |

## 1. Purpose

This document lists what Kuwenta is built with and why. Each choice is marked **Decided** when it comes from an architecture decision in the requirements (AD-xx, MNT-xx), or **Proposed** when it is new here and open for review. Exact package versions are pinned in `package.json` and Terraform lock files when the repository is created; this document names major versions only where they matter.

## 2. Principles

- **One language end to end.** TypeScript in the browser, the Lambdas and the shared packages, so types are written once (MNT-01).
- **Serverless and scale-to-zero.** Nothing runs or costs money while nobody uses the app (CST-02).
- **Code for fixed formats, AI for varied ones.** Deterministic parsers for statements; Gemini only for merchant emails and categorization (AD-18).
- **Same code locally and in AWS.** Local development needs no AWS account (MNT-04).
- **Few, well-known dependencies.** Prefer official SDKs and widely used libraries over niche ones.

## 3. Summary

| Layer | Choice | Status |
|---|---|---|
| Language | TypeScript (strict mode) | Decided (MNT-01) |
| Runtime | Node.js 24 LTS; AWS Lambda `nodejs24.x` on arm64 | Decided (MNT-01); arm64 Proposed |
| Web app | React + Vite, responsive, PWA later | Decided (CLI-01, AD-12) |
| API | Amazon API Gateway HTTP API + Lambda, Hono as the router inside each Lambda | API Gateway Decided (AD-07); Hono Proposed |
| Auth | Amazon Cognito managed login (OIDC, authorization code + PKCE) | Decided (AUTH-06, AD-13) |
| Database | Aurora PostgreSQL Serverless v2 (16.3+) via RDS Data API | Decided (AD-02, AD-03) |
| ORM / migrations | Drizzle ORM + Drizzle Kit | Decided (AD-17) |
| Key-value store | Amazon DynamoDB (provisioned, free tier) | Decided (AD-05, CST-05) |
| AI | Google Gemini API, gemini-3.5-flash-lite | Decided (AD-10) |
| PDF reading | pdf.js (`pdfjs-dist`) in the browser | Decided (AD-16) |
| Infrastructure as code | Terraform, separate state per environment | Decided (MNT-02) |
| Monorepo | pnpm workspaces | Proposed |
| Testing | Vitest; Playwright later | Proposed |
| CI/CD | GitHub Actions with AWS OIDC roles | Proposed |

## 4. Frontend (web app)

| Concern | Choice | Status | Why |
|---|---|---|---|
| Framework | React | Decided | CLI-01. |
| Build tool | Vite | Decided | CLI-01; fast dev server, static build for S3 + CloudFront. |
| Routing | React Router | Proposed | Mature, simple, no server needed for a static SPA. |
| Server data | TanStack Query | Proposed | Caching, retries and loading states; handles the "waking up the database" retry (DSH-03) cleanly. |
| Forms and validation | React Hook Form + Zod | Proposed | Zod schemas are shared with the backend (`packages/shared`), so the browser and API validate the same way. |
| Styling and components | Tailwind CSS + shadcn/ui | Proposed | Phone-width layouts (CLI-01) with accessible, copy-in components and no heavy UI framework. |
| Charts | Recharts | Proposed | Simple React charts for monthly totals by category and account (DSH-01). |
| Auth client | `react-oidc-context` (on `oidc-client-ts`) | Proposed | Standard OIDC client for Cognito managed login with PKCE; lighter than AWS Amplify. |
| Statement PDFs | `pdfjs-dist` | Decided | Opens password-protected PDFs and extracts text in the browser (IMP-02, AD-16). |
| Statement parsers | Plain TypeScript modules in `packages/parsers` | Decided | One deterministic parser per statement format, starting with Maya Wallet (AD-18); unit-tested against sanitized fixtures. |

## 5. Backend (Lambdas)

| Concern | Choice | Status | Why |
|---|---|---|---|
| Runtime | `nodejs24.x`, arm64 | Proposed (arm64) | Node.js 24 is supported on Lambda until April 2028; arm64 (Graviton) is cheaper per millisecond than x86. |
| Functions | Post-confirmation, Pre-sign-up, Gmail connect, Reconcile, Import, Expenses | Decided | Matches the architecture diagram; Import is separate so only Reconcile can decrypt Gmail tokens (AD-19, SEC-03). |
| Router | Hono (`hono/aws-lambda` adapter) | Proposed | The same app runs on Lambda and as a local Node server, so local development needs no SAM or emulator. |
| Validation | Zod | Proposed | Shared schemas with the frontend; also validates Gemini output (REC-11). |
| Database access | Drizzle ORM: `aws-data-api/pg` driver in AWS, `node-postgres` locally | Decided | AD-17, MNT-04. Row-level security is set with `SET LOCAL` inside each Data API transaction (SEC-02). |
| AWS SDK | AWS SDK for JavaScript v3 (DynamoDB Document Client, KMS, SSM, RDS Data) | Decided | Bundled with each function rather than using the runtime's copy, so versions are pinned. |
| Lambda utilities | Powertools for AWS Lambda (TypeScript): Logger, Parameters, Idempotency | Proposed | Logger supports field redaction (PRV-04); Parameters caches SSM secrets between invocations (SEC-09); Idempotency backs the one-time submission ID for manual entries (EXP-09). |
| Gmail client | `@googleapis/gmail` + `google-auth-library` | Proposed | Official Google packages, smaller than the full `googleapis` bundle. |
| Email text | `html-to-text` | Proposed | Converts HTML-only receipts to text while keeping table rows on their own lines. |
| Gemini client | `@google/genai` (Google Gen AI SDK) | Proposed | Official SDK; supports JSON-schema structured output. |
| Bundling | esbuild (one minified bundle per function) | Proposed | Small zip files and fast cold starts. |

## 6. Data

| Store | Holds | Status |
|---|---|---|
| Aurora PostgreSQL Serverless v2 (16.3+), min 0 ACU | Accounts, aliases, categories, transactions, transfers, statement uploads, FX rates | Decided (AD-02) |
| DynamoDB | Users, settings, sender list, Gmail cursor and processed message IDs, encrypted tokens, OAuth state (TTL), pending items (TTL 7 d), raw Gemini output (TTL 60 d) | Decided (AD-05) |
| Secrets Manager | Aurora master and app-user credentials | Decided (AD-09) |
| SSM Parameter Store (SecureString) | Gemini API key, Google OAuth client secret, CloudFront origin secret | Decided (AD-09, SEC-09) |
| KMS | One key per environment for Gmail refresh tokens | Decided (AD-08) |

Money is stored as `NUMERIC(14,2)` with a currency code; monthly totals are calculated in the Asia/Manila time zone.

## 7. Infrastructure

| Concern | Choice | Status |
|---|---|---|
| Cloud | AWS, ap-southeast-1 (WAF web ACL in us-east-1) | Decided (PRF-03, C-02) |
| Accounts | Separate AWS accounts for staging and prd | Decided (AD-11) |
| IaC | Terraform 1.10 or later with the AWS provider; S3 remote state with native lock files, one state per environment | Decided (MNT-02); S3 native locking Proposed |
| Edge | CloudFront (default domain) + S3 static hosting + WAF on prd | Decided (AD-07) |
| Observability | CloudWatch Logs (30-day retention), AWS Budgets alert at $15 | Decided (MNT-06, CST-03) |

## 8. Local development

| Concern | Choice | Status |
|---|---|---|
| Node.js | 24 LTS, managed with fnm (already installed) | Proposed |
| Package manager | pnpm workspaces | Proposed |
| Services | Docker Compose: `postgres:16` and `amazon/dynamodb-local` | Decided (MNT-04) |
| API | Hono app run with `@hono/node-server`, calling local Postgres and DynamoDB Local | Proposed |
| External APIs | Real Gmail (dummy account) and Gemini free tier; sanitized fixtures for automated tests | Decided (PRV-01, PRV-02) |

## 9. Quality and delivery

| Concern | Choice | Status |
|---|---|---|
| Unit tests | Vitest (parsers, redaction, matching, validation) | Proposed |
| AWS mocks | `aws-sdk-client-mock` | Proposed |
| Integration tests | Run against staging for Data API behavior | Decided (MNT-05) |
| End-to-end tests | Playwright | Proposed (Later) |
| Lint and format | ESLint + Prettier | Proposed |
| CI/CD | GitHub Actions: test on every push; deploy to staging on merge; deploy to prd on a manual approval. AWS access through OIDC roles, no long-lived keys | Proposed |

## 10. Repository layout

```
kuwenta/
├─ apps/
│  └─ web/                 React + Vite app
├─ services/
│  └─ api/                 Lambda handlers (Hono), one entry point per function
├─ packages/
│  ├─ shared/              Zod schemas, types, Drizzle schema
│  ├─ parsers/             Deterministic statement parsers (Maya first)
│  └─ redaction/           Account, card and mobile number redaction
├─ infra/
│  ├─ modules/             Terraform modules
│  └─ envs/                staging/ and prd/ (separate state)
├─ fixtures/               Sanitized receipts and statements for tests
├─ docs/                   Requirements, diagrams, this document
└─ docker-compose.yml      Postgres 16 + DynamoDB Local
```

`docs/sample-receipts/` (real data) stays out of git; tests use `fixtures/` only (R-11).

## 11. Open points

| ID | Question |
|---|---|
| TS-01 | Confirm the Proposed choices above, especially Hono, Tailwind CSS + shadcn/ui, pnpm and GitHub Actions. |
| TS-02 | Where will the repository be hosted (GitHub assumed for CI/CD)? |
