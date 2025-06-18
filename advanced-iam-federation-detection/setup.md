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
