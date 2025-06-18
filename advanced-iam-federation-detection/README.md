# Advanced IAM with Federation and Detection

This lab simulates a secure multi-account AWS environment using identity federation and real-time detection of IAM-related activity. It is designed to reflect real-world scenarios relevant to AWS Security Specialty Domain 4 (Identity and Access Management).

## Phase 1: AWS Organizations and IAM Admin Setup

- AWS Organization created with "All Features" enabled
- Three child accounts created:
  - security-account
  - prod-account
  - dev-account
- IAM admin users provisioned in each account:
  - security-admin
  - prod-admin
  - dev-admin
- MFA configured for all IAM admin users and root users
