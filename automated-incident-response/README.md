# Automated Incident Response to Suspicious IAM Activity

This lab builds on the "Advanced IAM with Federation and Detection" lab and focuses on Domain 1 of the AWS Certified Security – Specialty exam: **Incident Response**.

## Objective

Detect and automatically respond to suspicious IAM activity, such as unauthorized attachment of the `AdministratorAccess` policy in the `prod-account`. The system will alert security and optionally remove the policy or disable the IAM user.

## Architecture

```
CloudTrail (prod-account)
      ↓
EventBridge Rule (suspicious IAM events)
      ↓
SNS Topic (security-alerts)
      ↓
Lambda Function (remediates + alerts)
```

## Lab Phases

* **Phase 1**: CloudTrail and EventBridge Setup
* **Phase 2**: Lambda for Automated Response
* **Phase 3**: Testing and Verification
* **Phase 4** (Optional): Log Forensics + Slack/Email alert extensions

This lab assumes you've already configured:

* An AWS Organization
* Federated access via IAM Identity Center
* A `prod-account` with CloudTrail enabled

We will now move step-by-step through building the automated incident response system.
