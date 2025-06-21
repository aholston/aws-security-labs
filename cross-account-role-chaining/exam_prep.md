# AWS Security Specialty Exam Preparation: Cross-Account Role Chaining

## Domain 4 Focus Areas

This lab directly addresses the most challenging and frequently tested concepts in **Domain 4: Identity and Access Management** (30-35% of exam).

---

## Key Exam Topics Mastered

### 1. Cross-Account Role Assumption
**What the exam tests:**
- Understanding trust relationship requirements between accounts
- Troubleshooting failed role assumptions
- Identifying correct Principal and Condition syntax

**Real exam question pattern:**
*"A developer in Account A needs to access an S3 bucket in Account B. The role assumption is failing. Which trust policy condition would resolve this?"*

**Lab connection:** You implemented the exact trust policy structure and troubleshot the `aws:userid` condition format for federated users.

### 2. External ID Implementation
**What the exam tests:**
- Understanding confused deputy attack scenarios
- Proper External ID configuration for third-party access
- When External ID is required vs. optional

**Real exam question pattern:**
*"Your company needs to grant a third-party vendor access to specific AWS resources. What mechanism prevents the vendor from accessing your resources on behalf of other customers?"*

**Lab connection:** You configured External ID in the target role trust policy and tested the requirement during role assumption.

### 3. Federated Identity Trust Policies
**What the exam tests:**
- Differences between IAM user and federated user identification
- SAML assertion attributes in trust policies
- IAM Identity Center (AWS SSO) integration patterns

**Real exam question pattern:**
*"A SAML federated user cannot assume a cross-account role. The trust policy uses the user's IAM ARN in the Principal field. What's the issue?"*

**Lab connection:** You discovered that federated users have dynamic ARNs and require condition-based identification using `aws:userid`.

### 4. STS Service Mechanics
**What the exam tests:**
- Temporary credential lifetime and chaining
- Session token propagation across role assumptions
- STS API parameters and error handling

**Real exam question pattern:**
*"A multi-hop role assumption chain is failing after 2 hours. What could be the cause?"*

**Lab connection:** You experienced the full STS token flow through three separate role assumptions with different session durations.

---

## Common Exam Scenarios

### Scenario 1: Trust Policy Troubleshooting
**Exam Context:** Cross-account role assumption failing with AccessDenied

**Key Learning from Lab:**
- Trust policies must specify correct Principal (account root or specific role)
- Conditions must match actual caller attributes exactly
- Federated users require StringLike conditions, not exact StringEquals

**Exam Answer Pattern:** Look for trust policy conditions that don't match the caller's actual identity attributes.

### Scenario 2: External ID Requirements
**Exam Context:** Third-party vendor access security

**Key Learning from Lab:**
- External ID is required in BOTH trust policy AND assume-role call
- External ID should be unique per customer
- Missing External ID results in AccessDenied error

**Exam Answer Pattern:** Questions about vendor access always point toward External ID as the solution.

### Scenario 3: Permission Boundaries
**Exam Context:** Role has assume-role permissions but still can't access target account

**Key Learning from Lab:**
- Each role in chain needs explicit assume-role permissions
- Resource ARNs must be complete and accurate
- Permission sets in Identity Center require provisioning

**Exam Answer Pattern:** Check both trust policies AND permission policies in the assumption chain.

### Scenario 4: Session Duration Limits
**Exam Context:** Long-running processes failing with token expiration

**Key Learning from Lab:**
- Default role session duration is 1 hour
- Each hop in chain gets new session with own duration
- Applications must handle token refresh

**Exam Answer Pattern:** Look for MaxSessionDuration settings and token refresh mechanisms.

---

## Exam Question Types and Strategies

### Multiple Choice: Trust Policy Syntax
**Strategy:** Focus on Principal field and Condition syntax
**Common wrong answers:** 
- Using IAM user ARNs for federated users
- Incorrect condition operators (StringEquals vs. StringLike)
- Missing account boundaries in Principal

### Multiple Choice: Error Message Interpretation
**Strategy:** Map error messages to specific misconfigurations
**Common scenarios:**
- "AccessDenied" → Trust policy or permissions issue
- "InvalidParameterValue" → Malformed ARN or External ID
- "TokenRefreshRequired" → Session expiration

### Scenario-Based: End-to-End Troubleshooting
**Strategy:** Work through the assumption chain systematically
**Troubleshooting order:**
1. Verify trust policy allows the caller
2. Check caller has assume-role permissions
3. Validate External ID if required
4. Confirm target role permissions

---

## Memory Aids for Exam Day

### Trust Policy Template
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:root OR role-arn"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "required-for-third-party"
        }
      }
    }
  ]
}
```

### Federated User Conditions
- **SAML**: `aws:userid` with "SAML:" prefix
- **OIDC**: `aws:userid` with token subject
- **Identity Center**: `aws:userid` with "AROA" prefix + username

### External ID Rules
- **Required for**: Third-party vendor access
- **Not required for**: Internal cross-account access
- **Security benefit**: Prevents confused deputy attacks
- **Best practice**: Unique per customer/vendor relationship

---

## Quick Reference: Common Mistakes

### ❌ Trust Policy Errors
```json
// WRONG: Using IAM user ARN for federated user
"Principal": {
  "AWS": "arn:aws:iam::123456789012:user/federated-user"
}

// CORRECT: Using account root with condition
"Principal": {
  "AWS": "arn:aws:iam::123456789012:root"
},
"Condition": {
  "StringLike": {
    "aws:userid": "AROA*:username"
  }
}
```

### ❌ External ID Mistakes
```bash
# WRONG: Forgetting External ID parameter
aws sts assume-role --role-arn arn:aws:iam::account:role/role-name

# CORRECT: Including External ID
aws sts assume-role --role-arn arn:aws:iam::account:role/role-name --external-id unique-id
```

### ❌ Permission Policy Errors
```json
// WRONG: Too broad resource permission
"Resource": "*"

// CORRECT: Specific role ARN
"Resource": "arn:aws:iam::account:role/specific-role"
```

---

## Practice Questions Based on Lab

### Question 1
A federated user authenticated through AWS IAM Identity Center cannot assume a cross-account role. The trust policy includes the following:

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:user/devuser"
}
```

What is the issue?

**A)** The account ID is incorrect  
**B)** Federated users don't have persistent ARNs  
**C)** The action should be "sts:AssumeRoleWithSAML"  
**D)** The External ID is missing  

**Answer:** B - Federated users assume temporary roles and need condition-based identification.

### Question 2
A role assumption chain works for 45 minutes then fails. What is the most likely cause?

**A)** Network connectivity issues  
**B)** Session token expiration  
**C)** Invalid External ID  
**D)** Trust policy misconfiguration  

**Answer:** B - Default session duration is 1 hour, and tokens expire independently in each role.

### Question 3
Which condition prevents the confused deputy attack in cross-account access?

**A)** MFA requirement  
**B)** Source IP restriction  
**C)** External ID requirement  
**D)** Time-based conditions  

**Answer:** C - External ID ensures third parties can't trick services into accessing wrong accounts.

---

## Study Schedule Recommendations

### Week Before Exam
- **Day 1-2:** Review trust policy syntax variations
- **Day 3-4:** Practice External ID scenarios
- **Day 5-6:** Work through STS error troubleshooting
- **Day 7:** Take practice exams focusing on Domain 4

### Day Before Exam
- Review trust policy templates
- Memorize common error messages and causes
- Practice drawing role assumption flows
- Review External ID best practices

This hands-on lab experience provides the deep understanding needed to confidently tackle the most complex cross-account questions on the AWS Security Specialty exam.