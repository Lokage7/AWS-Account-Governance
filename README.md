# AWS Account Governance — Repo

This repository provides a **comprehensive, step-by-step implementation** of AWS account governance and security best practices using native AWS services. It includes CloudFormation templates, configuration examples, and documentation for setting up an enterprise-ready AWS environment with strong compliance, monitoring, and cost control foundations.

---

## Repo structure

```
aws-account-governance/
├─ README.md                 # This file
├─ LICENSE
├─ templates/
│  ├─ secure-bucket.yml      # S3 bucket for CloudTrail & AWS Config
│  ├─ config-rules.yml       # AWS Config managed rules and remediation (CFN)
│  └─ iam-identity-center.md  # notes + IAM permission-set examples
├─ scripts/
│  ├─ enable-security-hub.sh # CLI helper (optional)
│  └─ cleanup.sh             # tear-down helper to remove resources
└─ docs/
   ├─ verification.md        # testing & validation steps
   └─ cost-awareness.md      # budgets + Cost Explorer tips
```

---

## README — Quick overview

### Purpose

Provide a tracked, repeatable implementation of an AWS governance baseline using native services:

* IAM & IAM Identity Center (SSO)
* CloudTrail & CloudWatch (logging & visibility)
* AWS Config (compliance rules via CloudFormation)
* Security Hub (centralized findings)
* Budgets + Cost Explorer (cost awareness)

This repo contains documentation and example templates to get started quickly.

### Prerequisites

* AWS account with root access for initial setup
* AWS CLI configured with an admin profile (optional but recommended)
* `aws` CLI v2 installed
* Basic familiarity with IAM, CloudFormation, and AWS Console navigation

---

## Implementation steps

### Step 1 — Identity governance

1. Enable MFA on the root account (manual in console). Document which device is used and store recovery codes safely.
2. Enable IAM Identity Center (SSO) if using Organizations. Use groups and Permission Sets rather than long-lived IAM users.
3. Create permission sets such as `AdministratorAccess` and `ReadOnlyAccess`. Assign to groups.
4. Enable IAM Access Analyzer.

**Example notes:** See `templates/iam-identity-center.md` for permission-set JSON examples and CLI commands.

---

### Step 2 — Logging & monitoring

1. Create CloudTrail trail (All Regions) and send logs to an encrypted S3 bucket.
2. Configure CloudTrail to send to CloudWatch Logs; create a Log Group like `/aws/cloudtrail/logs`.
3. Create CloudWatch Metric Filters & Alarms for important events (root login attempts, console sign-in failures, IAM changes).
4. Send alarm notifications via SNS (email or webhook to Slack).

**Secure bucket CFN (templates/secure-bucket.yml)**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Secure S3 bucket for CloudTrail and AWS Config
Resources:
  GovernanceBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::AccountId}-governance-logs'
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      OwnershipControls:
        Rules:
          - ObjectOwnership: BucketOwnerPreferred
  BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref GovernanceBucket
      PolicyDocument:
        Statement:
          - Effect: Deny
            Principal: "*"
            Action: s3:PutObject
            Resource: !Sub '${GovernanceBucket.Arn}/*'
            Condition:
              StringEquals:
                s3:x-amz-acl: "public-read"
```

> Note: Replace the BucketName with your naming standard. Production usage should include encryption (SSE-KMS) and a KMS key resource in CFN.

---

### Step 3 — AWS Config rules via CloudFormation

Use CloudFormation to deploy baseline rules such as:

* `iam-password-policy` (password length, complexity)
* `cloudtrail-enabled` (CloudTrail existence)
* `s3-bucket-public-read-prohibited` / `s3-bucket-public-write-prohibited`
* `mfa-enabled-for-iam-users`

**Example (partial) - templates/config-rules.yml**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS Config Managed Rules
Resources:
  PasswordPolicyRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-password-policy
      Source:
        Owner: AWS
        SourceIdentifier: IAM_PASSWORD_POLICY
  CloudTrailCheck:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloudtrail-enabled
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED
```

You can add remediation actions or runbooks that trigger Lambdas for automated remediation.

---

### Step 4 — Security Hub

1. Enable Security Hub and selected standards: AWS Foundational Security Best Practices and CIS AWS Benchmark.
2. Enable GuardDuty, Inspector, and IAM Access Analyzer as integrations.
3. Triage findings by severity and compliance status and track remediation work.

**CLI helper (scripts/enable-security-hub.sh)** — optional wrapper that calls `aws securityhub enable-security-hub` and enables standards.

---

### Step 5 — Cost awareness

1. Enable Cost Explorer.
2. Create a Cost Budget (e.g., MonthlyBudget at $50) with alerts at 50%, 80%, and 100% via SNS.
3. Configure forecast reports.

See `docs/cost-awareness.md` for step-by-step guidance and sample AWS CLI commands.

---

## Testing & validation

Document the validation steps in `docs/verification.md`. Example tests:

* Attempt to sign in without MFA and confirm blocked access.
* Trigger a CloudTrail event (e.g., change an IAM policy) and confirm CloudWatch alarm and SNS notification.
* Run AWS Config compliance checks and review pass/fail.
* Check Security Hub for findings and verify severity tagging.

---

## Cleanup

`cleanup.sh` should remove resources created during the lab (trails, buckets, config rules, alarms, SNS topics). Be careful — removing S3 buckets with objects requires additional steps.

## Next Steps

Here are some meaningful next steps to extend this project:

1. **Expand CloudFormation templates** — Add encryption (SSE-KMS), custom Config rules, and AWS Organizations support.
2. **Integrate GitHub Actions** — Automate stack deployment to a test account on pull request merge.
3. **Add Security Hub automation** — Create Lambda remediations or EventBridge rules to handle findings automatically.
4. **Document cost management** — Add dashboards in AWS Budgets and Cost Explorer; export reports to S3.
5. **Publish a walkthrough video or blog post** — Demonstrate deployment and discuss lessons learned.
6. **Contribute to open source** — Invite collaborators, track enhancements, and encourage feedback.

---

*Last updated: October 2025*
