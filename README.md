## Amazon Connect Customer with AWS DevOps Agent

CloudFormation template accompanying the AWS Contact Center blog - Reduce MTTR in Amazon Connect Customer with AWS DevOps Agent

It deploys an AWS DevOps Agent Agent Space, CloudWatch alarms for Amazon Connect and DevOps Agent, and an alarm → Lambda → webhook pipeline that automatically starts DevOps Agent investigations.

## Prerequisites
- An existing Amazon Connect instance
- (Recommended) Connect flow logging enabled, with a *Set logging behavior* block in your flows

## Deploy
```bash
aws cloudformation deploy \
  --region REGION \
  --stack-name devops-agent-connect \
  --template-file devops-agent-stack.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    AlarmNotificationEmail=you@example.com \
    ConnectInstanceId=<connect-instance-id> \
    ConnectFlowLogGroupName=/aws/connect/<instance-alias>
```

## After deployment
1. Confirm the SNS subscription email.
2. Create a webhook in your DevOps Agent Agent Space, then update the
   `DevOpsAgent-WebhookSecret-<stack-name>` secret with its `WEBHOOK_URL` and
   `WEBHOOK_SECRET`. Investigations won't start until you do this.

## Notes
- Alarms named `Connect-*` and `DevOpsAgent-*` trigger investigations. `Infra-*` alarms only send email.
- For flow errors, set `ConnectFlowLogGroupName` *or* `MonitoredContactFlowName`, not both.
- Deploying this stack incurs charges (CloudWatch alarms, KMS, Lambda, DevOps Agent usage).

## Clean up
Delete the stack, then delete the retained Secrets Manager secret.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

