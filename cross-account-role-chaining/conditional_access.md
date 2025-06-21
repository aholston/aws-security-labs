# Advanced Conditional Access Controls

## Overview

This document extends the cross-account role chaining lab by implementing advanced conditional access controls. These conditions add additional security layers to role assumption and are heavily tested in AWS Security Specialty Domain 4 scenarios.

---

## Time-Based Access Control

### Implementation

Time-based conditions restrict role assumption to specific time windows, commonly used for:
- Business hours enforcement
- Maintenance window restrictions  
- Emergency access time limits
- Compliance requirement adherence

### Trust Policy Configuration

**Basic time window restriction:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringLike": {
          "aws:userid": "AROA*:devuser"
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2025-06-21T18:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2025-06-21T23:00:00Z"
        }
      }
    }
  ]
}
```

**Business hours restriction (Monday-Friday, 9 AM - 5 PM EST):**
```json
"Condition": {
  "StringLike": {
    "aws:userid": "AROA*:devuser"
  },
  "DateGreaterThan": {
    "aws:CurrentTime": "1970-01-05T14:00:00Z"
  },
  "DateLessThan": {
    "aws:CurrentTime": "1970-01-05T22:00:00Z"
  },
  "ForAllValues:StringEquals": {
    "aws:RequestedRegion": "us-east-1"
  }
}
```

### Key Implementation Details

**UTC Timezone Requirement:**
- All AWS time conditions use UTC (Coordinated Universal Time)
- Local business hours must be converted to UTC
- Example: 9 AM EST = 14:00 UTC, 5 PM EST = 22:00 UTC

**Time Condition Operators:**
- `DateGreaterThan`: Access allowed after specified time
- `DateLessThan`: Access allowed before specified time  
- `DateGreaterThanEquals`/`DateLessThanEquals`: Inclusive comparisons

**ISO 8601 Format:**
- Required format: `YYYY-MM-DDTHH:MM:SSZ`
- Example: `2025-06-21T18:00:00Z` = June 21, 2025 at 6:00 PM UTC

### Testing and Validation

**Test outside allowed window:**
```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::ACCOUNT-ID:role/CrossAccountIntermediateRole" \
  --role-session-name "time-test" \
  --profile devuser
```

**Expected error:**
```
An error occurred (AccessDenied) when calling the AssumeRole operation: 
User: arn:aws:sts::ACCOUNT-ID:assumed-role/AWSReservedSSO_DevAccessProd_.../devuser 
is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::ACCOUNT-ID:role/CrossAccountIntermediateRole
```

**Critical observation:** AWS error messages are intentionally generic and do not specify which condition failed (time, IP, MFA, etc.) for security reasons.

### Security Benefits

**Reduced Attack Window:**
- Limits potential compromise to business hours
- Prevents after-hours unauthorized access
- Reduces blast radius of credential theft

**Compliance Alignment:**
- Supports PCI DSS time-based access requirements
- Meets SOX controls for administrative access timing
- Aligns with corporate security policies

**Audit and Monitoring:**
- CloudTrail logs show attempted access outside allowed windows
- EventBridge rules can trigger alerts for time violations
- Clear pattern identification for security analysis

---

## Error Message Analysis

### Generic Access Denied Pattern

AWS provides consistent error messages regardless of which condition fails:
```
User: [identity] is not authorized to perform: sts:AssumeRole on resource: [role-arn]
```

This applies to failures from:
- Time-based conditions
- IP address restrictions  
- MFA requirements
- External ID mismatches
- Principal mismatches

### Troubleshooting Methodology

**1. Systematic Condition Testing:**
- Remove conditions one by one
- Test with minimal trust policy
- Add conditions incrementally

**2. CloudTrail Analysis:**
```bash
aws logs filter-log-events \
  --log-group-name CloudTrail/OrgTrail \
  --filter-pattern "AssumeRole AccessDenied" \
  --start-time 1640995200000
```

**3. Policy Simulator:**
- Use IAM Policy Simulator for condition testing
- Specify exact context (time, IP, MFA status)
- Validate policy logic before deployment

**4. Context Verification:**
- Check current UTC time: `date -u`
- Verify IP address: `curl ifconfig.me`
- Confirm MFA status in session

---

## Advanced Time Patterns

### Recurring Business Hours
```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  },
  "DateGreaterThan": {
    "aws:CurrentTime": "1970-01-05T14:00:00Z"
  },
  "DateLessThan": {
    "aws:CurrentTime": "1970-01-05T22:00:00Z"
  }
}
```

### Emergency Access Window
```json
"Condition": {
  "StringLike": {
    "aws:userid": "AROA*:emergency-user"
  },
  "DateGreaterThan": {
    "aws:CurrentTime": "2025-06-21T20:00:00Z"
  },
  "DateLessThan": {
    "aws:CurrentTime": "2025-06-22T02:00:00Z"
  },
  "StringEquals": {
    "sts:ExternalId": "emergency-incident-12345"
  }
}
```

### Maintenance Window Restriction
```json
"Condition": {
  "DateGreaterThan": {
    "aws:CurrentTime": "2025-06-21T06:00:00Z"
  },
  "DateLessThan": {
    "aws:CurrentTime": "2025-06-21T08:00:00Z"
  },
  "StringLike": {
    "aws:userid": "AROA*:maintenance-*"
  }
}
```

---

## Exam Preparation Notes

### Common Question Patterns

**Scenario 1:** "Users can access resources during business hours but get AccessDenied at night. What's the most likely cause?"
- Answer: Time-based condition in trust policy

**Scenario 2:** "How do you restrict role assumption to weekdays only?"
- Answer: Use day-of-week logic with 1970 epoch dates

**Scenario 3:** "Error messages don't specify which condition failed. How do you troubleshoot?"
- Answer: Use CloudTrail, policy simulator, and systematic testing

### Key Concepts to Remember

**Time Zone Conversion:**
- AWS always uses UTC for time conditions
- Convert local business hours to UTC
- Account for daylight saving time changes

**Error Message Security:**
- AWS never reveals which specific condition failed
- Generic AccessDenied messages prevent information disclosure
- Use additional tools for detailed troubleshooting

**Condition Precedence:**
- All conditions in a statement must be true (AND logic)
- Multiple statements create OR logic
- Explicit deny always overrides allow

---

## Next Implementation Steps

### Additional Conditional Controls

**IP Address Restrictions:**
```json
"IpAddress": {
  "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
}
```

**MFA Requirements:**
```json
"Bool": {
  "aws:MultiFactorAuthPresent": "true"
},
"NumericLessThan": {
  "aws:MultiFactorAuthAge": "3600"
}
```

**Geographic Restrictions:**
```json
"StringEquals": {
  "aws:RequestedRegion": ["us-east-1", "us-west-2"]
}
```

### Progressive Security Model

**Tier 1:** Basic federated access (Identity Center)
**Tier 2:** Time-restricted intermediate role
**Tier 3:** MFA + IP restricted target role  
**Tier 4:** Emergency access with external approval

This layered approach provides defense in depth while maintaining usability for legitimate access scenarios.