# Advanced IAM with Federation and Detection

This lab simulates a secure, multi-account AWS environment using centralized identity federation and detection of IAM-related activity. It aligns with best practices for AWS Organizations and IAM Identity Center, and targets learning objectives from Domain 4 (Identity and Access Management) of the AWS Certified Security – Specialty exam.

The lab emphasizes:
- Organization-level account and role structure
- Centralized federated access via IAM Identity Center
- Scoped permission sets and least-privilege principles
- Portfolio-quality documentation and reproducibility

---

## Phase 1: AWS Organizations and IAM Admin Setup

- AWS Organization created from the **management account** with "All Features" enabled
- Three member accounts created via AWS Organizations:
  - `security-account` (planned future use for centralized logging and detection)
  - `prod-account` (target account for federated access)
  - `dev-account` (not used in this lab)
- IAM admin users provisioned in each account:
  - `security-admin`
  - `prod-admin`
  - `dev-admin`
- MFA enabled on all root users and IAM admin users
- All accounts are accessible via long-term IAM user credentials secured with MFA

---

## Phase 2: Federated Identity and Access Control

- IAM Identity Center was **enabled as an organization-level instance** in the **management account**
- Internal Identity Center directory was used (not external IdP)
- Created user `devuser` and group `Developers`
- Created permission set `DevAccessProd` with scoped read-only permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    }
  ]
}
