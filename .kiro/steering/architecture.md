---
inclusion: always
---

# Architecture

## Overview

This project demonstrates AWS Blocks — an open-source TypeScript framework (currently in Preview) that bundles application code, local dev infrastructure, and AWS cloud infrastructure into composable npm packages called Blocks.

**Target audience**: A team of 300 developers who currently build serverless microservices using raw AWS CDK with Lambda, API Gateway, AppSync, and DynamoDB. Every demo must make the contrast between the existing approach and AWS Blocks explicit and tangible.

**Two modes of the same codebase:**
- **Local**: Runs a fully functional backend using local implementations (Postgres, local auth, local messaging) — no AWS account required
- **Deployed**: The identical code deploys to production AWS services (Lambda, DynamoDB, Cognito, etc.) without modification

---

## AWS Blocks vs Current Team Stack

This table is the core reference for every demo. Each scenario should make the trade-offs in this table visible:

| Concern | Current Team Approach | AWS Blocks Approach |
|---|---|---|
| Compute | Lambda functions (manual CDK) | Managed by Blocks runtime — no Lambda config needed |
| API layer | API Gateway (HTTP/REST) + manual CDK | Type-safe API Block — client types auto-generated |
| Auth | Cognito + custom Lambda authorizer CDK | Authentication Block — local + Cognito on deploy |
| Data store | DynamoDB single-table (manual CDK + SDK) | Data persistence Block — local Postgres + DynamoDB on deploy |
| Real-time | AppSync subscriptions + CDK | Real-time messaging Block |
| File storage | S3 + presigned URLs (manual CDK) | File storage Block |
| Background jobs | SQS + Lambda trigger (manual CDK) | Background jobs Block |
| Local dev | SAM CLI or mocked AWS services | Native local runtime — no AWS account required |
| Type safety | Manual interface definitions | End-to-end generated types from Block definitions |
| Infrastructure | Explicit CDK constructs per service | Managed by Blocks; CDK available as escape hatch |

---

## Demo Scenarios

Each demo lives in `blocks/<scenario>/` and must be independently runnable. Every scenario is paired with a `comparison/` equivalent showing the raw CDK/Lambda approach for direct contrast.

| Scenario | Block Used | Replaces |
|---|---|---|
| `auth/` | Authentication Block | Cognito + Lambda authorizer + CDK |
| `data/` | Data persistence Block | DynamoDB + SDK + single-table CDK |
| `api/` | Type-safe API Block | API Gateway + manual OpenAPI + CDK |
| `realtime/` | Real-time messaging Block | AppSync subscriptions + CDK |
| `storage/` | File storage Block | S3 + presigned URL Lambda + CDK |
| `background-jobs/` | Background jobs Block | SQS + Lambda trigger + CDK |

---

## AWS Blocks Architecture Principles

### Blocks are the primary building unit
- Each Block is an npm package (`@aws-blocks/*`) that provides a backend capability
- A Block bundles: runtime code, local dev implementation, and AWS CDK infrastructure
- Instantiate a Block in TypeScript — it provisions everything automatically

### Local-first development
- All demos must run with `npm run dev` before any AWS deployment is attempted
- Local execution requires no AWS credentials, no CDK bootstrap, no SAM CLI
- If a demo cannot run locally, it is incomplete

### Same code, two environments
- Application code must not branch on `process.env.NODE_ENV` or similar to switch between local and AWS behaviour — Blocks handle this internally
- Do not introduce local-only workarounds or AWS-only code paths in the same module

### CDK escape hatch
- AWS CDK is available when Block defaults are insufficient
- Escape hatch code lives in `infra/lib/` and must be clearly commented explaining why the Block default was overridden
- At least one demo must demonstrate a CDK escape hatch — this directly addresses the team's existing CDK expertise

### Type safety is a first-class feature
- Always use the generated TypeScript client from Block definitions — never write raw fetch/axios calls to Block-managed APIs
- Type-safe client usage is one of the strongest differentiators for this audience; make it visible in every demo

---

## Project Structure

```
.
├── blocks/                    # One subfolder per AWS Blocks demo scenario
│   ├── auth/                  # Authentication Block demo
│   ├── data/                  # Data persistence Block demo
│   ├── api/                   # Type-safe API Block demo
│   ├── realtime/              # Real-time messaging Block demo
│   ├── storage/               # File storage Block demo
│   └── background-jobs/       # Background jobs Block demo
├── comparison/                # Side-by-side: raw CDK/Lambda vs AWS Blocks
│   ├── auth/                  # Cognito + Lambda authorizer (current approach)
│   ├── data/                  # DynamoDB + SDK (current approach)
│   └── api/                   # API Gateway + CDK (current approach)
├── infra/                     # CDK escape-hatch overrides only
│   ├── bin/                   # CDK app entry point
│   └── lib/                   # Custom stacks/constructs not covered by Blocks
├── shared/
│   ├── models/                # Shared TypeScript types and interfaces
│   └── utils/                 # Pure, reusable helpers
├── test/                      # Tests mirroring blocks/ structure
└── scripts/                   # Setup, demo runner, and teardown scripts
```

---

## Comparison Demo Design Rules

The `comparison/` folder is as important as `blocks/` — it is what makes the value proposition legible to the audience.

- Each comparison demo must be functionally equivalent to its Blocks counterpart — same feature, different implementation
- Highlight lines of code, number of files, and manual steps required — these are the productivity metrics the audience cares about
- Include a `README.md` in each comparison scenario with a side-by-side summary table: lines of code, files created, AWS services manually configured, local dev complexity
- Do not make the comparison demos intentionally poor — write them as a competent developer on the current stack would

---

## Observability (POC Level)

- Use `@aws-lambda-powertools/logger` in comparison demos where Lambda is involved
- AWS Blocks manages its own observability for Block-managed compute — do not add separate CloudWatch config for Block internals
- X-Ray tracing via CDK escape hatch only if explicitly needed for a demo point
- Follow `poc-cost-guardrails.md` for log retention (1 day) and alarm restrictions
