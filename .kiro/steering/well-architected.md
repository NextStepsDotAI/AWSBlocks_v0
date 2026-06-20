---
inclusion: always
---

# Well-Architected Guidelines

This project follows the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/) across all six pillars. Apply these rules alongside the patterns defined in `infra-patterns.md` and `architecture.md`. These are minimum requirements — not aspirational goals.

---

## Pillar 1 — Operational Excellence

**Goal**: Run and monitor systems to deliver business value and continuously improve processes.

- All infrastructure changes go through CDK — never make manual changes in the AWS Console to managed resources
- Always run `cdk diff` before `cdk deploy` and review every resource change; document any unexpected diff before proceeding
- Every deployment must be traceable: use CDK stack naming convention `{Service}-{Environment}` so CloudFormation events are identifiable
- Lambda functions must emit a structured log entry at the start and end of every handler invocation (request in, response out)
- Use `@aws-lambda-powertools/metrics` to emit custom business metrics only when the metric has a clear POC learning purpose — do not add metrics by default
- Define a rollback plan before deploying breaking changes: use `cdk deploy --rollback` or maintain a previous artifact for re-deploy
- Automate repetitive operational tasks in `scripts/` — never rely on undocumented manual steps

---

## Pillar 2 — Security

**Goal**: Protect data, systems, and assets while delivering business value.

### Encryption

- **In transit**: All API endpoints must use HTTPS only — HTTP must be disabled or redirected
- **At rest**: Enable DynamoDB encryption (default with AWS-managed key is acceptable for POC; use CMK for sensitive data)
- **S3**: Enable server-side encryption on all buckets (`BucketEncryption.S3_MANAGED` minimum)

```typescript
const bucket = new s3.Bucket(this, 'Bucket', {
  encryption: s3.BucketEncryption.S3_MANAGED,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  enforceSSL: true,
});
```

### Audit Logging

- Enable AWS CloudTrail for the account — at minimum, log management events (API calls that create/modify/delete resources)
- Never disable CloudTrail logging for any environment, including `dev`
- API Gateway access logs must be enabled and sent to CloudWatch

### Dependency Security

- Run `npm audit` as part of the CI pipeline — fail the build on high or critical severity findings
- Add the following script to `package.json`:
  ```json
  "audit": "npm audit --audit-level=high"
  ```
- Regularly update pinned dependency versions; do not leave known vulnerabilities unaddressed

### Identity & Access

- Never use long-lived IAM access keys for Lambda — Lambda execution roles use temporary credentials automatically
- Never commit AWS credentials, `.env` files with keys, or access tokens to git
- Add a pre-commit check or `.gitignore` entry to block accidental credential commits
- API Gateway: `dev` endpoints must use at minimum an API key; `staging` and `prod` must use IAM auth or a JWT authorizer

---

## Pillar 3 — Reliability

**Goal**: Ensure a workload performs its intended function correctly and consistently.

### Retry & Error Recovery

- All Lambda functions invoked asynchronously (SQS, EventBridge, SNS) must be idempotent — the same event processed twice must produce the same result
- Use idempotency keys: `@aws-lambda-powertools/idempotency` with DynamoDB persistence layer
- SQS queues must have a DLQ with `maxReceiveCount: 3` — messages that fail 3 times go to DLQ for inspection
- EventBridge rules must have a DLQ on the target to catch undeliverable events

### Health Checks & Graceful Degradation

- Every API must have a `GET /health` route that returns `200` when the service is healthy
- Health check must verify downstream dependencies (DynamoDB reachability) — not just return a static response
- If a non-critical dependency is unavailable, degrade gracefully and return a partial response rather than a hard `500`

### Data Durability

- Enable DynamoDB Point-In-Time Recovery (PITR) for `prod` — see `infra-patterns.md`
- S3 buckets storing user data must have versioning enabled in `prod`
- Never rely on a single availability zone for stateful resources — DynamoDB and S3 are multi-AZ by default; preserve this by not using single-AZ configurations

### Timeout & Circuit Breaker

- Lambda timeouts must be set conservatively — fail fast rather than consume concurrency waiting
- When calling external HTTP APIs from Lambda, always set an explicit request timeout lower than the Lambda timeout:
  ```typescript
  const response = await fetch(url, { signal: AbortSignal.timeout(5000) }); // 5s max
  ```

---

## Pillar 4 — Performance Efficiency

**Goal**: Use resources efficiently to meet requirements and maintain efficiency as demand changes.

### Lambda

- Initialise AWS SDK clients, DB connections, and config outside the handler function — reuse across warm invocations
- Set `memorySize` based on profiling, not guessing — start at 256MB and adjust with Lambda Power Tuning results
- For latency-sensitive endpoints, consider provisioned concurrency to eliminate cold starts (document the trade-off in the stack)
- Avoid Lambda Layers unless the shared dependency genuinely can't be bundled — layers add cold start overhead

### DynamoDB

- Never perform full table scans — always access via partition key or a GSI designed for the access pattern
- Use DynamoDB DAX for read-heavy workloads where sub-millisecond latency is required (document if not used and why)
- Batch reads/writes where possible (`BatchGetItem`, `BatchWriteItem`) rather than individual calls in a loop
- Keep item size small — do not store large blobs in DynamoDB; use S3 and store the S3 key in DynamoDB instead

### API

- Use HTTP API Gateway (not REST API) — lower latency and cost for the common case
- Enable HTTP response compression at the API Gateway level for text payloads > 1KB
- Paginate all list endpoints — never return unbounded result sets

---

## Pillar 5 — Cost Optimization

**Goal**: Avoid unnecessary costs and understand where money is being spent.

- Use `PAY_PER_REQUEST` DynamoDB billing — never provision capacity without load testing data to justify it
- Use AWS SDK v3 modular imports only — smaller Lambda bundles mean lower cold start duration and marginally lower cost at scale
- Set Lambda `memorySize` to the minimum that meets the performance requirement — over-provisioned memory wastes money
- Delete unused stacks promptly — `cdk destroy` dev stacks when a POC phase ends
- Use `RemovalPolicy.DESTROY` on all non-production resources so stack deletion cleans up completely
- Review the AWS Cost Explorer monthly; tag all resources correctly (see `infra-patterns.md`) so costs are attributable per service
- S3 lifecycle rules: define a policy to transition or expire objects if the bucket will accumulate data over time:
  ```typescript
  bucket.addLifecycleRule({
    expiration: cdk.Duration.days(90), // adjust per use case
    enabled: !isProd,
  });
  ```

---

## Pillar 6 — Sustainability

**Goal**: Minimise environmental impact of running cloud workloads.

- Serverless by default — Lambda and DynamoDB scale to zero; there are no idle EC2 instances
- Right-size Lambda memory — over-provisioned functions consume more energy per invocation than necessary
- Avoid polling patterns — use event-driven triggers (SQS, EventBridge, DynamoDB Streams) instead of scheduled Lambda functions that poll for changes
- Set S3 lifecycle rules to expire data that is no longer needed rather than storing indefinitely
- Prefer a single AWS region per environment — cross-region replication multiplies resource consumption; only add it when the reliability requirement justifies it

---

## Well-Architected Review Checklist

Run through this before marking any feature or service as complete:

### Security
- [ ] All data encrypted at rest (DynamoDB, S3)
- [ ] All endpoints use HTTPS; HTTP disabled
- [ ] No IAM wildcard `*` on actions or resources
- [ ] No secrets in environment variables, code, or git
- [ ] `npm audit` passes with no high/critical findings
- [ ] API Gateway has authentication on all non-health routes

### Reliability
- [ ] All async Lambda consumers are idempotent
- [ ] SQS queues have DLQ with `maxReceiveCount: 3`
- [ ] `GET /health` endpoint exists and checks dependencies
- [ ] DynamoDB PITR enabled for `prod`

### Performance
- [ ] AWS SDK clients initialised outside handler
- [ ] No table scans in any code path
- [ ] All list endpoints are paginated
- [ ] Lambda `memorySize` and `timeout` explicitly set

### Cost
- [ ] DynamoDB uses `PAY_PER_REQUEST`
- [ ] Budget alarm exists for the account
- [ ] All resources tagged with `CostCenter`
- [ ] S3 lifecycle rules defined where applicable

### Operational Excellence
- [ ] Structured logging on every handler
- [ ] CloudTrail enabled for the account
- [ ] No manual console changes to CDK-managed resources
- [ ] CW alarms added only where essential (see `poc-cost-guardrails.md`)
- [ ] All CW log groups have retention set to 1 day
