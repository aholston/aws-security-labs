# Customer Identity and Access Control with Amazon Cognito

This lab demonstrates enterprise-grade customer identity management using Amazon Cognito User Pools and Identity Pools for application users. It showcases the core patterns tested in AWS Certified Security – Specialty Domain 2 (Logging and Monitoring) and Domain 4 (Identity and Access Management) scenarios.

The lab emphasizes:
- Customer authentication vs workforce identity (Cognito vs IAM Identity Center)
- User Pool groups with fine-grained authorization mapping
- Identity Pool credential vending with automatic customer data isolation
- Real-world application security patterns for customer-facing services

---

## Lab Architecture

```
Customer Users → Cognito User Pool (Groups: premium/basic) → Identity Pool → Temporary AWS Credentials → S3 (user-specific folders)
```

**Security Flow**:
1. **Customer authenticates** via User Pool with username/password
2. **JWT tokens issued** containing group claims (`cognito:groups`)
3. **Identity Pool maps groups** to IAM roles with different permissions
4. **STS issues temporary credentials** scoped to customer's data partition
5. **S3 access automatically isolated** using `${cognito-identity.amazonaws.com:sub}` variable

## Prerequisites

This lab requires:
- AWS Organization with prod-account for application resources
- Basic understanding of JWT tokens and OAuth flows
- Familiarity with IAM roles and S3 bucket policies
- AWS CLI configured with admin access to prod-account

---

## Core Concepts Demonstrated

### 1. Customer vs Workforce Identity

**Amazon Cognito**: Customer identity for application users
- Users create accounts specifically for your application
- Authentication handled by your app's User Pool
- Authorization through temporary AWS credentials
- **Exam context**: "Mobile app customers need secure file storage"

**IAM Identity Center**: Workforce identity for AWS resource access
- Employees accessing AWS services directly
- Federation with corporate identity providers
- Cross-account role assumption patterns
- **Exam context**: "DevOps teams need multi-account access"

### 2. User Pool Security Architecture

**Authentication Layer**:
- Internal user directory with configurable password policies
- Group-based organization (premium-users, basic-users)
- JWT token issuance with standard and custom claims
- App client configuration with security controls

**Authorization Layer**:
- Group claims in JWT tokens (`"cognito:groups": ["premium-users"]`)
- Identity Pool rule mapping groups to IAM roles
- Temporary credential vending through AWS STS
- Fine-grained S3 permissions using identity variables

### 3. Customer Data Isolation

**Automatic Partitioning**:
- Each authenticated user receives unique Identity ID
- S3 resource policies use `${cognito-identity.amazonaws.com:sub}` variable
- Customers can only access their own data partition
- No application code required for data isolation

**Tiered Service Model**:
- **Premium users**: Full file management (read, write, delete)
- **Basic users**: Read-only access to existing files
- **Group-based permissions** automatically enforced by IAM roles

---

## Implementation Results

### User Pool Configuration
- ✅ **Internal directory** with premium/basic user groups
- ✅ **App client** configured for traditional web application flow
- ✅ **Test users** assigned to different service tiers
- ✅ **Password policies** and security controls enabled

### Identity Pool Integration
- ✅ **Role mapping rules** connecting groups to IAM roles
- ✅ **Cross-account trust policies** properly configured
- ✅ **Temporary credential flow** validated end-to-end
- ✅ **S3 access control** automatically scoped by customer identity

### Security Validation
- ✅ **Premium users** can create, read, update, delete files in personal folder
- ✅ **Basic users** can only read files in personal folder
- ✅ **Cross-customer access** properly denied by IAM policies
- ✅ **JWT token claims** correctly mapped to AWS permissions

---

## Exam Preparation Value

### Frequently Tested Scenarios

**Domain 2: Logging and Monitoring**
- "How do you track customer authentication events across your application?"
- "What's the difference between CloudTrail events for workforce vs customer identity?"
- "How do you monitor suspicious access patterns in customer-facing applications?"

**Domain 4: Identity and Access Management**
- "A mobile app needs secure, scalable customer authentication. Which AWS service?"
- "How do you implement tiered service levels (premium vs basic) for application users?"
- "What's the most secure way to provide customers direct access to S3 objects?"
- "How do you automatically isolate customer data without application-level filtering?"

### Common Troubleshooting Patterns

**Authentication Failures**:
- User Pool password policy enforcement
- App client authentication flow configuration
- SECRET_HASH calculation for CLI testing
- JWT token expiration and refresh patterns

**Authorization Issues**:
- Identity Pool role mapping rule evaluation
- IAM trust policy configuration for federated identities
- S3 resource policy variable substitution
- Cross-customer access prevention validation

### Real-World Application

**Production Considerations**:
- Token security and validation in application code
- Scalable architecture patterns for millions of customers
- Integration with API Gateway for RESTful services
- Monitoring and alerting for security events

**Compliance Alignment**:
- Automatic audit trails for customer data access
- Fine-grained permission models for regulatory requirements
- Customer data residency and isolation controls
- Identity lifecycle management for application users

---

## Advanced Extensions

### Enhanced Security
- **MFA enforcement** for premium user accounts
- **Risk-based authentication** with Cognito advanced security
- **Custom authentication flows** with Lambda triggers
- **Token revocation** and session management

### Application Integration
- **API Gateway authorization** with Cognito authorizers
- **Lambda function access** using temporary credentials
- **CloudFront signed URLs** for private content delivery
- **Cross-origin resource sharing** (CORS) configuration

### Monitoring and Analytics
- **CloudTrail integration** for customer authentication events
- **CloudWatch metrics** for application usage patterns
- **EventBridge rules** for security event detection
- **AWS WAF integration** for application protection

---

## Success Metrics

✅ **Customer authentication** successfully implemented with User Pool  
✅ **Group-based authorization** automatically mapping to different service levels  
✅ **Customer data isolation** enforced at AWS IAM level without application code  
✅ **Temporary credentials** properly scoped to individual customer data partitions  
✅ **End-to-end security flow** validated from authentication through data access  
✅ **Production-ready patterns** demonstrated for scalable customer identity management  

This lab provides comprehensive hands-on experience with Amazon Cognito's most important security patterns, directly preparing for Security Specialty exam scenarios while building real-world customer identity management expertise.
