## Amazon Connect Customer with AWS DevOps Agent

CloudFormation template accompanying the AWS Contact Center blog - Reduce MTTR in Amazon Connect Customer with AWS DevOps Agent

`devops-agent-stack.yaml` is a single CloudFormation template that monitors an Amazon Connect instance with CloudWatch alarms and starts an AWS DevOps Agent investigation the moment an alarm fires. Your team gets the SNS email in parallel. The agent gives them a head start on the investigation.

## How it works

```text
Amazon Connect ──metrics──▶ CloudWatch alarm ──ALARM──▶ EventBridge rule ──▶ Lambda forwarder ──HMAC webhook──▶ DevOps Agent
      │                          │                          (Connect- or           │                                investigation
      └─flow logs─▶ metric filter┘                           DevOpsAgent- prefix)  └─on failure─▶ SQS dead-letter queue
                                 └────────────────────────▶ SNS topic ──▶ email
```

1. A CloudWatch alarm with a `Connect-` or `DevOpsAgent-` name transitions to `ALARM`.
2. An EventBridge rule matches the state change and invokes the Lambda forwarder.
3. The forwarder reads the webhook URL and HMAC secret from Secrets Manager.
4. It signs the payload with HMAC-SHA256 and POSTs it to the DevOps Agent webhook.
5. DevOps Agent starts an investigation.
6. If delivery fails, the event lands in an SQS dead-letter queue, and an alarm tells you.

The forwarder only acts on transitions into `ALARM`. `OK` events are ignored. The EventBridge rule only matches `ALARM`, so the forwarder never sees `OK` events in practice.

Two paths feed the dead-letter queue: the EventBridge target (3 retries, up to 1 hour event age) if it can't invoke the Lambda, and the Lambda's own async failure config if the function errors.

Separately, every alarm in the stack also sends to the SNS topic on `ALARM`, and on `OK` for every alarm except `DevOpsAgent-NoTopologyCompletions`. So you get recovery emails as well as alert emails.

## What gets deployed

| Category | Resources |
|---|---|
| DevOps Agent | An Agent Space, its operator app role, and an association with your AWS account in monitor mode |
| Alarms | 21 always-on CloudWatch alarms, plus 6 optional ones (see below) |
| Auto-investigation | EventBridge rule, Lambda forwarder (Python 3.12), Secrets Manager secret, Lambda permission |
| Notifications | SNS topic encrypted with a customer-managed KMS key (plus alias), a topic policy, and an email subscription |
| Reliability | SQS dead-letter queue (14-day retention) and its queue policy |
| Observability | CloudWatch dashboard, and log delivery (source, destination, and delivery) for Agent Space logs to CloudWatch Logs |
| IAM | Three roles: Agent Space, operator app, and alarm forwarder. The Agent Space role also may create the Resource Explorer service-linked role. |

The roles, KMS key, SNS topic, secret, DLQ, Lambda function, and log group are tagged with `Project` and `Environment`. The alarms, dashboard, and Agent Space are not tagged by the template.

### Always-on alarms (21)

| Group | Count | Alarms |
|---|---|---|
| Voice | 7 | `Connect-MisconfiguredPhoneNumbers`, `Connect-ThrottledCalls`, `Connect-CallsBreachingConcurrencyQuota`, `Connect-HighMissedCalls`, `Connect-HighConcurrentCallsPercent`, `Connect-CallRecordingUploadErrors`, `Connect-HighPacketLoss` |
| Chat, tasks, email | 5 | `Connect-ChatsBreachingActiveChatQuota`, `Connect-HighConcurrentChatsPercent`, `Connect-TasksBreachingConcurrencyQuota`, `Connect-HighConcurrentTasksPercent`, `Connect-HighConcurrentEmailsPercent` |
| Contact Lens | 2 | `Connect-ContactLens-HighConcurrentRealTimeAnalytics`, `Connect-ContactLens-HighConcurrentPostCallJobs` |
| DevOps Agent health | 4 | `DevOpsAgent-HighChatRequestConsumption`, `DevOpsAgent-HighInvestigationTime`, `DevOpsAgent-HighEvaluationTime`, `DevOpsAgent-NoTopologyCompletions` |
| Webhook health | 3 | `Infra-WebhookDeliveryFailures`, `Infra-AlarmForwarderErrors`, `Infra-AlarmForwarderThrottles` |

`Connect-HighPacketLoss` only watches agent-side WebRTC voice. Its dimensions are fixed in the template (Participant=Agent, Type of Connection=WebRTC, Stream Type=Voice).

The three webhook-health alarms use an `Infra-` prefix on purpose. The EventBridge rule only matches `Connect-` and `DevOpsAgent-`, so a failure in the forwarder doesn't trigger an investigation of the forwarder itself. These three alarms still email you through SNS.

### Optional alarms (6)

These are created only when you set the matching parameter.

| Set this parameter | You get |
|---|---|
| `ConnectFlowLogGroupName` | A metric filter on that log group, plus `Connect-FlowErrors-FromLogs` |
| `MonitoredContactFlowName` | `Connect-ContactFlowErrors`, `Connect-ContactFlowFatalErrors`, `Connect-CallBackNotDialableNumber` |
| `MonitoredQueueName` | `Connect-QueueCapacityExceeded`, `Connect-LongestQueueWaitTime` |

**Flow errors: use the log-based detector.** Connect publishes flow-error metrics per contact flow, so a per-flow alarm needs one specific flow name. The log-based detector (`ConnectFlowLogGroupName`) counts error events across every flow that writes to the log group, with no flow names needed. It requires flow logging on the instance and a **Set logging behavior** block in the flows you want covered. A metric filter attaches to one log group, so this covers one instance's flow-log group, not every log group.

**Pick one approach, not both.** If you set both `ConnectFlowLogGroupName` and `MonitoredContactFlowName`, a single flow error trips both alarms and starts two investigations for one root cause.

### TasksExpired has no alarm

Connect publishes `TasksExpired` per contact, which CloudWatch alarms can't aggregate into an instance-level total. The template leaves it out of the alarms. Monitor it with a scheduled metric-math query.

## Prerequisites

- An Amazon Connect instance, and its instance ID (UUID format, lowercase hex only)
- AWS DevOps Agent available in your Region. The template targets `us-east-1`.
- Permissions to create IAM roles (including named roles), Lambda functions, CloudWatch resources (alarms, dashboards, metric filters), CloudWatch Logs log delivery (delivery source, destination, and delivery), Secrets Manager secrets, KMS keys and aliases, SNS topics and policies, SQS queues and policies, EventBridge rules, and DevOps Agent resources. The Agent Space role is allowed to create the Resource Explorer service-linked role, so your deploying identity needs IAM permission to create the roles that grant it.
- AWS CLI v2

## Parameters

Only two parameters are required. Everything else has a default.

| Parameter | Default | Description |
|---|---|---|
| `AlarmNotificationEmail` | none (required) | Email address for alarm notifications |
| `ConnectInstanceId` | none (required) | Amazon Connect instance ID, in UUID format (lowercase hex, enforced by the template) |
| `ConnectFlowLogGroupName` | blank | Connect flow-log group, for example `/aws/connect/<alias>`. Blank skips the log-based detector. |
| `MonitoredContactFlowName` | blank | Exact flow name for the per-flow alarms. Blank skips them. |
| `MonitoredQueueName` | blank | Exact queue name for the per-queue alarms. Blank skips them. |
| `AgentSpaceName` | `ProductionAgentSpace` | Name of the Agent Space |
| `AgentSpaceDescription` | `Production DevOps Agent space with full observability` | Description of the Agent Space |
| `EnvironmentTag` | `production` | `production`, `staging`, or `development` |
| `LogRetentionDays` | `30` | Retention for the Agent Space log group |

Every alarm threshold is also a parameter:

| Parameter | Default |
|---|---|
| `ChatRequestAlarmThreshold` | `100` |
| `InvestigationTimeAlarmThreshold` | `3600` (seconds) |
| `EvaluationTimeAlarmThreshold` | `1800` (seconds) |
| `TopologyCompletionMinCount` | `0` |
| `ContactFlowErrorThreshold` | `5` |
| `ContactFlowFatalErrorThreshold` | `1` |
| `FlowLogErrorThreshold` | `0` (alert on any flow error) |
| `MissedCallsThreshold` | `10` |
| `ThrottledCallsThreshold` | `1` |
| `ConcurrentCallsPercentThreshold` | `80` |
| `ConcurrentChatsPercentThreshold` | `80` |
| `ConcurrentTasksPercentThreshold` | `80` |
| `ConcurrentEmailsPercentThreshold` | `80` |
| `QueueWaitTimeThreshold` | `300` (seconds) |
| `PacketLossPercentThreshold` | `5` |
| `ContactLensConcurrentRealTimeThreshold` | `240` |
| `ContactLensConcurrentPostCallThreshold` | `160` |
| `MisconfiguredPhoneNumbersThreshold` | `0` |
| `CallsBreachingQuotaThreshold` | `0` |
| `CallRecordingUploadErrorThreshold` | `0` |
| `QueueCapacityExceededThreshold` | `0` |
| `ChatsBreachingQuotaThreshold` | `0` |
| `TasksBreachingQuotaThreshold` | `0` |
| `CallBackNotDialableThreshold` | `0` |
| `WebhookDeliveryFailureThreshold` | `0` |
| `AlarmForwarderErrorThreshold` | `0` |
| `AlarmForwarderThrottleThreshold` | `0` |

Most `0` defaults alert on any occurrence. Raise them if your baseline is noisy. `TopologyCompletionMinCount` works differently: the alarm fires when the daily topology completion count is at or below this value, so `0` means "alert when none ran". Missing data counts as breaching for that alarm.

The dashboard's threshold lines (5% packet loss, 80% quota, 300 and 200 Contact Lens quotas) are fixed in the template. They don't follow these parameters.

## Deploy

The template is large enough that you should deploy it from S3 rather than inline. These commands use bash. In PowerShell, set variables as `$BUCKET = "..."` and use a backtick (`` ` ``) instead of `\` for line continuation.

**1. Upload the template**

```bash
BUCKET=<your-bucket>
REGION=us-east-1

aws s3 cp devops-agent-stack.yaml s3://$BUCKET/devops-agent/devops-agent-stack.yaml
```

**2. Validate it**

```bash
aws cloudformation validate-template \
  --template-url https://$BUCKET.s3.$REGION.amazonaws.com/devops-agent/devops-agent-stack.yaml
```

**3. Create the stack**

```bash
aws cloudformation create-stack \
  --stack-name devops-agent-stack \
  --template-url https://$BUCKET.s3.$REGION.amazonaws.com/devops-agent/devops-agent-stack.yaml \
  --parameters \
    ParameterKey=ConnectInstanceId,ParameterValue=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee \
    ParameterKey=AlarmNotificationEmail,ParameterValue=you@example.com \
    ParameterKey=ConnectFlowLogGroupName,ParameterValue=/aws/connect/your-instance-alias \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION

aws cloudformation wait stack-create-complete --stack-name devops-agent-stack --region $REGION
```

Parameter keys are case-sensitive: `ConnectInstanceId`, not `ConnectInstanceID`. Drop the `ConnectFlowLogGroupName` line to skip the log-based detector.

Because the stack contains named IAM roles, `CAPABILITY_NAMED_IAM` is required.

### Stack names

The SNS topic, KMS alias, secret, Lambda, EventBridge rule, DLQ, dashboard, metric filter, and Agent Space log group all include the stack name. The IAM roles use the stack ID suffix instead. Deploying under a different name creates a second set, including a new unconfirmed email subscription.

## After deployment

Do these three steps in order. The pipeline can't start investigations until all three are done.

**1. Confirm the email subscription.** AWS sends a confirmation message to `AlarmNotificationEmail`. The subscription stays `PendingConfirmation` until you click the link.

**2. Create the webhook in the DevOps Agent console.** Open your Agent Space and create a webhook with the **HMAC** authentication type. Copy the webhook URL and the secret.

**3. Store them in Secrets Manager.** Update the secret `DevOpsAgent-WebhookSecret-<stack-name>` with this JSON:

```json
{
  "WEBHOOK_URL": "<your webhook URL>",
  "WEBHOOK_SECRET": "<your webhook secret>"
}
```

Then open the dashboard from the stack's `DashboardURL` output.

### Do this before the first alarm fires

The secret starts with placeholder values. While it does, the forwarder logs a warning and skips the webhook call. Alarms trigger on the transition into `ALARM`, so an alarm that fired while the secret was a placeholder won't retry on its own. It has to return to `OK` and trigger again.

From reading the forwarder code, it also caches the secret in memory between invocations, including the placeholder. If an alarm invoked the function before you updated the secret, the function may keep using the cached placeholder until its execution environment recycles. To force a refresh, change any setting on the function, for example:

```bash
aws lambda update-function-configuration \
  --function-name DevOpsAgent-AlarmForwarder-devops-agent-stack \
  --description "refresh $(date +%s)" --region us-east-1
```

I haven't tested this refresh behavior.

## Test it

**Option A: synthetic alarm (quick check)**

Set an alarm state by hand to confirm the EventBridge, Lambda, and webhook path works. The alarm name must start with `Connect-` or `DevOpsAgent-`.

```bash
aws cloudwatch set-alarm-state \
  --alarm-name Connect-HighMissedCalls \
  --state-value ALARM \
  --state-reason "Manual test of auto-investigation pipeline" \
  --region us-east-1
```

**Option B: real flow error (full check)**

Run a contact through a flow that has logging enabled and a block that fails at runtime. The error is written to the flow logs, the metric filter increments `FlowErrorCount`, `Connect-FlowErrors-FromLogs` goes to `ALARM`, and the investigation starts. This needs `ConnectFlowLogGroupName` set.

`test-broken-flow.ps1` (PowerShell) builds an isolated, deliberately broken test flow and drives chat contacts through it. It's a dry run by default, and the test flow is never attached to a phone number. Pass your own instance ID:

```powershell
./test-broken-flow.ps1 -InstanceId <your-instance-id>            # dry run
./test-broken-flow.ps1 -InstanceId <your-instance-id> -Execute -Count 10
./test-broken-flow.ps1 -InstanceId <your-instance-id> -Cleanup -Execute   # delete the test flow
```

After either test, open the DevOps Agent console. You should see an investigation titled `CloudWatch Alarm: <alarm name>`.

## Stack outputs

| Output | Description |
|---|---|
| `AgentSpaceId`, `AgentSpaceArn` | The Agent Space |
| `AgentSpaceRoleArn`, `OperatorRoleArn` | The Agent Space and operator app roles |
| `AlarmTopicArn` | SNS topic for alarm notifications |
| `DashboardURL` | Link to the CloudWatch dashboard |
| `LogGroupName` | Log group for Agent Space logs |
| `ConnectInstanceMonitored` | The Connect instance being monitored |
| `AlarmForwarderDLQUrl`, `AlarmForwarderDLQArn` | The dead-letter queue |

## Security

- **SNS topic:** encrypted with a customer-managed KMS key (rotation enabled) whose key policy lets CloudWatch publish. The topic policy denies publishes from other accounts.
- **Webhook secret:** encrypted with the AWS-managed key `alias/aws/secretsmanager`, not a customer-managed key.
- **Dead-letter queue:** encrypted with SQS-managed encryption (SSE-SQS), not a KMS key. Message bodies are encrypted at rest. Message metadata isn't.
- **IAM:** the Agent Space and operator roles can only be assumed by the DevOps Agent service from your account. The forwarder role can read one secret and send to one queue.
- **Lambda:** reserved concurrency of 10 limits the blast radius of an alarm storm. If `Infra-AlarmForwarderThrottles` fires, consider raising it.
- **Webhook calls:** each request is signed with HMAC-SHA256 over the timestamp and payload.

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.


## Clean up

Deleting the stack isn't enough. The template deliberately keeps the webhook secret and the Agent Space log group (`DeletionPolicy: Retain`), and Lambda creates its own log group outside the stack. Use the cleanup script to remove everything:

```bash
python3 cleanup_devops_agent_stack.py --stack-name devops-agent-stack            # dry run
python3 cleanup_devops_agent_stack.py --stack-name devops-agent-stack --execute
```

See [README-cleanup.md](README-cleanup.md) for details.

## Files

| File | Purpose |
|---|---|
| `devops-agent-stack.yaml` | The CloudFormation template |
| `cleanup_devops_agent_stack.py` | Removes the stack and everything it leaves behind |
| `README-cleanup.md` | Documentation for the cleanup script |
| `test-broken-flow.ps1` | Creates a broken test flow and drives contacts through it |

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
