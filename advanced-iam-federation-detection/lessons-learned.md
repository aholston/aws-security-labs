# Lessons Learned

## IAM Identity Center (Formerly AWS SSO)

### 1. Organization Instance vs. Account Instance

- **Mistake**: Initially enabled IAM Identity Center in a standalone member account (`dev-account`) thinking it could be used to manage federated access across accounts.
- **Lesson**: Only an **organization instance** (enabled from the management/root account) supports multi-account access. This is the only production-ready setup AWS recommends.
- **Reference**: [AWS Docs – IAM Identity Center instance types](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)

### 2. Delegated Administrator

- **Clarified**: In an org-level IAM Identity Center setup, you can (and should) delegate administration to a specific member account (e.g., `security-account`).
- This is configured from the **management account** under **AWS Organizations > Services > Delegated Administrators**.

### 3. Identity Source Options

- **Default (IAM Identity Center directory)** allows internal user/group creation and is great for labs and small orgs.
- **External IdPs (e.g., Google Workspace, Okta)** can be integrated for centralized authentication using SAML.
- **Important Note**: Free Gmail accounts are not supported. You need a custom domain managed by Google Workspace to use it as an identity provider.

### 4. Cleanup Process

- Deleting an IAM Identity Center instance requires:
  - Removing all users, groups, permission sets, and assignments
  - Then deleting the instance from the **Settings** page
- AWS won’t let you delete the instance unless all resources are removed.

### 5. Region Selection Matters

- IAM Identity Center is a **regional service with a single home region**.
- All administration (user creation, permission set assignment) must happen in the region it was enabled in (e.g., `us-east-1`).
- Users can access resources in any region regardless of Identity Center's home region.

### 6. Portal Behavior

- Federated users sign in via a special **access portal URL** (`*.awsapps.com/start`)
- Once authenticated, they see all accounts and permission sets they’ve been assigned
- This interface replaces the need to manually distribute role ARNs or console links
