---
inclusion: always
---

# Build & Workflow

## Overview

This document defines how to build, test, lint, and deploy the project. When verifying code changes, always run the relevant commands below. Never present a change as complete without confirming it compiles and passes linting/tests.

## Key Commands

| Task | Command |
|---|---|
| Install dependencies | `npm install` |
| Start local dev (all Blocks) | `npm run dev` |
| Compile TypeScript | `npm run build` |
| Lint | `npm run lint` |
| Lint (auto-fix) | `npm run lint:fix` |
| Run all tests | `npm test` |
| Run tests (single run, no watch) | `npm test -- --runInBand` |
| Run tests with coverage | `npm run test:coverage` |
| CDK synth (escape hatch / reference implementation stacks) | `npm run cdk:synth` |
| CDK diff | `npm run cdk:diff` |
| CDK deploy | `npm run cdk:deploy -- --context env=dev` |
| Destroy POC stacks | `npm run cdk:destroy -- --context env=dev` |

## npm Scripts (expected in package.json)

```json
{
  "scripts": {
    "dev": "blocks dev",
    "build": "tsc --project tsconfig.json",
    "lint": "eslint 'blocks/**/*.ts' 'reference-implementation/**/*.ts' 'infra/**/*.ts' 'shared/**/*.ts' 'test/**/*.ts'",
    "lint:fix": "eslint 'blocks/**/*.ts' 'reference-implementation/**/*.ts' 'infra/**/*.ts' 'shared/**/*.ts' 'test/**/*.ts' --fix",
    "test": "jest --runInBand",
    "test:coverage": "jest --runInBand --coverage",
    "audit": "npm audit --audit-level=high",
    "cdk:synth": "cdk synth",
    "cdk:diff": "cdk diff",
    "cdk:deploy": "cdk deploy --all --require-approval never",
    "cdk:destroy": "cdk destroy --all --force"
  }
}
```

## TypeScript Configuration

Two tsconfig files are required:

**`tsconfig.json`** — production source compilation covering all project source folders:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["blocks", "reference-implementation", "infra", "shared"],
  "exclude": ["node_modules", "dist", "test", ".kiro"]
}
```

Note: there is no `src/` directory in this project. Source files live in `blocks/`, `reference-implementation/`, `infra/`, and `shared/`. Never add a `rootDir` constraint — the multi-folder layout makes a single `rootDir` impractical.

**`tsconfig.test.json`** — extends base, adds test files:
```json
{
  "extends": "./tsconfig.json",
  "include": ["blocks", "reference-implementation", "infra", "shared", "test"],
  "exclude": ["node_modules", "dist", ".kiro"]
}
```

## Linting

Use ESLint with the TypeScript plugin. Required rules:
- `@typescript-eslint/no-explicit-any`: error
- `@typescript-eslint/no-unused-vars`: error
- `@typescript-eslint/explicit-function-return-type`: warn
- `no-console`: warn (use the logger instead)

## Testing

- Framework: **Jest** with `ts-jest`
- Test files: `test/**/*.test.ts`, mirroring `blocks/` and `reference-implementation/` structure
- Block-level tests must not make real AWS calls — mock at the Block boundary
- Reference implementation tests must not make real AWS calls — mock at the repository/SDK boundary
- CDK stack tests (for escape hatch and reference implementation stacks) use `aws-cdk-lib/assertions`
- Shared utilities in `shared/utils/` must have ≥ 80% line coverage — these are the only modules with a hard coverage requirement in a POC

### Test Coverage Guardrails

Coverage is a ratchet — it only moves up, never down. Apply these rules without exception:

**Deleting test cases is not allowed.**
- Do not remove `it()`, `test()`, or `describe()` blocks from any test file
- Do not comment out test cases to fix a failing build — fix the implementation or the test assertion instead
- If a feature is removed, the corresponding test file may be deleted **only** if the entire source module it covers is also deleted in the same change

**Skipping tests is not allowed in CI.**
- Do not use `it.skip()`, `test.skip()`, `xit()`, or `xdescribe()` except temporarily during local development
- `.skip` must never appear in a committed test file — treat it as a lint error
- Add the following ESLint rule to catch skipped tests at commit time:
  ```json
  "jest/no-disabled-tests": "error"
  ```

**Coverage must not decrease.**
- Before committing changes to `blocks/` or `shared/`, run `npm run test:coverage` and confirm the coverage report shows no regression
- If a PR reduces line coverage below the current baseline, it must not be merged until tests are added to compensate
- Coverage thresholds enforced in `jest.config.ts`:
  ```typescript
  coverageThreshold: {
    global: {
      lines: 80,
      functions: 80,
      branches: 70,
      statements: 80,
    },
    './shared/utils/': { lines: 85 },  // shared utilities are the only hard coverage target
  }
  ```

**Adding new source code requires new tests.**
- Every new function or method in `shared/utils/` must have at least one corresponding test
- New Lambda handlers in `reference-implementation/` must have at least: one happy-path test, one validation-failure test, and one error-handling test
- CDK stacks must have at least one `aws-cdk-lib/assertions` snapshot or fine-grained assertion test

## CDK Workflow

1. **Synth first** — always run `cdk synth` before deploying to catch template errors
2. **Diff before deploy** — run `cdk diff` to review infra changes before applying
3. **Bootstrap once per account/region** — `cdk bootstrap aws://<account>/<region>`
4. Pass the target environment via CDK context: `--context env=dev|staging|prod`
5. Never run `cdk deploy` against production without a prior `cdk diff` review

## Verification Checklist

Before marking any task complete, confirm:
- [ ] `npm run build` exits with no errors
- [ ] `npm run lint` exits with no errors
- [ ] `npm test` passes with no failures
- [ ] If infra was changed: `npm run cdk:synth` exits with no errors

## Build Exclusions

The `.kiro/` directory contains Kiro tooling metadata (steering files, specs, settings). It must be excluded from all build, lint, and test tooling — similar to `.git/`.

Apply these exclusions when scaffolding config files:

- **`tsconfig.json`** — add `.kiro` to the `exclude` array
- **`tsconfig.test.json`** — add `.kiro` to the `exclude` array
- **`.eslintignore`** — add `.kiro/`
- **`jest.config.ts`** — add `'/.kiro/'` to `testPathIgnorePatterns`

**`.gitignore` note**: Do **not** ignore `.kiro/` itself — steering files and specs are team artifacts and must be committed. Only ignore generated or sensitive content within it:
```
.kiro/settings/
```

## Local Development

- Use `.env.local` for local environment variables — never commit this file
- Add `.env.local` to `.gitignore`
- Use `dotenv` in test setup to load local env vars; never in production Lambda code
- For local Lambda testing, use AWS SAM CLI (`sam local invoke`) or a test event in the test suite
