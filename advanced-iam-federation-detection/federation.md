# Phase 2: Federated Identity and Access Control with IAM Identity Center

## Overview

This phase implements centralized, role-based access control using AWS IAM Identity Center (formerly AWS SSO) at the organization level. IAM Identity Center was enabled directly from the AWS Organizations management account and is currently being managed from that account.

## Directory Configuration

- **Identity source**: AWS IAM Identity Center internal directory
- **Provisioning method**: Manual (direct user creation)
- **Authentication**: Username/password via IAM Identity Center portal
- **Portal URL**: Provided by IAM Identity Center instance configuration

## User and Group

- **Group name**: Developers
- **User name**: devuser
- **Email**: you+devuser@example.com
- **Assigned group**: Developers

## Permission Set

- **Name**: DevAccessProd
- **Session duration**: 1 hour
- **Policy**: Custom inline policy with limited read-only access:

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
