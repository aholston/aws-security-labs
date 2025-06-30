# Phase 5 & 6: Detection of IAM Events via CloudTrail and EventBridge

## Overview

This phase adds real-time detection of suspicious IAM activity in the `prod-account`. It builds on the CloudTrail configuration by integrating Amazon EventBridge and Amazon SNS to trigger email alerts when high-risk IAM actions are performed.

This is directly aligned with Domain 4 of the AWS Certified Security – Specialty exam (IAM visibility and monitoring).

---

## CloudTrail Configuration

- **Trail name**: `OrgTrail`
- **Trail type**: Single-account (currently logging from `prod-account` only)
- **Destination S3 bucket**: `org-cloudtrail-logs-sec123` (owned by `security-account`)
- **Log file encryption**: Enabled using AWS KMS CMK (customer-managed key)
- **KMS key**: Created in `security-account` and configured with a cross-account policy to allow encryption from `prod-account`

---

## EventBridge Rule: Detect AttachUserPolicy

- **Account**: `prod-account`
- **Service**: Amazon EventBridge
- **Rule name**: `DetectIamPolicyAttachment`
- **Event pattern**:

```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["iam.amazonaws.com"],
    "eventName": ["AttachUserPolicy"]
  }
}
