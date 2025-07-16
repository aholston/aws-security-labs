# Implementation Guide: Multi-Account VPC Security Architecture

## Overview

This guide implements enterprise-grade network security architecture using multi-tier VPC design, progressive access controls, and comprehensive network monitoring. The implementation creates a foundation for secure application deployment while integrating with existing IAM federation and incident response systems.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with prod-account for application resources
- Existing IAM Identity Center federation and cross-account role chaining
- Security Hub administrator setup with automated incident response
- Admin access to prod-account (`prod-admin` IAM user)

### Account Structure
| Account | Purpose | Network Role |
|---------|---------|--------------|
| prod-account | Application resources, VPC implementation | Target network architecture |
| security-account | Network monitoring integration | VPC Flow Log analysis hub |

---

## Phase 1: VPC Foundation and Network Segmentation

### Step 1.1: Create Production VPC

**Console Navigation:** `prod-account` → **VPC** → **Create VPC**

**VPC Configuration:**
- **Name**: `ProdVPC`
- **IPv4 CIDR**: `10.0.0.0/16`
- **IPv6**: No IPv6 CIDR block
- **Tenancy**: Default

**CIDR Strategy Rationale:**
- `10.0.0.0/16` provides 65,536 IP addresses for growth
- Organized allocation: Public (1-2), App (11-12), Data (21-22)
- Human-readable security zone identification
- Simplified security group and NACL rule creation

### Step 1.2: Create Multi-Tier Subnet Architecture

**Console Navigation:** **VPC** → **Subnets** → **Create subnet**

**Public Tier Subnets (Internet-facing):**
```
Name: ProdVPC-Public-1A
VPC: ProdVPC
AZ: us-east-1a
CIDR: 10.0.1.0/24

Name: ProdVPC-Public-1B
VPC: ProdVPC
AZ: us-east-1b
CIDR: 10.0.2.0/24
```

**Application Tier Subnets (Private with NAT access):**
```
Name: ProdVPC-App-1A
VPC: ProdVPC
AZ: us-east-1a
CIDR: 10.0.11.0/24

Name: ProdVPC-App-1B
VPC: ProdVPC
AZ: us-east-1b
CIDR: 10.0.12.0/24
```

**Data Tier Subnets (Isolated, no internet):**
```
Name: ProdVPC-Data-1A
VPC: ProdVPC
AZ: us-east-1a
CIDR: 10.0.21.0/24

Name: ProdVPC-Data-1B
VPC: ProdVPC
AZ: us-east-1b
CIDR: 10.0.22.0/24
```

---

## Phase 2: Internet Access Architecture

### Step 2.1: Create Internet Gateway

**Console Navigation:** **VPC** → **Internet gateways** → **Create internet gateway**
- **Name**: `ProdVPC-IGW`
- **Create and attach to VPC**: Select `ProdVPC`

### Step 2.2: Create NAT Gateway for Outbound-Only Access

**Console Navigation:** **VPC** → **NAT gateways** → **Create NAT gateway**
- **Name**: `ProdVPC-NAT-1A`
- **Subnet**: `ProdVPC-Public-1A`
- **Connectivity**: Public
- **Elastic IP**: Allocate Elastic IP

**Security Design:** NAT Gateway placement in public subnet provides controlled outbound internet access for application tier while preventing inbound connections.

### Step 2.3: Configure Route Tables

**Public Route Table:**
```
Name: ProdVPC-Public-RT
Routes:
  - Destination: 10.0.0.0/16 → Target: local
  - Destination: 0.0.0.0/0 → Target: ProdVPC-IGW

Associated Subnets:
  - ProdVPC-Public-1A
  - ProdVPC-Public-1B
```

**Application Route Table:**
```
Name: ProdVPC-App-RT
Routes:
  - Destination: 10.0.0.0/16 → Target: local
  - Destination: 0.0.0.0/0 → Target: ProdVPC-NAT-1A

Associated Subnets:
  - ProdVPC-App-1A
  - ProdVPC-App-1B
```

**Data Route Table:**
```
Name: ProdVPC-Data-RT
Routes:
  - Destination: 10.0.0.0/16 → Target: local
  [No internet route - complete isolation]

Associated Subnets:
  - ProdVPC-Data-1A
  - ProdVPC-Data-1B
```

---

## Phase 3: Security Group Implementation

### Step 3.1: Application Server Security Group

**Console Navigation:** **VPC** → **Security Groups** → **Create security group**

**Basic Configuration:**
- **Name**: `AppServer-SG`
- **Description**: `Security group for application servers in private subnets`
- **VPC**: `ProdVPC`

**Inbound Rules:**
| Type | Port | Source | Description |
|------|------|--------|-------------|
| HTTP | 80 | 10.0.1.0/23 | HTTP from load balancers |
| HTTPS | 443 | 10.0.1.0/23 | HTTPS from load balancers |
| SSH | 22 | 10.0.1.0/23 | Emergency SSH from bastion hosts |

**Outbound Rules:**
| Type | Port | Destination | Description |
|------|------|-------------|-------------|
| HTTPS | 443 | 0.0.0.0/0 | Software updates and API calls |
| HTTP | 80 | 0.0.0.0/0 | Software updates |
| MySQL/Aurora | 3306 | 10.0.21.0/23 | Database access to data tier |
| Custom UDP | 53 | 0.0.0.0/0 | DNS resolution |

### Step 3.2: Database Security Group

**Basic Configuration:**
- **Name**: `Database-SG`
- **Description**: `Security group for databases in isolated data tier`
- **VPC**: `ProdVPC`

**Inbound Rules:**
| Type | Port | Source | Description |
|------|------|--------|-------------|
| MySQL/Aurora | 3306 | AppServer-SG | Database access from app servers only |
| PostgreSQL | 5432 | AppServer-SG | PostgreSQL access from app servers only |

**Outbound Rules:** None (databases should not initiate outbound connections)

### Step 3.3: Load Balancer Security Group

**Basic Configuration:**
- **Name**: `LoadBalancer-SG`
- **Description**: `Security group for load balancers in public subnets`
- **VPC**: `ProdVPC`

**Inbound Rules:**
| Type | Port | Source | Description |
|------|------|--------|-------------|
| HTTP | 80 | 0.0.0.0/0 | HTTP access from internet |
| HTTPS | 443 | 0.0.0.0/0 | HTTPS access from internet |

**Outbound Rules:**
| Type | Port | Destination | Description |
|------|------|-------------|-------------|
| HTTP | 80 | AppServer-SG | Forward HTTP to application servers |
| HTTPS | 443 | AppServer-SG | Forward HTTPS to application servers |

---

## Phase 4: Network ACL Implementation

### Step 4.1: Application Tier NACL

**Console Navigation:** **VPC** → **Network ACLs** → **Create network ACL**
- **Name**: `App-Tier-NACL`
- **VPC**: `ProdVPC`

**Inbound Rules:**
| Rule # | Type | Protocol | Port Range | Source | Allow/Deny | Description |
|--------|------|----------|------------|--------|------------|-------------|
| 100 | HTTP | TCP | 80 | 10.0.1.0/23 | ALLOW | HTTP from public tier |
| 110 | HTTPS | TCP | 443 | 10.0.1.0/23 | ALLOW | HTTPS from public tier |
| 120 | SSH | TCP | 22 | 10.0.1.0/23 | ALLOW | SSH from bastion hosts |
| 130 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW | Return traffic (ephemeral ports) |

**Outbound Rules:**
| Rule # | Type | Protocol | Port Range | Destination | Allow/Deny | Description |
|--------|------|----------|------------|-------------|------------|-------------|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW | HTTP for updates |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW | HTTPS for updates |
| 120 | MySQL | TCP | 3306 | 10.0.21.0/23 | ALLOW | Database access |
| 130 | DNS | UDP | 53 | 0.0.0.0/0 | ALLOW | DNS resolution |
| 140 | Custom TCP | TCP | 1024-65535 | 10.0.1.0/23 | ALLOW | Response traffic to public tier |

**Associate with App Subnets:**
- `ProdVPC-App-1A`
- `ProdVPC-App-1B`

### Step 4.2: Data Tier NACL (Most Restrictive)

**Create NACL:**
- **Name**: `Data-Tier-NACL`
- **VPC**: `ProdVPC`

**Inbound Rules:**
| Rule # | Type | Protocol | Port Range | Source | Allow/Deny | Description |
|--------|------|----------|------------|--------|------------|-------------|
| 100 | MySQL | TCP | 3306 | 10.0.11.0/23 | ALLOW | MySQL from app tier only |
| 110 | PostgreSQL | TCP | 5432 | 10.0.11.0/23 | ALLOW | PostgreSQL from app tier only |
| 120 | Custom TCP | TCP | 1024-65535 | 10.0.11.0/23 | ALLOW | Return traffic to app tier |

**Outbound Rules:**
| Rule # | Type | Protocol | Port Range | Destination | Allow/Deny | Description |
|--------|------|----------|------------|-------------|------------|-------------|
| 100 | Custom TCP | TCP | 1024-65535 | 10.0.11.0/23 | ALLOW | Response traffic to app tier |

**Associate with Data Subnets:**
- `ProdVPC-Data-1A`
- `ProdVPC-Data-1B`

---

## Phase 5: Network Monitoring and Threat Detection

### Step 5.1: Create CloudWatch Log Group

**Console Navigation:** **CloudWatch** → **Logs** → **Log groups** → **Create log group**
- **Log group name**: `/aws/vpc/flowlogs`
- **Retention setting**: `30 days`
- **Log class**: Standard

### Step 5.2: Create IAM Role for VPC Flow Logs

**Console Navigation:** **IAM** → **Roles** → **Create role**

**Trust Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Permissions:** Attach `AmazonVPCFullAccess` managed policy

**Role Configuration:**
- **Role name**: `flowlogsRole`
- **Description**: `Allows VPC Flow Logs to deliver logs to CloudWatch`

### Step 5.3: Enable VPC Flow Logs

**Console Navigation:** **VPC** → Select `ProdVPC` → **Flow logs tab** → **Create flow log**

**Configuration:**
- **Name**: `ProdVPC-FlowLogs`
- **Filter**: `All` (capture accepted and rejected traffic)
- **Maximum aggregation interval**: `1 minute`
- **Destination**: `Send to CloudWatch Logs`
- **Destination log group**: `/aws/vpc/flowlogs`
- **IAM role**: `flowlogsRole`
- **Log format**: Default format

---

## Phase 6: Attack Simulation Infrastructure

### Step 6.1: Deploy Test Instances

**Application Server Test Instance:**
- **Name**: `TestAppServer`
- **AMI**: Amazon Linux 2023
- **Instance type**: `t2.micro`
- **Subnet**: `ProdVPC-App-1A`
- **Security Group**: `AppServer-SG`
- **Public IP**: Disabled

**Database Test Instance:**
- **Name**: `TestDatabase`
- **AMI**: Amazon Linux 2023
- **Instance type**: `t2.micro`
- **Subnet**: `ProdVPC-Data-1A`
- **Security Group**: `Database-SG`
- **Public IP**: Disabled

**Bastion Host for Access:**
- **Name**: `TestBastion`
- **AMI**: Amazon Linux 2023
- **Instance type**: `t2.micro`
- **Subnet**: `ProdVPC-Public-1A`
- **Security Group**: Create `Bastion-SG` (SSH from 0.0.0.0/0)
- **Public IP**: Enabled

### Step 6.2: Install Testing Tools

**User Data for Test Instances:**
```bash
#!/bin/bash
yum update -y
yum install -y nmap telnet nc mysql
echo "Test instance ready" > /var/log/setup.log
```

---

## Validation and Verification

### Network Architecture Verification

```bash
# Set VPC ID variable
VPC_ID="vpc-08bbc0c908f6c726f"

# Verify VPC configuration
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=ProdVPC" --query 'Vpcs[0].[VpcId,CidrBlock,State]'

# List all subnets with their tiers
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].[SubnetId,CidrBlock,Tags[?Key==`Name`].Value|[0]]'

# Verify route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query 'RouteTables[].[RouteTableId,Tags[?Key==`Name`].Value|[0],Routes[].DestinationCidrBlock]'

# Check security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[].[GroupId,GroupName,Description]'

# Verify VPC Flow Logs
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID"
```

### Security Control Testing

**Connectivity Testing Sequence:**
1. **Internet → Bastion Host** (should succeed)
2. **Bastion → App Server via SSH** (should succeed)
3. **App Server → Internet HTTPS** (should succeed via NAT)
4. **App Server → Database port 3306** (should succeed)
5. **Internet → App Server direct** (should fail - no route)
6. **Database → Internet** (should fail - no route)

### VPC Flow Log Analysis

**Check Flow Logs:**
```bash
# View recent VPC Flow Logs
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs \
  --start-time $(date -u -v-10M +%s)000 \
  --query 'events[].message'
```

**Sample Flow Log Entry Analysis:**
```
2 947874856077 eni-1235b8ca 10.0.11.25 10.0.21.45 3306 45678 6 7 840 1625097600 1625097660 ACCEPT OK
```
- Source: `10.0.11.25` (app tier)
- Destination: `10.0.21.45` (data tier)
- Port: `3306` (MySQL)
- Action: `ACCEPT` (allowed by security controls)

---

## Attack Simulation Scenarios

### Scenario 1: Lateral Movement Prevention

**Test:** App server attempting SSH to other app servers
```bash
# From TestAppServer, attempt SSH to other instances
ssh ec2-user@10.0.11.X  # Should fail (no SSH between app servers)
```

**Expected VPC Flow Log:** REJECT entries for port 22 between app tier IPs

### Scenario 2: Data Exfiltration Prevention

**Test:** Database attempting external connections
```bash
# From TestDatabase, attempt internet access
curl -m 5 https://google.com  # Should timeout (no internet route)
```

**Expected Result:** No connectivity, timeout after 5 seconds

### Scenario 3: Direct Access Prevention

**Test:** Direct internet access to app/data tiers
```bash
# From internet, attempt connection to app server private IP
nc -zv 10.0.11.X 80  # Should fail (no route from internet)
```

**Expected Result:** Connection refused or timeout

### Scenario 4: Port Scanning Detection

**Test:** Generate scanning traffic patterns
```bash
# From bastion, scan app server
nmap -p 1-1000 10.0.11.X
```

**Expected VPC Flow Log:** Multiple REJECT entries for various ports

---

## Integration with Existing Systems

### IAM Identity Center Integration

**Federated User Access:**
- Federated users access resources through network security controls
- Identity Center roles work within Security Group restrictions
- Network access complements IAM permission boundaries

### Security Hub Automation Integration

**Network Threat Detection:**
- VPC Flow Logs provide network-based findings for Security Hub
- EventBridge rules can trigger on network anomalies
- Lambda automation can respond to network threats

**Sample EventBridge Pattern for Network Threats:**
```json
{
  "source": ["aws.vpc-flow-logs"],
  "detail": {
    "action": ["REJECT"],
    "srcaddr": [{"exists": true}],
    "dstaddr": [{"exists": true}]
  }
}
```

### Customer Identity Integration

**Cognito Application Access:**
- Customer applications access S3 through controlled network paths
- VPC endpoints can provide direct AWS service access
- Network controls complement customer data isolation

---

## Troubleshooting Guide

### Common Issues

**Security Group Rule Conflicts:**
- **Symptom**: Application cannot connect to database despite correct Security Group configuration
- **Cause**: NACL rules blocking traffic or missing ephemeral port ranges
- **Solution**: Verify NACL allows both directions and ephemeral ports (1024-65535)

**NAT Gateway Connectivity Issues:**
- **Symptom**: App servers cannot reach internet despite NAT Gateway configuration
- **Cause**: Route table not pointing to NAT Gateway or NAT Gateway in wrong subnet
- **Solution**: Verify app route table has 0.0.0.0/0 → NAT Gateway and NAT Gateway is in public subnet

**VPC Flow Logs Not Appearing:**
- **Symptom**: No flow logs in CloudWatch despite configuration
- **Cause**: IAM role permissions insufficient or wrong log group
- **Solution**: Verify flowlogsRole has VPC permissions and log group exists

**Instance Cannot Reach Internet:**
- **Symptom**: EC2 instances time out when accessing internet
- **Cause**: Missing route to NAT Gateway or Internet Gateway
- **Solution**: Check route table associations and default routes

### Validation Commands

```bash
# Test connectivity from app server
ssh -i key.pem ec2-user@APP-SERVER-IP
curl -I https://amazon.com  # Should work via NAT

# Test database isolation
ssh -i key.pem ec2-user@DATABASE-IP
curl -m 5 https://google.com  # Should timeout

# Check Security Group rules
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx --query 'SecurityGroups[0].IpPermissions'

# Verify NACL associations
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" --query 'NetworkAcls[].Associations[].SubnetId'

# Monitor real-time flow logs
aws logs tail /aws/vpc/flowlogs --follow
```

---

## Security Best Practices

### Network Architecture
- **Principle of Least Access**: Each tier communicates only with necessary resources
- **Defense in Depth**: Multiple security layers from routing to Security Groups to NACLs
- **Segregation of Duties**: Clear separation between public, private, and isolated tiers

### Monitoring and Alerting
- **Comprehensive Logging**: VPC Flow Logs capture all network traffic for analysis
- **Real-time Detection**: 60-second aggregation enables rapid threat identification
- **Integration Points**: Network monitoring complements existing Security Hub automation

### Operational Security
- **Organized IP Addressing**: Human-readable CIDR blocks for operational efficiency
- **Documentation**: Clear security zone identification and traffic flow documentation
- **Change Control**: Network modifications should follow established security procedures

---

## Performance Considerations

### NAT Gateway Scaling
- **Bandwidth**: NAT Gateway supports up to 45 Gbps
- **High Availability**: Consider NAT Gateway in each AZ for fault tolerance
- **Cost Optimization**: Monitor data processing charges for high-traffic applications

### VPC Flow Log Impact
- **Storage Costs**: CloudWatch Logs charges for ingestion and storage
- **Network Performance**: Minimal impact on network performance
- **Log Retention**: Balance security requirements with storage costs

### Security Group Limits
- **Rules per Security Group**: 60 inbound, 60 outbound rules maximum
- **Groups per Interface**: 5 security groups per network interface
- **Referenced Groups**: Security group references don't count against rule limits

---

## Success Validation

✅ **Multi-tier network architecture implemented** with clear security boundaries  
✅ **Progressive internet access controls configured** (full → outbound-only → isolated)  
✅ **Security Groups and NACLs deployed** with defense-in-depth protection  
✅ **VPC Flow Logs operational** with comprehensive network monitoring  
✅ **Test instances deployed** for attack simulation and validation  
✅ **Integration verified** with existing IAM and Security Hub systems  
✅ **Attack scenarios documented** with expected security control responses  

This implementation provides a production-ready network security foundation that supports enterprise applications while maintaining comprehensive security controls and monitoring capabilities.