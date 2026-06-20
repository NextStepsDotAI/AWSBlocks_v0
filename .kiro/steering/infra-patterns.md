---
inclusion: always
---

# Infrastructure Patterns (AWS CDK)

## Overview

In this project, AWS CDK is used in two distinct roles:

1. **Escape hatch** — overriding or extending Block-managed infrastructure when Block defaults are insufficient for a demo point. This code lives in `infra/lib/`.
2. **Comparison demos** — the `comparison/` folder uses raw CDK to replicate the team's current approach. This code is intentionally verbose to make the contrast with AWS Blocks visible.

Apply the patterns in this document to both roles. CDK is never used to re-implement something a Block already handles.

## CDK Version & Imports

- Use **AWS CDK v2** — single package `aws-cdk-lib`; never import from individual `@aws-cdk/*` packages
- Import pattern:
  ```typescript
  import * as cdk from 'aws-cdk-lib';
  import * as lambda from 'aws-cdk-lib/aws-lambda';
  import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
  // etc.
  ```
- Use `constructs` package for the `Construct` base class

## Construct Hierarchy

Organise CDK code into three levels:

| Level | Description | Example |
|---|---|---|
| L1 (Cfn*) | Raw CloudFormation — avoid unless no L2 exists | `CfnTable` |
| L2 | AWS-provided high-level constructs — prefer these | `Table`, `Function` |
| L3 (Patterns) | Custom reusable constructs wrapping L2s — build these for repeated patterns | `LambdaWithDlq` |

Build L3 constructs when the same combination of resources appears more than once. Place them in `infra/lib/shared/constructs/`.

## Stack Structure

**Escape hatch stacks** (in `infra/lib/`) follow this pattern — always document why CDK is needed instead of a Block:

```typescript
export interface UserServiceStackProps extends cdk.StackProps {
  environment: 'dev' | 'staging' | 'prod';
}

export class UserServiceStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: UserServiceStackProps) {
    super(scope, id, props);
    // WHY CDK: [explain what Block default was insufficient and why]
    // 1. Data layer overrides
    // 2. Compute overrides
    // 3. API layer overrides
    // 4. Event sources
  }
}
```

**Comparison stacks** (in `comparison/`) are intentionally explicit — they should represent how the team currently builds, not a simplified version:

```typescript
export class ComparisonUserServiceStack extends cdk.Stack {
  // Deliberately verbose: shows full CDK config a team member would write today
  // This is the "before" in the AWS Blocks before/after contrast
}
```

## Environment Handling

```typescript
const isProd = props.environment === 'prod';

// Removal policy
const removalPolicy = isProd ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY;

// DynamoDB point-in-time recovery
const pointInTimeRecovery = isProd;
```

- `dev` and `staging`: resources can be destroyed on stack deletion (`DESTROY`)
- `prod`: stateful resources (DynamoDB tables, S3 buckets) must use `RETAIN`

## Resource Tagging

Tagging is mandatory for all AWS resources. Tags enable cost allocation, access control, and operational visibility. Apply tags at the **stack level** so they propagate automatically to every child resource via CloudFormation.

### Mandatory Tags

Every stack must apply all of the following tags:

| Tag Key | Description | Example Values |
|---|---|---|
| `Project` | Project or product name | `AWSBlocks` |
| `Environment` | Deployment environment | `dev`, `staging`, `prod` |
| `Service` | Microservice name within the project | `user-service`, `order-service` |
| `Owner` | Team or individual responsible | `platform-team`, `jane.doe` |
| `ManagedBy` | How the resource was provisioned | `CDK` (always this value) |
| `CostCenter` | Billing/cost allocation code | `cc-1234` |

### Enforcement Pattern

Apply mandatory tags in the stack constructor **before** any resources are defined, so all child constructs inherit them:

```typescript
export class UserServiceStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: UserServiceStackProps) {
    super(scope, id, props);

    // Apply mandatory tags immediately — must come before resource definitions
    cdk.Tags.of(this).add('Project', 'AWSBlocks');
    cdk.Tags.of(this).add('Environment', props.environment);
    cdk.Tags.of(this).add('Service', props.serviceName);
    cdk.Tags.of(this).add('Owner', props.owner);
    cdk.Tags.of(this).add('ManagedBy', 'CDK');
    cdk.Tags.of(this).add('CostCenter', props.costCenter);

    // Resource definitions follow...
  }
}
```

### Stack Props Interface

Extend the base stack props interface to enforce required tagging inputs:

```typescript
export interface BaseStackProps extends cdk.StackProps {
  environment: 'dev' | 'staging' | 'prod';
  serviceName: string;
  owner: string;
  costCenter: string;
}
```

All service stacks must extend `BaseStackProps` — never use raw `cdk.StackProps` directly.

### Additional Tags

Add resource-level tags for context where stack-level tags are insufficient:

```typescript
// Example: tag a specific Lambda with its runtime characteristics
cdk.Tags.of(handler).add('Runtime', 'nodejs24.x');
cdk.Tags.of(handler).add('Trigger', 'api-gateway');
```

### Rules

- Never create a stack without applying all mandatory tags
- Tag values must be lowercase kebab-case or dot-notation — no spaces or special characters
- `Environment` values are strictly `dev`, `staging`, or `prod` — no other values
- `ManagedBy` is always `CDK` — never change this value manually
- Do not apply tags inline on individual `cdk.StackProps` — use `cdk.Tags.of()` for consistency

## Lambda Construct Pattern

```typescript
const handler = new lambda.Function(this, 'Handler', {
  runtime: lambda.Runtime.NODEJS_24_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset(path.join(__dirname, '../../../dist/handlers/user')),
  memorySize: 256,
  timeout: cdk.Duration.seconds(30),
  environment: {
    TABLE_NAME: table.tableName,
    POWERTOOLS_SERVICE_NAME: 'user-service',
    LOG_LEVEL: isProd ? 'INFO' : 'DEBUG',
  },
  tracing: lambda.Tracing.ACTIVE,
  reservedConcurrentExecutions: isProd ? 100 : undefined,
});
```

- Always enable `tracing: lambda.Tracing.ACTIVE`
- Always set explicit `memorySize` and `timeout` — never rely on defaults
- Pass resource names/ARNs via environment variables; never hardcode inside Lambda source code
- Grant permissions using CDK grant methods (`table.grantReadWriteData(handler)`) — never write inline IAM policies manually unless no grant method exists

## DynamoDB Construct Pattern

```typescript
const table = new dynamodb.Table(this, 'Table', {
  tableName: `awsblocks-${props.environment}-users`,
  partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  pointInTimeRecovery: isProd,
  removalPolicy,
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES, // if streams needed
});
```

## SQS + DLQ Pattern

Always pair every SQS queue with a Dead Letter Queue:

```typescript
const dlq = new sqs.Queue(this, 'Dlq', {
  retentionPeriod: cdk.Duration.days(14),
});

const queue = new sqs.Queue(this, 'Queue', {
  visibilityTimeout: cdk.Duration.seconds(90), // 3x Lambda timeout
  deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
});
```

- Set `visibilityTimeout` to at least 3× the Lambda timeout
- Set `maxReceiveCount` to 3 unless there is a specific reason to change it
- Create a CloudWatch alarm on DLQ depth > 0

## IAM Least Privilege

- Never use `*` in resource ARNs for IAM policies — scope to the specific resource
- Never use `AdministratorAccess` or `PowerUserAccess` for Lambda execution roles
- Use CDK grant methods as the first choice; write `PolicyStatement` only when no grant method covers the need
- Lambda execution roles are created automatically by CDK — add only the permissions the function actually needs

## Secrets & Configuration

- Secrets (API keys, DB passwords): store in **AWS Secrets Manager**; access via SDK at Lambda cold start, outside the handler
- Non-secret config (feature flags, URLs): store in **SSM Parameter Store**; pass as environment variable via CDK during deploy
- Never put secret values in CDK code, CloudFormation parameters, or Lambda environment variables

## CDK Context & Config

Define environment-specific values in `cdk.json` under the `context` key:

```json
{
  "context": {
    "dev": { "account": "123456789012", "region": "us-east-1" },
    "prod": { "account": "987654321098", "region": "us-east-1" }
  }
}
```

Read in `bin/app.ts`:

```typescript
const env = app.node.tryGetContext('env') ?? 'dev';
const config = app.node.tryGetContext(env);
```

## POC Guardrails

This project runs as a POC. All environments (including `dev`) must apply execution and cost controls to prevent runaway usage, unexpected charges, and unintended access. These guardrails are **non-negotiable** — apply them to every service stack.

### Lambda Concurrency Limits

Always set `reservedConcurrentExecutions` on every Lambda function. Never leave it unbounded.

| Environment | Default Limit |
|---|---|
| `dev` | 5 |
| `staging` | 20 |
| `prod` | 100 |

```typescript
const pocLimits: Record<string, number> = { dev: 5, staging: 20, prod: 100 };

const handler = new lambda.Function(this, 'Handler', {
  // ...
  reservedConcurrentExecutions: pocLimits[props.environment],
});
```

Override only with documented justification in the stack file.

### API Gateway Throttling

Apply throttling at the stage level on every HTTP API. These are POC-safe defaults:

| Environment | Rate (req/s) | Burst |
|---|---|---|
| `dev` | 10 | 20 |
| `staging` | 50 | 100 |
| `prod` | 200 | 400 |

```typescript
const api = new apigatewayv2.HttpApi(this, 'Api', {
  defaultStage: new apigatewayv2.HttpStage(this, 'DefaultStage', {
    throttle: {
      rateLimit: isProd ? 200 : props.environment === 'staging' ? 50 : 10,
      burstLimit: isProd ? 400 : props.environment === 'staging' ? 100 : 20,
    },
    autoDeploy: true,
  }),
});
```

### SQS Message Limits

Cap SQS throughput to prevent Lambda fan-out storms in POC environments:

```typescript
new lambda.EventSourceMapping(this, 'QueueTrigger', {
  target: handler,
  eventSourceArn: queue.queueArn,
  batchSize: props.environment === 'prod' ? 10 : 2,       // small batches in dev
  maxConcurrency: props.environment === 'prod' ? 50 : 5,  // cap concurrent consumers
});
```

### DynamoDB Capacity Limits

Use `PAY_PER_REQUEST` for all environments. For `dev`, additionally set a CloudWatch alarm if read/write capacity units exceed a threshold to detect runaway consumers early.

Never provision DynamoDB with `PROVISIONED` billing mode in POC — it offers no upper cost bound without auto-scaling guardrails.

### EventBridge Rule Guards

- Disable EventBridge rules in `dev` by default unless explicitly needed for the test scenario
- Always set a `DLQ` on EventBridge targets to catch undelivered events

```typescript
const rule = new events.Rule(this, 'Rule', {
  schedule: events.Schedule.rate(cdk.Duration.minutes(5)),
  enabled: props.environment !== 'dev', // disabled in dev by default
});
```

### AWS Budgets & Cost Alerts

Define a budget alarm in the CDK app entry point (`bin/app.ts`) for the POC account:

```typescript
new budgets.CfnBudget(this, 'PocBudget', {
  budget: {
    budgetType: 'COST',
    timeUnit: 'MONTHLY',
    budgetLimit: { amount: 50, unit: 'USD' }, // adjust per POC scope
  },
  notificationsWithSubscribers: [{
    notification: {
      notificationType: 'ACTUAL',
      comparisonOperator: 'GREATER_THAN',
      threshold: 80, // alert at 80% of budget
    },
    subscribers: [{ subscriptionType: 'EMAIL', address: 'team@example.com' }],
  }],
});
```

### Access Control Guardrails

- POC Lambda functions must never have internet egress unless explicitly required — use VPC endpoint or keep functions outside VPC with explicit security group rules
- IAM roles must include a `Condition` block limiting invocation to the specific AWS account:
  ```typescript
  new iam.PolicyStatement({
    effect: iam.Effect.DENY,
    actions: ['*'],
    resources: ['*'],
    conditions: { StringNotEquals: { 'aws:RequestedRegion': 'us-east-1' } },
  });
  ```
- Never enable public S3 bucket access — all buckets must have `blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL`
- API Gateway endpoints in `dev` must require an API key or IAM auth — never leave them fully open

### Guardrail Checklist

Before deploying any stack, verify:
- [ ] Every Lambda has `reservedConcurrentExecutions` set
- [ ] Every API Gateway stage has throttle limits configured
- [ ] Every SQS event source mapping has `maxConcurrency` set
- [ ] Every S3 bucket has `blockPublicAccess: BLOCK_ALL`
- [ ] A budget alarm exists for the account
- [ ] No IAM role has `*` on actions or resources

---

## CloudWatch Alarms

**POC rule: Do not create CloudWatch alarms unless essential.**

CloudWatch alarms are continuously executing monitors — each alarm evaluates on a recurring schedule and counts against your CloudWatch metrics quota. In a POC, most operational insight can be gained by inspecting logs and traces on demand rather than through always-on alarms.

### When an alarm IS essential (permitted)

Only create a CW alarm when all three of the following are true:
1. The condition it detects could cause **uncontrolled cost** if unnoticed (e.g., runaway Lambda invocations, DLQ depth growing unboundedly)
2. The team cannot reasonably detect the condition by checking CloudWatch manually during active POC sessions
3. There is a defined response action tied to the alarm (e.g., an SNS alert that someone will act on)

**Permitted alarms for POC:**

| Alarm | Reason | Threshold |
|---|---|---|
| Estimated AWS charges | Uncontrolled cost — cannot afford to miss this | > 80% of monthly budget |
| SQS DLQ depth | Indicates a broken consumer loop that may keep retrying | ≥ 1 message |
| Lambda concurrency | Detects a runaway execution pattern before it drains the account | > 80% of `reservedConcurrentExecutions` |

All other alarms (Lambda errors, throttles, duration, API Gateway 5xx) are **optional** in POC — use X-Ray traces and CloudWatch Logs Insights for on-demand investigation instead.

### How to create a permitted alarm

```typescript
// Only create this if it meets the 3 criteria above
new cloudwatch.Alarm(this, 'DlqDepthAlarm', {
  metric: dlq.metricApproximateNumberOfMessagesVisible(),
  threshold: 1,
  evaluationPeriods: 1,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
  alarmDescription: 'POC essential: DLQ has messages — consumer may be in a broken loop',
});
```

Always add an `alarmDescription` explaining why this alarm meets the essential criteria.
