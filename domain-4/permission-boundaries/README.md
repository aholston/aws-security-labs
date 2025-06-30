# Permission Boundaries with Break-Glass Emergency Access

Enterprise-grade permission boundaries implementation with automated emergency access controls using AWS IAM Identity Center. Demonstrates the most advanced permission boundary concepts tested in AWS Certified Security – Specialty Domain 4.

## Learning Objectives

- **Permission boundary policy evaluation**: Understanding intersection logic and maximum permission enforcement
- **Cross-account policy management**: Identity Center provisioning patterns and policy synchronization  
- **Break-glass emergency access**: Time-limited elevated permissions with automated revocation
- **Complex policy troubleshooting**: Systematic debugging methodology for access denial scenarios

## Lab Architecture

```
boundaryuser (IAM Identity Center)
    ↓ (federated authentication)
SimpleBoundedDeveloper Permission Set
    ↓ (permission boundary enforcement)
SimpleDeveloperBoundary Policy (max permissions)
    ↓ (intersection with identity policy)
Effective Permissions: EC2 t2.micro, S3 dev-*, emergency role access

Emergency Access Flow:
boundaryuser → EmergencyBreakGlassRole → EventBridge Detection → Lambda Scheduling → Auto-Revocation (4 hours)
```

## Prerequisites

- AWS Organization with IAM Identity Center enabled
- Delegated administration to security-account
- Existing federated users and groups
- Multi-account structure (management, security-account, prod-account)
- CloudTrail logging enabled for audit trails

## Key Implementation Discoveries

### Permission Boundary Behavior
- **Effective permissions** = Identity Policy ∩ Permission Boundary Policy
- **Maximum permission ceiling** - boundaries never grant, only restrict
- **Cross-account provisioning** uses policy name matching, not ARN references
- **Identity Center roles** are protected and can only be modified through Identity Center

### Break-Glass Access Patterns
- **MFA requirements incompatible** with Identity Center federated sessions
- **External ID provides** sufficient security for emergency access scenarios  
- **EventBridge detection** enables real-time automation and audit trails
- **Lambda scheduled rules** provide reliable automated revocation mechanisms

### Policy Evaluation Precedence
1. **Service Control Policies** (organization/account restrictions)
2. **Permission Boundaries** (maximum permission ceiling)  
3. **Identity Policies** (granted permissions)
4. **Resource Policies** (resource-specific access)

## Success Criteria

✅ **Permission boundary successfully restricts** broad identity policy permissions  
✅ **Emergency role assumption functional** with proper security controls  
✅ **Automated detection and scheduling** working via EventBridge and Lambda  
✅ **Cross-account policy management** implemented with Identity Center patterns  
✅ **Systematic troubleshooting methodology** validated through complex debugging scenarios  

## Security Controls Implemented

- **Deliberate emergency access** requiring External ID knowledge
- **Automatic revocation** preventing persistent elevated permissions
- **Comprehensive audit trail** through CloudTrail and Lambda logging
- **Principle of least privilege** with broad-but-limited emergency permissions
- **Protection against destructive operations** during emergency scenarios

This implementation provides hands-on experience with the most sophisticated IAM concepts tested in Domain 4, demonstrating production-ready enterprise security patterns while building deep understanding of AWS policy evaluation logic.