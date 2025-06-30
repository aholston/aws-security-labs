# Automated Incident Response to Suspicious IAM Activity

This lab builds comprehensive automated incident response capabilities for IAM security threats using AWS Security Hub, EventBridge, and Lambda. It demonstrates enterprise-grade threat containment patterns that respond to security incidents within 60 seconds while preserving forensic evidence and maintaining complete audit trails.

The lab emphasizes:
- Real-time IAM threat detection and automated containment
- Cross-account security orchestration with least-privilege access
- Production-ready incident response workflows with comprehensive logging
- Enterprise security patterns for SOC automation and compliance

---

## Lab Architecture

```
Security Hub Finding (HIGH/CRITICAL IAM)
    ↓ (EventBridge rule triggers)
    ├── SNS Topic (immediate human notification)
    └── Lambda Function (automated containment)
        ↓ (assumes cross-account role with External ID)
        IAM Remediation Actions (prod-account)
        ├── Detach dangerous policies
        ├── Deactivate access keys  
        ├── Remove from groups
        ├── Apply forensic tags
        └── Update finding status (bidirectional sync)
```

## Prerequisites

This lab extends the foundational Security Hub detection setup:
- AWS Organization with Security Hub administrator/member relationship
- EventBridge integration with Security Hub configured for finding processing
- SNS alerting infrastructure for security team notifications
- CloudTrail logging enabled for comprehensive audit trails

---

## Phase 1: Lambda-Powered Automated Remediation

### 🎯 Learning Objectives
- Implement cross-account automated remediation with proper security controls
- Master External ID patterns for confused deputy attack prevention
- Build production-ready incident response with forensic preservation
- Practice enterprise SOC automation patterns and audit requirements

### ⚡ Key Exam Topics Covered
- Domain 1.3: Respond to compromised resources and workloads (automated response)
- Cross-account IAM remediation patterns (heavily tested scenarios)
- Security Hub finding lifecycle management and workflow automation
- EventBridge integration patterns for security orchestration

---

## Implementation Summary

### Step 1: Automated Remediation Engine
- **Account**: security-account (Security Hub administrator)
- **Function**: `SecurityHubIAMRemediation` Lambda with intelligent finding processing
- **Capabilities**: Policy detachment, access key deactivation, group removal, forensic tagging

### Step 2: Cross-Account Security Role
- **Account**: prod-account (target resources)
- **Role**: `SecurityRemediationRole` with least-privilege remediation permissions
- **Security**: External ID requirement, explicit deny on destructive operations

### Step 3: EventBridge Orchestration
- **Integration**: Dual-target EventBridge rule (SNS + Lambda)
- **Pattern**: HIGH/CRITICAL severity IAM findings with NEW workflow status
- **Flow**: Parallel human notification and automated containment

### Step 4: Comprehensive Testing
- **Scenario**: Dangerous IAM user with AdministratorAccess policy
- **Validation**: Policy detachment, access key deactivation, forensic tagging
- **Verification**: Complete audit trail and finding status updates

---

## Security Design Principles

### Automated Threat Containment
**Speed**: Mean time to containment reduced from hours to under 60 seconds
**Precision**: Intelligent finding analysis with type-specific remediation actions
**Safety**: Non-destructive containment preserves user accounts for investigation

### Cross-Account Security Controls
**External ID**: Prevents confused deputy attacks in cross-account role assumption
**Least Privilege**: Minimal permissions with explicit deny on destructive operations
**Audit Trail**: Complete CloudTrail logging of all cross-account remediation actions

### Forensic Evidence Preservation
**User Account Retention**: Users disabled but preserved for investigation
**Security Tagging**: Incident metadata, remediation timestamps, finding references
**Finding Lifecycle**: Bidirectional Security Hub updates for compliance tracking

---

## Real-World Application Scenarios

### Enterprise SOC Automation
- **24/7 incident response** without human intervention for common threats
- **Escalation workflows** for complex scenarios requiring manual analysis
- **Compliance reporting** with automated finding resolution documentation

### Multi-Account Security Operations
- **Centralized remediation** from Security Hub administrator account
- **Distributed execution** across member accounts with proper isolation
- **Consistent response** regardless of account size or complexity

### Regulatory Compliance
- **Immediate containment** meeting incident response time requirements
- **Complete audit trails** for compliance reporting and forensic analysis
- **Evidence preservation** maintaining investigative capabilities

---

## Key Implementation Discoveries

### EventBridge Integration Patterns
**Dual Targeting**: Single rule triggering both human notification (SNS) and automated response (Lambda)
**Event Pattern Precision**: Exact matching on finding severity, workflow status, and content
**Cross-Account Events**: Security Hub findings flow from member to administrator accounts

### Lambda Security Architecture
**Cross-Account Role Assumption**: External ID provides security while enabling automation
**Error Handling**: Graceful degradation with comprehensive logging for troubleshooting
**Timeout Management**: Efficient remediation within Lambda execution limits

### Security Hub Finding Management
**Bidirectional Sync**: Finding status updates in administrator account sync to member accounts
**Product ARN Complexity**: Cross-account finding updates require proper ARN formatting
**Workflow States**: NEW → IN_PROGRESS → RESOLVED lifecycle with automated transitions

---

## Exam Preparation Value

### Frequently Tested Scenarios
- "Design automated response to IAM privilege escalation incidents"
- "Implement cross-account security remediation with proper isolation"
- "Configure Security Hub for automated finding lifecycle management"
- "Establish incident response automation meeting compliance requirements"

### Common Question Patterns
**Architecture Design**: Multi-account incident response orchestration
**Security Controls**: External ID implementation and cross-account trust policies
**Automation Logic**: EventBridge pattern matching and Lambda integration
**Audit Requirements**: CloudTrail logging and Security Hub workflow management

### Real-World Skills
**Production Security**: Enterprise-grade automated incident response implementation
**Compliance Alignment**: Audit trail requirements and evidence preservation
**Operational Excellence**: Monitoring, logging, and troubleshooting automation workflows

---

## Advanced Extensions

### Enhanced Detection Sources
- **AWS Config integration** for real-time configuration change detection
- **GuardDuty behavioral analysis** for sophisticated threat identification
- **Custom detection logic** with CloudTrail-based EventBridge rules

### Comprehensive Remediation Actions
- **EC2 instance isolation** with security group modifications and network ACLs
- **S3 bucket protection** with policy updates and access logging enhancement
- **Database security** with parameter group modifications and access controls

### Enterprise Integration
- **SIEM forwarding** with standardized incident data formats
- **Ticketing system integration** for complex scenarios requiring human analysis
- **Metrics and dashboards** for security operations center visibility

---

## Success Criteria

✅ **Automated remediation functional** with sub-60-second response times  
✅ **Cross-account security implemented** with External ID and least-privilege access  
✅ **EventBridge orchestration operational** with dual SNS/Lambda targeting  
✅ **Forensic preservation verified** with user retention and security tagging  
✅ **Audit trail comprehensive** with CloudTrail logging and Security Hub updates  
✅ **Production-ready patterns demonstrated** with enterprise security controls  

This automated incident response implementation provides hands-on experience with the most advanced security automation patterns tested in the AWS Security Specialty exam while building real-world SOC capabilities for immediate threat containment and compliance requirements.