# Implementing a Budget Circuit Breaker Using AWS Budgets and IAM Identity Center

Read the full AWS blog post here: [Implementing a Budget Circuit Breaker Using AWS Budgets and IAM Identity Center](https://aws.amazon.com/blogs/placeholder)

## Solution Architecture

![Budget Circuit Breaker Architecture Diagram](./budget-blog-diagram.png)

This solution leverages several AWS services to provide automated cost protection across multi-account environments:

- [**AWS Budgets**](https://aws.amazon.com/aws-cost-management/aws-budgets/) - Monitors actual spend against defined monthly thresholds and triggers notifications when exceeded.

- [**Amazon SNS**](https://aws.amazon.com/sns/) - Receives budget threshold notifications and fans out to email subscribers and the Lambda function.

- [**AWS Lambda**](https://aws.amazon.com/lambda/) - Executes the access revocation logic, resolving Identity Center groups and removing permission set assignments.

- [**AWS IAM Identity Center**](https://aws.amazon.com/iam/identity-center/) - Manages workforce access to AWS accounts; the circuit breaker revokes or downgrades group assignments here.

- [**Amazon SQS**](https://aws.amazon.com/sqs/) - Dead-letter queue captures failed Lambda invocations for retry and observability.

- [**Amazon CloudWatch**](https://aws.amazon.com/cloudwatch/) - Alarm monitors the DLQ for messages, alerting when revocation attempts fail.

The workflow begins when AWS Budgets detects that actual spend has crossed a defined threshold. At 80%, an email alert notifies your team. At 100%, Budgets publishes to an SNS topic which triggers a Lambda function. The function resolves IAM Identity Center group IDs, lists all permission set assignments for those groups across target accounts, and revokes access — acting as a circuit breaker that stops further resource creation.

## Deployment Instructions

### 1. Download the CloudFormation Template

Download the CloudFormation YAML template file from the provided source.

📥 **[Download CloudFormation Template](./circuit-breaker-template_final-draft.yaml)**

### 2. Deploy the CloudFormation Stack

1. Navigate to the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/) in your management account.
2. Click on "Create stack" → "With new resources (standard)".
3. Upload the downloaded template file and click "Next".

### 3. Configure Stack Parameters

**Stack name**: Enter a name for your stack (e.g., `budget-circuit-breaker`).

#### Budget Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| **BudgetAmount** | Monthly budget threshold in USD | `100` |
| **Threshold80** | First alert threshold percentage (email only) | `80` |
| **Threshold100** | Second threshold percentage (triggers revocation) | `100` |
| **EmailRecipients** | Comma-separated list of email addresses for budget alerts | `user@example.com` |

#### IAM Identity Center Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| **InstanceArn** | Your IAM Identity Center instance ARN | `arn:aws:sso:::instance/ssoins-123456789012` |
| **TargetGroups** | Comma-separated list of Identity Center group display names to revoke access from | `group1` |
| **TargetAccounts** | Comma-separated list of AWS account IDs to monitor and revoke access from | `123456789012` |

> **Tip:** To find your IAM Identity Center instance ARN, run:
> ```bash
> aws sso-admin list-instances --query "Instances[0].InstanceArn" --output text
> ```

#### Action Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| **BudgetAction** | Action to take when threshold is exceeded | `revoke-full-access` |
| **ReadOnlyPermissionSetArn** | ARN of the permission set for read-only access (required when action is `convert-to-read-only`) | *(empty)* |

- **revoke-full-access** — removes all permission set assignments for the target groups.
- **convert-to-read-only** — assigns a read-only permission set before revoking existing access, so teams retain visibility without the ability to create resources.

#### Deployment Region

| Parameter | Description | Default |
|-----------|-------------|---------|
| **DeploymentRegion** | Region where SNS and Lambda will be deployed | `ca-central-1` |

Allowed values: `us-east-1`, `us-west-2`, `ap-southeast-1`, `ap-northeast-1`, `eu-west-1`, `eu-central-1`, `ca-central-1`

### 4. Complete Stack Creation

1. Click "Next" to proceed to the stack options page.
2. Configure any additional stack options as desired.
3. Click "Next" to proceed to the review page.
4. Review your configuration and scroll to the bottom of the page.
5. Check the acknowledgement box confirming AWS CloudFormation might create IAM resources with custom names.
6. Click "Submit" to launch the deployment.

### 5. Confirm the SNS Email Subscription

After deployment, each email address in `EmailRecipients` will receive a subscription confirmation email. Click **Confirm subscription** in that email to start receiving budget alerts.

## Next Steps

After the stack creation completes successfully, click on the "Outputs" tab of your stack in the CloudFormation console. Here you'll find:
- SNS Topic ARN
- Lambda Function ARN
- Budget Name

## Troubleshooting

If stack creation fails, check the "Events" tab in the CloudFormation console for error messages that can help diagnose the issue.

Common issues:
- **IAM Identity Center instance ARN is incorrect** — verify using the CLI command in Step 3.
- **Target group names don't match** — group display names are case-sensitive; confirm them in the Identity Center console.
- **DLQ alarm fires** — check the Lambda CloudWatch logs for the specific revocation error.

## Solution Walkthrough

Head back to the [Implementing a Budget Circuit Breaker Using AWS Budgets and IAM Identity Center](https://aws.amazon.com/blogs/placeholder) AWS blog for a walkthrough of the solution, future considerations, and clean up.

## Clean Up

To remove all resources created by this solution, delete the stack from the CloudFormation console or run:

```bash
aws cloudformation delete-stack --stack-name budget-circuit-breaker
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
