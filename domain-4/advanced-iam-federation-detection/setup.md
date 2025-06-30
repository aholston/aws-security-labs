# Phase 1: AWS Organizations and IAM Setup

## AWS Organization Details

An AWS Organization was created from the management (root) account with "All Features" enabled. The following member accounts were created using the "Create an AWS Account" flow within the Organizations console:

### Accounts

| Account Name       | Account ID     | Purpose                                  |
|--------------------|----------------|------------------------------------------|
| security-account   | <ID>           | Centralized logging and alerts           |
| prod-account       | <ID>           | Sensitive resources (target for federation) |
| dev-account        | <ID>           | Federation and IAM Identity Center setup |

## OrganizationAccountAccessRole

Each new account includes an IAM role named `OrganizationAccountAccessRole`, which is automatically trusted by the management account. This allows delegated administrative access via role assumption from the root account.

## IAM Admin Users

Each account has an IAM admin user with full administrator permissions and multi-factor authentication (MFA) enabled. These users are used for all administrative tasks instead of the root account.

| Account          | IAM Admin Username | MFA Enabled | Notes                             |
|------------------|--------------------|-------------|-----------------------------------|
| security-account | security-admin     | Yes         | Used to manage CloudTrail and EventBridge |
| prod-account     | prod-admin         | Yes         | Manages IAM roles and policies         |
| dev-account      | dev-admin          | Yes         | Used for IAM Identity Center configuration |

## Additional Notes

- Root user passwords were initialized using the "Forgot Password" flow since they were not set at account creation.
- Root users have been secured with MFA.
- IAM users were created manually through the AWS Console under each account’s root user.
- IAM Identity Center will be configured in the `dev-account` in the next phase.
# Environment Setup: Multi-Account AWS Lab Environment

This document outlines the foundational setup of a secure, multi-account AWS environment used in the *Advanced IAM with Federation and Detection* lab. It includes AWS Organizations configuration, IAM admin user setup, and account structure.

---

## Phase 1: AWS Organizations and Account Structure

### 🏢 AWS Organization

- **Enabled** from the **root management account**
- **Feature set**: All features (not consolidated billing-only)

### 🧾 Account Structure

| Account Name       | Description                                  |
|--------------------|----------------------------------------------|
| `management`       | Organization root account (central admin)    |
| `security-account` | Reserved for central logging and detection   |
| `prod-account`     | Target environment for federated IAM access  |
| `dev-account`      | Placeholder for future testing               |

---

## Phase 1.1: Account Creation

- From the management account, created the above accounts using the **"Create account"** workflow in AWS Organizations.
- AWS assigns each new account the default role:  
  `OrganizationAccountAccessRole`
- No delegated administrator was configured initially.

---

## Phase 1.2: IAM Admin User Setup

In each member account (`security-account`, `prod-account`, `dev-account`):

- Signed in via **management account → Switch role** into `OrganizationAccountAccessRole`
- Created one IAM user per account:

| Account          | IAM Admin User     |
|------------------|--------------------|
| `security-account` | `security-admin` |
| `prod-account`     | `prod-admin`     |
| `dev-account`      | `dev-admin`      |

### 🛡️ IAM Admin Permissions

- Attached AWS-managed policy: `AdministratorAccess`
- Set up **MFA** on each admin user
- Created long-term credentials (access keys) for CLI access when needed

---

## Phase 1.3: Initial Security Controls

- Verified MFA setup on all **root users**
- Disabled root API keys (if present)
- Stored all IAM credentials securely with MFA required for sign-in

---

## Optional: CLI Profile Configuration

Configured AWS CLI named profiles for each admin user:

```bash
aws configure --profile prod-admin
aws configure --profile security-admin
aws configure --profile dev-admin
