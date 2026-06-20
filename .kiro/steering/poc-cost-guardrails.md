---
inclusion: always
---

# POC Cost Guardrails

This project is a POC. Every architectural and service choice must be evaluated against cost impact first. The rules in this document are **mandatory** — they override convenience or habit. When in doubt, choose the cheaper option and document the trade-off.

---

## Region Selection

**Always deploy to `us-east-1` (N. Virginia) unless there is a documented reason not to.**

`us-east-1` is consistently the lowest-cost AWS region for serverless services (Lambda, DynamoDB, API Gateway, SQS, S3). Other regions carry a premium:

| Region | Relative Lambda Cost |
|---|---|
| `us-east-1` (N. Virginia) | Baseline — cheapest |
| `us-west-2` (Oregon) | ~same as us-east-1 |
| `eu-west-1` (Ireland) | ~+10–15% |
| `ap-southeast-1` (Singapore) | ~+25% |
| `sa-east-1` (São Paulo) | ~+50% |

**Rules:**
- Set `env.region = 'us-east-1'` as the default in `cdk.json`
- Never deploy a POC stack to a more expensive region just because it is geographically closer to the developer
- If a specific region is required (data residency, latency testing), document it explicitly in the stack file with a cost justification comment

---

## Free Tier First

**Only use AWS services that have an Always Free tier or a generous 12-month free tier for POC workloads.**

The following services are pre-approved for POC use because they have always-free or 12-month free allowances that cover typical POC traffic:

| Service | Free Tier Allowance | Type |
|---|---|---|
| AWS Lambda | 1M requests/month + 400K GB-seconds/month | Always free |
| Amazon DynamoDB | 25 GB storage + 25 WCU + 25 RCU | Always free |
| Amazon S3 | 5 GB standard storage + 20K GET + 2K PUT | 12-month free |
| Amazon SQS | 1M requests/month | Always free |
| Amazon SNS | 1M publishes/month | Always free |
| Amazon API Gateway (HTTP) | 1M HTTP API calls/month | 12-month free |
| AWS CloudWatch | 10 custom metrics + 5GB log ingestion/month | Always free |
| AWS X-Ray | 100K traces/month recorded, 1M scanned | Always free |
| AWS Secrets Manager | — | **No free tier — see below** |
| AWS Systems Manager (SSM Parameter Store standard) | No charge for standard parameters | Always free |

**Rules:**
- Before adding any new AWS service to a stack, verify it has a free tier at [aws.amazon.com/free](https://aws.amazon.com/free)
- If a service has no free tier (e.g., AWS Secrets Manager charges ~$0.40/secret/month, NAT Gateway ~$32/month), evaluate whether a free-tier alternative can substitute:
  - Secrets Manager → SSM Parameter Store `SecureString` (encrypted, free for standard tier) for POC secrets
  - NAT Gateway → avoid VPC entirely for POC Lambdas; use public subnets or VPC endpoints only if strictly required
  - ElastiCache → avoid caching layers in POC; DynamoDB DAX has no free tier either
- Never introduce a service solely for architectural completeness if it adds cost with no POC learning value

---

## Cross-Region Data Transfer — Strictly Prohibited

**Never design a flow that transfers data across AWS regions.**

Cross-region data transfer is charged at $0.02/GB outbound. This sounds small but adds up quickly in iterative development and testing. More importantly, it is never necessary in a single-region POC.

**Prohibited patterns:**
- Calling an API Gateway or Lambda in a different region from the caller
- DynamoDB Global Tables (multi-region replication) — POC does not need this
- S3 Cross-Region Replication — use a single bucket in `us-east-1`
- CloudFront with origins in a different region than the CDK deployment region
- SNS/SQS cross-region subscriptions or message forwarding

**Rules:**
- All resources in a POC stack must be in the same region
- All Lambda-to-Lambda calls, Lambda-to-DynamoDB calls, and Lambda-to-SQS calls must target resources in the same region
- If an existing resource (e.g., a shared S3 bucket) is in another region, do not reference it — create a local equivalent for the POC

---

## Cross-AZ Data Transfer

**Minimise cross-AZ traffic — it is billed at $0.01/GB in each direction.**

- Lambda functions are not VPC-attached by default — this is the preferred POC configuration as it avoids AZ-to-AZ charges entirely
- If VPC is required, ensure Lambda and DynamoDB/SQS endpoints are in the same AZ where possible, or use VPC Interface Endpoints (which also have hourly charges — avoid in POC unless necessary)
- Never place a NAT Gateway in the architecture unless absolutely required; a single NAT Gateway costs ~$32/month at zero traffic

---

## Data Egress (Internet-Out) Costs

**Minimise data sent out of AWS to the internet — charged at $0.09/GB after the first 100GB/month free.**

- Keep API response payloads small — paginate, filter, and compress
- Enable HTTP response compression on API Gateway for payloads > 1KB
- Do not stream large files (video, large CSVs) directly through Lambda — use S3 pre-signed URLs so the client downloads directly from S3 (S3 egress is cheaper and better for large objects)
- CloudFront's free tier covers 1TB/month outbound — use it as a CDN layer if static assets need to be served externally

---

## CloudWatch Log Costs

**CloudWatch Logs are a common surprise cost — $0.50/GB ingested, $0.03/GB stored/month.**

- Set Lambda log retention explicitly — never leave it at the default (infinite):
  ```typescript
  const handler = new lambda.Function(this, 'Handler', {
    // ...
    logRetention: logs.RetentionDays.ONE_DAY, // POC: 1 day — logs have no value beyond the active session
  });
  ```
- Use `DEBUG` log level only during active development; switch to `INFO` before leaving a stack running unattended
- Never log full request/response payloads — log only IDs and status codes
- Set a CloudWatch log group retention policy of 1 day on every log group created by CDK:
  ```typescript
  new logs.LogGroup(this, 'ApiLogs', {
    retention: logs.RetentionDays.ONE_DAY,
    removalPolicy: cdk.RemovalPolicy.DESTROY,
  });
  ```
- **Do not create CloudWatch alarms unless essential** — see `infra-patterns.md` for the 3-criteria test. Most POC observability needs are served by checking CloudWatch Logs Insights and X-Ray traces on demand.

---

## Idle Resource Cleanup

**POC resources left running with no traffic still incur charges for some services.**

- Destroy dev and staging stacks when not actively in use: `cdk destroy --context env=dev`
- Use `RemovalPolicy.DESTROY` on all non-production resources so teardown is complete
- Never leave scheduled EventBridge rules or SQS polling Lambdas running overnight on an idle POC — disable or destroy
- Set a calendar reminder to review and destroy stale POC stacks monthly

---

## Cost Monitoring

- Enable Free Tier usage alerts in AWS Billing (Console → Billing → Free Tier alerts)
- Set a monthly budget alarm at $50 (or appropriate POC budget) — see `infra-patterns.md` for CDK pattern
- Review AWS Cost Explorer weekly during active POC development
- Tag all resources correctly (`CostCenter`, `Project`, `Environment`) — without tags, cost attribution is impossible

---

## Execution Lifecycle Controls

**Every scheduled or continuous execution must have a defined stop condition. Nothing runs indefinitely in a POC.**

This applies to all compute, scheduled triggers, and message-driven flows. If a process can start automatically, it must also stop automatically or require a manual restart after stopping.

### EC2 Instances (if used)

EC2 should be avoided in this POC in favour of Lambda — but if an EC2 instance is introduced:

- **Start**: always manual — never configure auto-start on boot or via a scheduled rule
- **Stop**: always automated — every instance must have a scheduled shutdown:
  ```typescript
  // SSM Maintenance Window or EventBridge rule to stop instance nightly
  new events.Rule(this, 'StopInstance', {
    schedule: events.Schedule.cron({ hour: '18', minute: '0' }), // stops at 6PM UTC daily
    targets: [new targets.AwsApi({
      service: 'EC2',
      action: 'stopInstances',
      parameters: { InstanceIds: [instance.instanceId] },
    })],
  });
  ```
- Never create an EC2 instance without a corresponding stop rule in the same CDK stack
- Set Instance Initiated Shutdown Behaviour to `terminate` for POC instances so accidental shutdowns clean up the resource entirely

### EventBridge Scheduled Rules

- Every scheduled rule must have an explicit `enabled: false` default in `dev` — enable only when actively testing:
  ```typescript
  new events.Rule(this, 'ScheduledJob', {
    schedule: events.Schedule.rate(cdk.Duration.minutes(5)),
    enabled: false, // must be manually enabled; never auto-enabled in POC
  });
  ```
- Every scheduled rule must have a paired **disable rule** triggered at a fixed time after enabling, implemented as a second EventBridge rule or a Step Functions state machine with a wait state
- Scheduled rules must not run at intervals shorter than 5 minutes in POC — there is no POC use case that requires sub-5-minute polling
- Document the purpose and expected run duration in a comment on every scheduled rule

### Lambda–SQS Flows

**A Lambda–SQS consumer flow must not execute indefinitely.** Every such flow must have at least one of the following stop conditions:

1. **Message TTL**: Set `messageRetentionPeriod` on the queue — messages expire and are not processed after the retention window:
   ```typescript
   const queue = new sqs.Queue(this, 'Queue', {
     messageRetentionPeriod: cdk.Duration.hours(1), // POC default: 1 hour max
     visibilityTimeout: cdk.Duration.seconds(90),
     deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
   });
   ```

2. **Event source mapping disabled by default**: The Lambda–SQS trigger must start disabled and be enabled only during active testing:
   ```typescript
   const esm = new lambda.EventSourceMapping(this, 'Trigger', {
     target: handler,
     eventSourceArn: queue.queueArn,
     enabled: false, // must be manually enabled for each test run
     batchSize: 2,
     maxConcurrency: 5,
   });
   ```

3. **Maximum receive count + DLQ**: After `maxReceiveCount` (set to 3) failures, messages route to the DLQ and stop triggering the Lambda. The DLQ must not have its own Lambda consumer in POC — it is for inspection only.

4. **Scheduled disable**: If a flow must run continuously during a test window, pair it with an EventBridge rule that disables the event source mapping after a fixed duration:
   ```typescript
   // Disable the ESM 2 hours after a test session starts
   new events.Rule(this, 'DisableConsumer', {
     schedule: events.Schedule.cron({ hour: '20', minute: '0' }),
     targets: [new targets.AwsApi({
       service: 'Lambda',
       action: 'updateEventSourceMapping',
       parameters: { UUID: esm.eventSourceMappingId, Enabled: false },
     })],
   });
   ```

### Step Functions

- Every Step Functions state machine must have a defined end state — no infinite loops (`while true` patterns)
- If a polling loop is required (wait for condition), use a `Wait` state with a maximum iteration count enforced via a `Choice` state:
  ```typescript
  // Pattern: retry up to N times then fail, never loop forever
  const checkCondition = new sfn.Choice(this, 'CheckCondition')
    .when(sfn.Condition.numberLessThan('$.retryCount', 10), waitAndRetry)
    .otherwise(new sfn.Fail(this, 'MaxRetriesExceeded'));
  ```
- Set `timeout` on every state machine execution to cap wall-clock runtime:
  ```typescript
  new sfn.StateMachine(this, 'Machine', {
    definition,
    timeout: cdk.Duration.hours(1), // POC: hard cap at 1 hour
  });
  ```

### General Rules

- No process may run continuously without a stop condition — document the stop condition in a comment in the CDK stack
- If a resource must be re-enabled for a new test run, that is a feature not a bug — it forces intentional activation
- Any flow that could generate unbounded invocations (recursive Lambda, infinite SQS loop, chained EventBridge events) is **prohibited** in POC without an explicit circuit breaker
- Recursive Lambda patterns (Lambda invoking itself) are banned entirely in POC

---

## POC Cost Checklist

Before deploying any new service or stack:

- [ ] Deployment region is `us-east-1` or documented exception exists
- [ ] Every AWS service used has a free tier or cost is explicitly accepted
- [ ] No cross-region data flows exist in the architecture
- [ ] No NAT Gateway in the stack
- [ ] Lambda log retention set to 1 day (`RetentionDays.ONE_DAY`)
- [ ] No CloudWatch alarms created unless they meet the 3-criteria essential test
- [ ] All stacks use `RemovalPolicy.DESTROY`
- [ ] Budget alarm exists for the account
- [ ] Free Tier alerts enabled in AWS Billing Console
- [ ] Every scheduled rule has `enabled: false` as default
- [ ] Every Lambda–SQS event source mapping has `enabled: false` as default
- [ ] Every SQS queue has `messageRetentionPeriod` ≤ 1 hour for POC queues
- [ ] No EC2 instance exists without a paired scheduled stop rule
- [ ] No Step Functions state machine has an infinite loop — every execution has a timeout
- [ ] No recursive Lambda invocation patterns exist in any handler
