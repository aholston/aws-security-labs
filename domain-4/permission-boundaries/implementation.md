# Implementation Guide: Permission Boundaries with Break-Glass Access

## Phase 1: Permission Boundary Policy Creation

### Step 1.1: Create Boundary Policy in prod-account

**As prod-admin:**

```bash
aws iam create-policy \
  --policy-name SimpleDeveloperBoundary \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowBasicEC2Operations",
        "Effect": "Allow",
        "Action": [
          "ec2:DescribeInstances",
          "ec2:DescribeInstanceStatus",
          "ec2:RunInstances"
        ],
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "ec2:InstanceType": ["t2.micro"]
          }
        }
      },
      {
        "Sid": "AllowBasicS3Operations",
        "Effect": "Allow",
        "Action": [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:PutObject"
        ],
        "Resource": [
          "arn:aws:s3:::dev-*",
          "arn:aws:s3:::dev-*/*"
        ]
      },
      {
        "Sid": "AllowEmergencyRoleAssumption",
        "Effect": "Allow",
        "Action": ["sts:AssumeRole"],
        "Resource": [
          "arn:aws:iam::ACCOUNT-ID:role/EmergencyBreakGlassRole"
        ]
      },
      {
        "Sid": "DenyIAMOperations",
        "Effect": "Deny",
        "Action": ["iam:*"],
        "Resource": "*"
      }
    ]
  }'
```

## Phase 2: Emergency Break-Glass Role

### Step 2.1: Create Emergency Role

```bash
aws iam create-role \
  --role-name EmergencyBreakGlassRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:userid": "AROA5ZMN6PSGXRZG3DWXS:boundaryuser",
          "sts:ExternalId": "emergency-break-glass-2025"
        }
      }
    }]
  }' \
  --max-session-duration 14400
```

### Step 2.2: Attach Emergency Permissions

```bash
aws iam put-role-policy \
  --role-name EmergencyBreakGlassRole \
  --policy-name EmergencyBreakGlassPermissions \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "EmergencyInfrastructureAccess",
        "Effect": "Allow",
        "Action": [
          "ec2:*", "rds:*", "s3:*", "lambda:*",
          "cloudformation:*", "logs:*", "cloudwatch:*",
          "route53:*", "elasticloadbalancing:*", "autoscaling:*"
        ],
        "Resource": "*"
      },
      {
        "Sid": "EmergencyReadOnlyAccess",
        "Effect": "Allow",
        "Action": [
          "iam:List*", "iam:Get*", "organizations:List*",
          "ce:GetCostAndUsage", "ce:GetDimensionValues"
        ],
        "Resource": "*"
      },
      {
        "Sid": "DenyDestructiveOperations",
        "Effect": "Deny",
        "Action": [
          "ec2:TerminateInstances", "rds:DeleteDBInstance",
          "s3:DeleteBucket", "lambda:DeleteFunction",
          "cloudformation:DeleteStack"
        ],
        "Resource": "*"
      },
      {
        "Sid": "DenyDangerousIAMOperations",
        "Effect": "Deny",
        "Action": [
          "iam:CreateUser", "iam:DeleteUser", "iam:CreateRole",
          "iam:DeleteRole", "iam:AttachUserPolicy", "iam:AttachRolePolicy"
        ],
        "Resource": "*"
      }
    ]
  }'
```

## Phase 3: Identity Center Configuration

### Step 3.1: Create Federated User

**In security-account (Identity Center):**

1. **Users** → **Create user**
   - Username: `boundaryuser`
   - Attributes: Department=Engineering, CostCenter=CC-1001, Title=Developer
   - Add to `Developers` group

### Step 3.2: Create Permission Set with Boundary

1. **Permission sets** → **Create permission set**
   - Name: `SimpleBoundedDeveloper`
   - Permission boundary: `SimpleDeveloperBoundary` (by name)
   - Inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:*"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["iam:ListRoles", "iam:GetRole"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["sts:AssumeRole"],
      "Resource": "arn:aws:iam::ACCOUNT-ID:role/EmergencyBreakGlassRole"
    }
  ]
}
```

### Step 3.3: Assign Permission Set

1. **AWS accounts** → **prod-account** → **Assign users or groups**
2. **Groups** → Select `Developers`
3. **Permission sets** → Select `SimpleBoundedDeveloper`

## Phase 4: Automated Revocation System

### Step 4.1: Create Lambda Function

```python
import json
import boto3
import logging
from datetime import datetime, timedelta

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    iam = boto3.client('iam')
    events = boto3.client('events')
    
    try:
        detail = event['detail']
        assumed_role_arn = detail['responseElements']['assumedRoleUser']['arn']
        source_ip = detail['sourceIPAddress']
        user_identity = detail['userIdentity']
        event_time = detail['eventTime']
        
        logger.info(f"Emergency break-glass role assumed: {assumed_role_arn}")
        
        # Calculate revocation time (4 hours from now)
        revocation_time = datetime.utcnow() + timedelta(hours=4)
        rule_name = f"RevokeEmergencyAccess-{int(datetime.utcnow().timestamp())}"
        
        # Schedule the revocation
        events.put_rule(
            Name=rule_name,
            ScheduleExpression=f"at({revocation_time.strftime('%Y-%m-%dT%H:%M:%S')})",
            Description=f"Auto-revoke emergency access granted at {event_time}",
            State='ENABLED'
        )
        
        # Add this Lambda as target for the scheduled rule
        events.put_targets(
            Rule=rule_name,
            Targets=[{
                'Id': '1',
                'Arn': context.invoked_function_arn,
                'Input': json.dumps({
                    'action': 'revoke',
                    'rule_name': rule_name,
                    'original_assumption_time': event_time
                })
            }]
        )
        
        logger.info(f"Emergency access scheduled for revocation at {revocation_time}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Emergency access logged and revocation scheduled',
                'revocation_time': revocation_time.isoformat()
            })
        }
        
    except Exception as e:
        logger.error(f"Error processing emergency access event: {str(e)}")
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

### Step 4.2: Create EventBridge Rule

```bash
aws events put-rule \
  --name DetectEmergencyBreakGlassAccess \
  --event-pattern '{
    "source": ["aws.sts"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventSource": ["sts.amazonaws.com"],
      "eventName": ["AssumeRole"],
      "requestParameters": {
        "roleArn": ["arn:aws:iam::ACCOUNT-ID:role/EmergencyBreakGlassRole"]
      }
    }
  }'

aws events put-targets \
  --rule DetectEmergencyBreakGlassAccess \
  --targets "Id"="1","Arn"="arn:aws:lambda:REGION:ACCOUNT-ID:function:EmergencyBreakGlassRevocation"
```

## Phase 5: Testing and Validation

### Step 5.1: Configure CLI Profile

```bash
aws configure sso --profile boundaryuser
# Follow prompts to authenticate via Identity Center
# Select prod-account and SimpleBoundedDeveloper role
```

### Step 5.2: Test Permission Boundary

```bash
# Should work - within boundary limits
aws ec2 run-instances \
  --image-id ami-12345 \
  --instance-type t2.micro \
  --profile boundaryuser

# Should fail - outside boundary limits  
aws ec2 run-instances \
  --image-id ami-12345 \
  --instance-type m5.large \
  --profile boundaryuser

# Should fail - denied by boundary
aws iam create-role --role-name TestRole --profile boundaryuser
```

### Step 5.3: Test Emergency Access

```bash
# Assume emergency role
aws sts assume-role \
  --role-arn "arn:aws:iam::ACCOUNT-ID:role/EmergencyBreakGlassRole" \
  --role-session-name "emergency-$(date +%s)" \
  --external-id "emergency-break-glass-2025" \
  --profile boundaryuser

# Export credentials and test elevated access
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Should now work - broader permissions
aws ec2 run-instances --instance-type m5.large
aws iam list-roles

# Should still fail - protected by deny statements
aws ec2 terminate-instances --instance-ids i-12345
aws iam create-role --role-name ShouldFail
```

### Step 5.4: Verify Automation

```bash
# Check Lambda logs for automation trigger
aws logs filter-log-events \
  --log-group-name /aws/lambda/EmergencyBreakGlassRevocation \
  --start-time $(date -d '10 minutes ago' +%s)000

# Check EventBridge rules for scheduled revocation
aws events list-rules --name-prefix RevokeEmergencyAccess
```

## Troubleshooting Guide

### Common Issues

**Permission Boundary Not Applied**
- Verify boundary policy exists in target account (prod-account)
- Check permission set references correct policy name
- Ensure permission set is provisioned to account

**Role Assumption Failures**
- Verify user ID in trust policy matches actual federated user ID
- Remove MFA requirements for Identity Center federated sessions
- Check External ID matches exactly between trust policy and CLI command

**Automation Not Triggering**
- Ensure CloudTrail is enabled and logging to EventBridge
- Verify EventBridge rule pattern matches role assumption events
- Check Lambda function has correct IAM permissions

### Validation Commands

```bash
# Check boundary attachment
aws iam get-role --role-name AWSReservedSSO_SimpleBoundedDeveloper_[suffix]

# Verify policy versions
aws iam list-policy-versions --policy-arn arn:aws:iam::ACCOUNT:policy/SimpleDeveloperBoundary

# Check current session identity
aws sts get-caller-identity --profile boundaryuser

# Monitor CloudTrail events
aws logs filter-log-events \
  --log-group-name CloudTrail/OrgTrail \
  --filter-pattern "AssumeRole EmergencyBreakGlassRole"
```