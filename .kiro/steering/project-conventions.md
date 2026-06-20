---
inclusion: always
---

# Project Conventions

## Project Purpose

This project is a **demonstration and awareness showcase** of [AWS Blocks](https://docs.aws.amazon.com/blocks/latest/devguide/what-is-blocks.html) (currently in Preview), targeted at a team of 300 developers who currently build on Lambda, API Gateway, AppSync, and DynamoDB using raw AWS CDK.

**Goal**: Show concretely what AWS Blocks offers compared to the team's existing approach, so each team can independently assess where AWS Blocks fits into their own modules. This is exploratory and educational — not a production system.

**What AWS Blocks is**: A backend toolkit where each Block is an npm package that bundles application code, local dev infrastructure, and AWS cloud infrastructure together. Blocks compose freely. The same code runs locally (no AWS account needed) and deploys to AWS unchanged. AWS CDK is available as an escape hatch for direct resource configuration at any point.

**Available Block capabilities** (as of Preview): authentication, data persistence, type-safe APIs, real-time messaging, file storage, background jobs.

**Audience assumptions**: Developers are comfortable with TypeScript, CDK, Lambda, DynamoDB, AppSync, and API Gateway. Demos should directly contrast the AWS Blocks approach against the raw CDK/Lambda patterns they already know.

---

## Language & Runtime

- **Language**: TypeScript with `strict: true` — never disable or relax strict flags
- **Runtime**: Node.js 24.x (Active LTS — supported until April 2028)
- **Package manager**: npm
- **TypeScript version**: 5.x

---

## Code Style

- Prefer `const` over `let`; never use `var`
- Use `async/await` — never raw Promise chains or callbacks
- Use named exports; avoid default exports
- Use early returns to flatten nesting rather than deeply nested `if/else`
- Keep functions single-purpose; aim for ≤ 50 lines per function
- Do not use `any` — type-narrow explicitly instead

---

## File & Symbol Naming

| Concept | Convention | Example |
|---|---|---|
| Source files (modules) | `kebab-case.ts` | `todo-service.ts` |
| Source files (classes/constructs) | `PascalCase.ts` | `TodoStack.ts` |
| Variables & functions | `camelCase` | `createTodoItem` |
| Classes & interfaces | `PascalCase` | `TodoService`, `ITodoRepository` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Enum members | `PascalCase` | `Status.Active` |
| CDK constructs & stacks | `PascalCase` | `TodoAppStack` |
| Environment variables | `UPPER_SNAKE_CASE` | `TABLE_NAME` |

---

## Project Structure

```
.
├── blocks/                  # One subfolder per AWS Blocks demo scenario
│   ├── auth/                # Authentication Block demo
│   ├── data/                # Data persistence Block demo
│   ├── api/                 # Type-safe API Block demo
│   ├── realtime/            # Real-time messaging Block demo
│   ├── storage/             # File storage Block demo
│   └── background-jobs/     # Background jobs Block demo
├── reference-implementation/  # Side-by-side: raw CDK/Lambda vs AWS Blocks
├── infra/                   # Any CDK escape-hatch overrides
│   ├── bin/                 # CDK app entry point
│   └── lib/                 # Stacks and constructs
├── shared/
│   ├── models/              # Shared types and interfaces
│   └── utils/               # Pure, reusable helpers
├── test/                    # Tests mirroring blocks/ structure
├── scripts/                 # Setup and demo runner scripts
└── .kiro/                   # Kiro steering and specs
```

**Structure intent**: Each demo in `blocks/` should be self-contained and runnable independently. The `reference-implementation/` folder exists specifically to show the before/after contrast for the target audience.

---

## AWS Blocks Usage Patterns

- Each Block demo must be runnable locally with `npm run dev` — no AWS account required for local execution
- The same demo code must deploy to AWS without modification — do not introduce local-only workarounds
- When reaching for CDK directly (escape hatch), isolate it in `infra/lib/` and document clearly why the Block default was insufficient
- Do not mix raw AWS SDK calls and Block APIs for the same capability — pick one per feature boundary
- Always demonstrate type-safe client usage; the end-to-end TypeScript type safety is a key selling point for this audience

---

## Demonstration Design Principles

- **Each demo must have a clear "so what"** — the README or inline comments must explain what problem the Block solves compared to the team's current Lambda/CDK approach
- **Favour completeness over cleverness** — the audience needs to understand the pattern, not be impressed by the implementation
- **Show the local dev story explicitly** — include `npm run dev` instructions; the no-AWS-account local workflow is one of the strongest differentiators
- **Highlight CDK escape hatch usage** — at least one demo should show how to drop into CDK for customisation, since this directly addresses the team's existing CDK expertise
- **Keep demos independent** — avoid shared state between Block demos so each team can lift and study one scenario in isolation

---

## Error Handling

- Type-narrow all caught errors before use: `if (error instanceof Error) { ... }`
- Never silently swallow errors — log before re-throwing or returning a typed error response
- Lambda handlers (used in reference implementation demos) must return structured HTTP responses — never propagate unhandled rejections
- Pick one error strategy per module (typed throws **or** result objects) and stay consistent within it

---

## Logging

- Use `@aws-lambda-powertools/logger` for structured JSON output where Lambda is involved
- Always include `correlationId` / `requestId` in log entries
- Log levels: `DEBUG` locally, `INFO` in production deploys, `ERROR` for failures
- Never log PII, secrets, tokens, or any sensitive data

---

## Testing

- Mirror `blocks/` structure under `test/` (e.g., `test/auth/`, `test/data/`)
- Use Jest as the test runner
- Unit tests must not make real AWS calls — mock at the repository/Block boundary
- CDK stack tests use `aws-cdk-lib/assertions`
- Demos are illustrative, but critical business logic in shared utilities should have ≥ 80% line coverage

---

## Dependencies

- Pin exact versions in `package.json` — no `^` or `~` on production dependencies
- Use `devDependencies` for all build-only and test-only packages
- Prefer AWS-maintained packages (`@aws-blocks/*`, `@aws-sdk/*`, `@aws-lambda-powertools/*`)
- For Lambda reference implementation demos: import only the specific AWS SDK v3 client needed — never the full `aws-sdk` v2
- Minimize Lambda bundle size in reference implementation demos to keep the contrast fair

---

## CDK Conventions (Escape Hatch Usage)

- Use CDK escape hatch only when a Block's default configuration is genuinely insufficient for the demo point being made
- One stack per deployment boundary; keep stacks focused
- No hardcoded ARNs, account IDs, or region strings — use CDK context or environment variables
- Tag all resources with at minimum: `Project: aws-blocks-demo`, `Environment`, and `Owner`
- Use `RemovalPolicy.DESTROY` for demo stacks (these are not production resources)
