# Advanced IAM with Federation and Detection

This lab simulates a secure, multi-account AWS environment using centralized identity federation and detection of IAM-related activity. It aligns with best practices for AWS Organizations and IAM Identity Center, and targets learning objectives from Domain 4 (Identity and Access Management) of the AWS Certified Security – Specialty exam.

The lab emphasizes:
- Organization-level account and role structure
- Centralized federated access via IAM Identity Center
- Scoped permission sets and least-privilege principles
- Portfolio-quality documentation and reproducibility

---

## Phase 1: AWS Organizations and IAM Admin Setup

- AWS Organization created from the management (root) account with "All Features" enabled
- Three member accounts created via AWS Organizations:
  - `security-account` (future use for CloudTrail logging and event detection)
  - `prod-account` (target for cross-account access)
  - `dev-account` (not used in current phase)
- IAM admin users provisioned in each account:
  - `security-admin`
  - `prod-admin`
  - `dev-admin`
- MFA enabled on all root users and IAM admin users
- All accounts are accessible via the IAM admin users using long-term credentials secured with MFA

---

## Phase 2: Federated Identity and Access Control

- IAM Identity Center was enabled as an **organization-level instance** in the management account
- `security-account` was registered as the **delegated administrator** for IAM Identity Center using the AWS CLI:
  ```bash
  aws organizations register-delegated-administrator \
    --account-id <security-account-id> \
    --service-principal sso.amazonaws.com

    ---

## Phase 3: Permissions Boundaries – Limitations with IAM Identity Center

### Goal

We attempted to apply a permissions boundary to an `AWSReservedSSO_*` role provisioned by IAM Identity Center to demonstrate constraint-based access control.

### Outcome

❌ **Failed**: AWS blocks modification of `AWSReservedSSO_*` roles. These roles are fully managed by IAM Identity Center and cannot be edited — including attaching permission boundaries.

### Why

While IAM roles generally support permissions boundaries, roles created by IAM Identity Center are:
- Owned and maintained by AWS
- Automatically replaced or re-created when changes occur in permission sets
- Locked down to prevent manual modification (even from account admins)

### Takeaway

This is **by design**. AWS Identity Center expects access to be controlled through:
- Permission sets
- Scoped permissions
- Service control policies (SCPs) at the org/OUs level
- Logging and detection for misuse

### Alternative Plan (Phase 4)

Instead of permission boundaries, we’ll pivot to:
- Assigning over-permissive access intentionally
- Detecting unusual or risky behavior via CloudTrail and EventBridge

This approach better reflects how many orgs operate in federated environments using centralized access controls and detection mechanisms.


