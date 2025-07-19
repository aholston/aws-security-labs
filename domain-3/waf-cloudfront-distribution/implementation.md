# Implementation Guide: AWS WAF and CloudFront Security Architecture

## Overview

This guide implements enterprise-grade web application protection using AWS WAF and CloudFront with comprehensive threat coverage. The implementation creates global edge security that blocks application layer attacks while maintaining high performance and providing detailed security monitoring.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with prod-account for application resources
- Admin access to prod-account (`prod-admin` IAM user)
- Basic understanding of web application security concepts
- Existing VPC infrastructure (optional - integrates with existing architecture)

### Account Structure
| Account | Purpose | Access Method |
|---------|---------|---------------|
| prod-account | CloudFront distribution, S3 origin, WAF implementation | prod-admin IAM user |

---

## Phase 1: S3 Static Website and Origin Protection

### Step 1.1: Create S3 Bucket for Website Origin

**Console Navigation:** `prod-account` → **S3** → **Create bucket**

**Bucket Configuration:**
- **Name**: `waf-lab-webapp-[random-number]` (must be globally unique)
- **Region**: `us-east-1`
- **Block all public access**: ✅ Keep enabled (CloudFront OAC will provide access)
- **Bucket versioning**: Disabled
- **Default encryption**: Server-side encryption with Amazon S3 managed keys (SSE-S3)

```bash
# Alternative CLI approach
BUCKET_NAME="waf-lab-webapp-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME --region us-east-1 --profile prod-admin
echo "Bucket created: $BUCKET_NAME"
```

### Step 1.2: Create Attack-Testable Website Content

**Create comprehensive test website with security testing capabilities:**

```bash
# Create main website page
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AWS WAF Security Lab</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .container { max-width: 800px; background: white; padding: 30px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .attack-test { background: #fff3cd; border: 1px solid #ffeaa7; padding: 15px; margin: 20px 0; border-radius: 4px; }
        input[type="text"] { width: 300px; padding: 8px; margin: 5px; }
        button { background: #007bff; color: white; padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer; }
        .results { margin: 20px 0; padding: 15px; border: 1px solid #ddd; border-radius: 4px; }
        .security-status { margin-top: 30px; padding: 20px; background: #e8f4f8; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🛡️ AWS WAF Protection Lab</h1>
        <p>This website is protected by AWS WAF and delivered via CloudFront global edge locations.</p>
        
        <div class="attack-test">
            <h3>🚨 Security Testing Area</h3>
            <p>Try these attack patterns to test WAF protection:</p>
            
            <form action="/" method="GET">
                <label>Search Query (XSS Test):</label><br>
                <input type="text" name="query" placeholder="Try: &lt;script&gt;alert('XSS')&lt;/script&gt;">
                <button type="submit">Test XSS Protection</button>
            </form>
            
            <form action="/user" method="GET">
                <label>User ID (SQL Injection Test):</label><br>
                <input type="text" name="id" placeholder="Try: 1' OR '1'='1">
                <button type="submit">Test SQL Injection Protection</button>
            </form>
            
            <form action="/admin" method="GET">
                <label>Admin Panel (Path Traversal Test):</label><br>
                <input type="text" name="file" placeholder="Try: ../../etc/passwd">
                <button type="submit">Test Path Traversal Protection</button>
            </form>
        </div>
        
        <div id="results" class="results" style="display:none;">
            <h4>Query Results:</h4>
            <div id="query-display"></div>
        </div>
        
        <div class="security-status">
            <h4>🔒 Current Security Status:</h4>
            <p><strong>Protection Layer:</strong> AWS WAF + CloudFront</p>
            <p><strong>Geographic Restrictions:</strong> Active</p>
            <p><strong>Rate Limiting:</strong> 50 requests per 5 minutes</p>
            <p><strong>SSL/TLS:</strong> Enforced with security headers</p>
        </div>
    </div>
    
    <script>
        // Process URL parameters to simulate vulnerable behavior
        // (WAF should block malicious attempts before they reach this code)
        const urlParams = new URLSearchParams(window.location.search);
        const query = urlParams.get('query');
        const userId = urlParams.get('id');
        const file = urlParams.get('file');
        
        if (query || userId || file) {
            document.getElementById('results').style.display = 'block';
            let displayText = '';
            
            if (query) displayText += `Search Query: ${query}<br>`;
            if (userId) displayText += `User ID: ${userId}<br>`;
            if (file) displayText += `File Path: ${file}<br>`;
            
            document.getElementById('query-display').innerHTML = displayText;
        }
        
        // Display current timestamp and edge location info
        document.addEventListener('DOMContentLoaded', function() {
            const timestamp = new Date().toISOString();
            console.log('Page loaded at:', timestamp);
            console.log('If you see this in browser console, the request bypassed WAF protection');
        });
    </script>
</body>
</html>
EOF

# Create error page for WAF blocks
cat > error.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Access Denied - AWS WAF</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin: 100px; background: #f8f9fa; }
        .container { max-width: 600px; margin: 0 auto; background: white; padding: 40px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .error { color: #d32f2f; font-size: 24px; margin-bottom: 20px; }
        .message { color: #666; font-size: 16px; line-height: 1.6; }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="error">🚫 Access Denied</h1>
        <p class="message">Your request has been blocked by AWS WAF security rules.</p>
        <p class="message">This indicates that our application layer protection is working correctly.</p>
        <p class="message">If you believe this is an error, please contact the website administrator.</p>
    </div>
</body>
</html>
EOF

# Upload files to S3
aws s3 cp index.html s3://$BUCKET_NAME/ --profile prod-admin
aws s3 cp error.html s3://$BUCKET_NAME/ --profile prod-admin

echo "Website files uploaded to S3 bucket: $BUCKET_NAME"
```

### Step 1.3: Create Origin Access Control (OAC)

**Console Navigation:** **CloudFront** → **Origin access control settings** → **Create control setting**

**OAC Configuration:**
- **Name**: `WAF-Lab-OAC`
- **Description**: `Origin Access Control for WAF lab S3 bucket`
- **Origin type**: S3
- **Signing behavior**: Sign requests (recommended)
- **Origin request policy**: CORS-S3Origin

```bash
# CLI alternative for OAC creation
aws cloudfront create-origin-access-control \
  --origin-access-control-config Name="WAF-Lab-OAC",Description="WAF Lab OAC",OriginAccessControlOriginType="s3",SigningBehavior="always",SigningProtocol="sigv4" \
  --profile prod-admin
```

**Security Benefits of OAC:**
- Uses AWS Signature Version 4 for authentication (more secure than OAI)
- Better CloudTrail integration for audit requirements
- Supports additional AWS services beyond S3
- Future-proof with active AWS development and enhancement

---

## Phase 2: CloudFront Distribution Configuration

### Step 2.1: Create CloudFront Distribution

**Console Navigation:** **CloudFront** → **Create distribution**

**Origin Configuration:**
- **Origin domain**: Select your S3 bucket from the dropdown
- **Origin access**: ✅ Origin access control settings (recommended)
- **Origin access control**: Select the OAC created in Step 1.3
- **Origin path**: Leave blank
- **Name**: `S3-WAF-Lab-Origin`

**Default Cache Behavior:**
- **Viewer protocol policy**: Redirect HTTP to HTTPS
- **Allowed HTTP methods**: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
- **Cache policy**: `CachingDisabled` (better for testing dynamic responses)
- **Origin request policy**: `CORS-S3Origin`
- **Response headers policy**: None (will add security headers later)

**Distribution Settings:**
- **Price class**: Use all edge locations (global protection)
- **Default root object**: `index.html`
- **Custom error pages**:
  - HTTP error code: 403 → Response page path: `/error.html` → HTTP response code: 403
  - HTTP error code: 404 → Response page path: `/error.html` → HTTP response code: 404

**Create distribution** and note the **Distribution Domain Name** (e.g., `d1234567890.cloudfront.net`)

### Step 2.2: Update S3 Bucket Policy for OAC

**After distribution creation, CloudFront provides the required bucket policy:**

**Console Navigation:** **CloudFront** → Your distribution → **Origins** → **Edit origin** → **Copy policy**

**Apply the bucket policy:**
```bash
# Example bucket policy for OAC (replace with actual values)
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT-ID:distribution/DISTRIBUTION-ID"
        }
      }
    }
  ]
}
EOF

# Apply the policy (replace placeholders with actual values)
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy file://bucket-policy.json --profile prod-admin
```

### Step 2.3: Verify CloudFront and Origin Protection

```bash
# Test CloudFront distribution (should work)
curl -I https://YOUR-DISTRIBUTION-DOMAIN.cloudfront.net/
# Expected: 200 OK with CloudFront headers

# Verify S3 direct access is blocked (should fail)
curl -I https://$BUCKET_NAME.s3.amazonaws.com/index.html
# Expected: 403 Forbidden (OAC protection working)

# Test website functionality
echo "Test website at: https://YOUR-DISTRIBUTION-DOMAIN.cloudfront.net"
```

---

## Phase 3: AWS WAF Implementation

### Step 3.1: Create Global WAF Web ACL

**Critical**: Ensure you're creating a **Global (CloudFront)** WAF, not regional.

**Console Navigation:** **WAF & Shield** → Verify region shows "Global (CloudFront)" → **Web ACLs** → **Create web ACL**

**Basic Information:**
- **Name**: `WebApp-Protection-WAF`
- **Description**: `Comprehensive application layer protection for web applications`
- **CloudWatch metric name**: `WebAppProtectionWAF`

**Resource Configuration:**
- **Resource type**: CloudFront distributions
- **Region**: Global (CloudFront)

**Associated AWS Resources:**
- **Add AWS resources** → Select your CloudFront distribution → **Add**

**Default Action:** Allow (explicit blocking through rules)

### Step 3.2: Add AWS Managed Rule Groups

**Add rules** → **Add managed rule groups**

#### Core Rule Set (Priority 1)
- **Rule group**: `AWS-AWSManagedRulesCommonRuleSet`
- **Priority**: 1
- **Override all rule actions to**: Count (for initial testing)
- **CloudWatch metrics**: Enabled
- **Rule group capacity**: 700 WCUs

**Protection Coverage:**
- Cross-site scripting (XSS) detection
- SQL injection prevention
- Local file inclusion (LFI) protection
- Remote file inclusion (RFI) protection
- HTTP protocol compliance enforcement

#### Known Bad Inputs (Priority 2)
- **Rule group**: `AWS-AWSManagedRulesKnownBadInputsRuleSet`
- **Priority**: 2
- **Override all rule actions to**: Count
- **Rule group capacity**: 200 WCUs

**Protection Coverage:**
- Known malicious request patterns
- Exploit kit detection
- Vulnerability scanner identification

#### SQL Database Protection (Priority 3)
- **Rule group**: `AWS-AWSManagedRulesSQLiRuleSet`
- **Priority**: 3
- **Override all rule actions to**: Count
- **Rule group capacity**: 200 WCUs

**Protection Coverage:**
- Advanced SQL injection techniques
- Database-specific attack patterns
- Union-based and blind SQL injection

### Step 3.3: Create Custom Rate Limiting Rule

**Add rules** → **Add my own rules and rule groups** → **Rule builder**

**Rule Configuration:**
- **Name**: `DDoSProtection-RateLimit`
- **Type**: Rate-based rule
- **Priority**: 10

**Rate-based Statement:**
- **Rate limit**: 50 requests per 5 minutes (adjust based on legitimate traffic patterns)
- **IP address to use for aggregation**: Source IP address
- **Scope of rate limiting**: All requests that match rule scope

**Action**: Block
- **Block duration**: 300 seconds (5 minutes)
- **Custom response**: Optional 429 status code

### Step 3.4: Add Geographic Restrictions

**Add rules** → **Add my own rules and rule groups** → **Rule builder**

**Rule Configuration:**
- **Name**: `ComplianceGeoBlocking`
- **Type**: Regular rule
- **Priority**: 5

**Statement Configuration:**
- **Statement type**: Geographic match
- **Country codes**: CN, RU, KP, IR (adjust based on compliance requirements)
- **Action**: Block

**Custom Response (Optional):**
- **Response code**: 403
- **Response headers**: `X-Blocked-Reason: Geographic-Restriction`

### Step 3.5: Enable WAF Logging

**Console Navigation:** **WAF & Shield** → Your Web ACL → **Logging and metrics**

**Logging Configuration:**
- **Enable logging**: Toggle on
- **Log destination**: CloudWatch Logs
- **Log group**: `/aws/waf/WebApp-Protection-WAF`
- **Log format**: Full logs (for comprehensive analysis)
- **Sampling rate**: 100% (for complete testing visibility)

**Create Web ACL** to finalize configuration

---

## Phase 4: Security Testing and Rule Optimization

### Step 4.1: Test Rules in Count Mode

**Wait 15-20 minutes for global WAF deployment**, then perform comprehensive testing:

```bash
# Test XSS detection (should be counted, not blocked initially)
curl -s "https://YOUR-DOMAIN.cloudfront.net/?query=<script>alert('XSS')</script>"

# Test SQL injection detection
curl -s "https://YOUR-DOMAIN.cloudfront.net/?id=' OR '1'='1"

# Test legitimate traffic (should work normally)
curl -s "https://YOUR-DOMAIN.cloudfront.net/?query=legitimate search terms"
```

### Step 4.2: Analyze WAF Logs

```bash
# Check WAF logs for detected attacks
aws logs filter-log-events \
  --log-group-name /aws/waf/WebApp-Protection-WAF \
  --start-time $(date -u -v-30M +%s)000 \
  --profile prod-admin \
  --region us-east-1

# Look for specific attack patterns
aws logs filter-log-events \
  --log-group-name /aws/waf/WebApp-Protection-WAF \
  --filter-pattern "script" \
  --start-time $(date -u -v-30M +%s)000 \
  --profile prod-admin \
  --region us-east-1
```

### Step 4.3: Switch to Block Mode

**Once Count mode testing confirms proper detection:**

**Console Navigation:** **WAF & Shield** → Your Web ACL → **Rules**

**For each managed rule group:**
1. **Edit** the rule group
2. **Override all rule actions to**: Block (change from Count)
3. **Save changes**

**Repeat for:**
- Core rule set
- Known bad inputs rule set
- SQL database rule set

### Step 4.4: Validate Blocking Behavior

```bash
# Test XSS blocking (should return 403)
curl -s -w "%{http_code}\n" "https://YOUR-DOMAIN.cloudfront.net/?query=<script>alert('XSS')</script>"

# Test SQL injection blocking (should return 403)
curl -s -w "%{http_code}\n" "https://YOUR-DOMAIN.cloudfront.net/?id=' OR '1'='1"

# Verify legitimate traffic still works (should return 200)
curl -s -w "%{http_code}\n" "https://YOUR-DOMAIN.cloudfront.net/?query=normal search"
```

---

## Phase 5: Security Headers Implementation

### Step 5.1: Create Response Headers Policy

**Console Navigation:** **CloudFront** → **Policies** → **Response headers** → **Create policy**

**Policy Configuration:**
- **Name**: `ComprehensiveSecurityHeaders`
- **Description**: `Enterprise-grade browser security controls`

**Security Headers Configuration:**

**Strict Transport Security:**
- **Strict-Transport-Security**: `max-age=31536000; includeSubDomains; preload`

**Content Type Options:**
- **X-Content-Type-Options**: `nosniff`

**Frame Options:**
- **X-Frame-Options**: `DENY`

**XSS Protection:**
- **X-XSS-Protection**: `1; mode=block`

**Referrer Policy:**
- **Referrer-Policy**: `strict-origin-when-cross-origin`

**Content Security Policy:**
- **Content-Security-Policy**: `default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';`

**Permissions Policy:**
- **Permissions-Policy**: `geolocation=(), microphone=(), camera=(), payment=(), usb=(), interest-cohort=()`

### Step 5.2: Apply Headers to Distribution

**Console Navigation:** **CloudFront** → Your distribution → **Behaviors** → **Edit default behavior**

- **Response headers policy**: Select `ComprehensiveSecurityHeaders`
- **Save changes**

**Wait 15-20 minutes for deployment**

### Step 5.3: Validate Security Headers

```bash
# Test comprehensive security headers
curl -I "https://YOUR-DOMAIN.cloudfront.net/" | grep -E "(X-Frame-Options|Content-Security-Policy|Strict-Transport-Security|X-Content-Type-Options|Permissions-Policy)"

# Full header analysis
curl -s -I "https://YOUR-DOMAIN.cloudfront.net/" | head -25
```

**Expected headers output:**
```
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'...
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
Permissions-Policy: geolocation=(), microphone=(), camera=()...
```

---

## Phase 6: Rate Limiting Testing

### Step 6.1: Rate Limiting Validation Script

```bash
#!/bin/bash
# Comprehensive Rate Limiting Test
DOMAIN="YOUR-DOMAIN.cloudfront.net"

echo "🛡️ Testing DDoS Protection Rate Limiting"
echo "Target: https://$DOMAIN"
echo "Limit: 50 requests per 5 minutes"
echo "========================================"

for i in {1..60}; do
  response=$(curl -s -o /dev/null -w "%{http_code}" "https://$DOMAIN/")
  timestamp=$(date '+%H:%M:%S')
  
  echo "Request $i [$timestamp]: HTTP $response"
  
  if [ "$response" != "200" ]; then
    echo ""
    echo "🛡️ RATE LIMIT TRIGGERED at request $i!"
    echo "WAF is blocking subsequent requests (HTTP $response)"
    echo "Protection working correctly ✅"
    break
  fi
  
  sleep 0.2
done

echo ""
echo "Test completed at $(date)"
```

---

## Phase 7: Comprehensive Security Validation

### Step 7.1: Complete Attack Simulation Suite

```bash
#!/bin/bash
# AWS WAF Security Validation Test Suite
DOMAIN="YOUR-DOMAIN.cloudfront.net"

echo "🛡️ Comprehensive WAF Security Test Suite"
echo "Target: https://$DOMAIN"
echo "========================================="

# Test 1: XSS Protection
echo "1. Testing XSS Protection..."
XSS_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "https://$DOMAIN/?test=<script>alert(1)</script>")
echo "   XSS Attack: HTTP $XSS_RESPONSE $([ $XSS_RESPONSE -eq 403 ] && echo '✅ BLOCKED' || echo '❌ ALLOWED')"

# Test 2: SQL Injection Protection
echo "2. Testing SQL Injection Protection..."
SQL_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "https://$DOMAIN/?id=' OR '1'='1")
echo "   SQL Injection: HTTP $SQL_RESPONSE $([ $SQL_RESPONSE -eq 403 ] && echo '✅ BLOCKED' || echo '❌ ALLOWED')"

# Test 3: Path Traversal Protection
echo "3. Testing Path Traversal Protection..."
PATH_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "https://$DOMAIN/?file=../../etc/passwd")
echo "   Path Traversal: HTTP $PATH_RESPONSE $([ $PATH_RESPONSE -eq 403 ] && echo '✅ BLOCKED' || echo '❌ ALLOWED')"

# Test 4: Security Headers
echo "4. Testing Security Headers..."
HSTS=$(curl -s -I "https://$DOMAIN/" | grep -i "strict-transport-security")
CSP=$(curl -s -I "https://$DOMAIN/" | grep -i "content-security-policy")
FRAME=$(curl -s -I "https://$DOMAIN/" | grep -i "x-frame-options")
echo "   HSTS: $([ -n "$HSTS" ] && echo '✅ PRESENT' || echo '❌ MISSING')"
echo "   CSP: $([ -n "$CSP" ] && echo '✅ PRESENT' || echo '❌ MISSING')"
echo "   X-Frame-Options: $([ -n "$FRAME" ] && echo '✅ PRESENT' || echo '❌ MISSING')"

# Test 5: Legitimate Traffic
echo "5. Testing Legitimate Traffic..."
LEGIT_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "https://$DOMAIN/?query=normal search terms")
echo "   Legitimate Request: HTTP $LEGIT_RESPONSE $([ $LEGIT_RESPONSE -eq 200 ] && echo '✅ ALLOWED' || echo '❌ BLOCKED')"

# Test 6: HTTPS Enforcement
echo "6. Testing HTTPS Enforcement..."
HTTP_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "http://$DOMAIN/" 2>/dev/null || echo "301")
echo "   HTTP Redirect: HTTP $HTTP_RESPONSE $([ $HTTP_RESPONSE -eq 301 ] && echo '✅ REDIRECTED' || echo '❌ NOT ENFORCED')"

echo ""
echo "🎯 Security Stack Test Complete!"
echo "Time: $(date)"
```

### Step 7.2: WAF Metrics and Monitoring

**Console Navigation:** **WAF & Shield** → Your Web ACL → **Monitoring**

**Key Metrics to Monitor:**
- **Allowed requests**: Legitimate traffic volume
- **Blocked requests**: Attack attempts stopped
- **Sampled requests**: Detailed view of recent traffic patterns
- **Rule group performance**: Individual rule effectiveness

**CloudWatch Integration:**
```bash
# Create custom CloudWatch dashboard for WAF metrics
aws cloudwatch put-dashboard \
  --dashboard-name "WAF-Security-Monitoring" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/WAFV2", "AllowedRequests", "WebACL", "WebApp-Protection-WAF"],
            [".", "BlockedRequests", ".", "."]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1",
          "title": "WAF Request Volume"
        }
      }
    ]
  }' \
  --profile prod-admin
```

---

## Phase 8: Integration with Existing Security Architecture

### Step 8.1: Security Hub Integration

**Create EventBridge Rule for WAF Alerts:**

**Console Navigation:** **EventBridge** → **Rules** → **Create rule**

**Rule Configuration:**
- **Name**: `WAF-SecurityHub-Integration`
- **Description**: `Forward WAF blocks to Security Hub for incident response`

**Event Pattern:**
```json
{
  "source": ["aws.wafv2"],
  "detail-type": ["WAF Block Event"],
  "detail": {
    "action": ["BLOCK"],
    "webaclId": ["arn:aws:wafv2:global:ACCOUNT-ID:global/webacl/WebApp-Protection-WAF/*"]
  }
}
```

**Targets:**
- **SNS Topic**: Your existing security alerts topic
- **Lambda Function**: Your existing Security Hub automation

### Step 8.2: CloudWatch Alarms for Threshold Monitoring

```bash
# Create alarm for high block rate (potential attack)
aws cloudwatch put-metric-alarm \
  --alarm-name "WAF-High-Block-Rate" \
  --alarm-description "Alert when WAF blocks exceed normal thresholds" \
  --metric-name BlockedRequests \
  --namespace AWS/WAFV2 \
  --statistic Sum \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=WebACL,Value=WebApp-Protection-WAF \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT-ID:security-alerts \
  --profile prod-admin
```

---

## Troubleshooting Guide

### Common Issues and Solutions

**WAF Rules Not Blocking:**
- **Symptom**: Attacks getting through despite WAF configuration
- **Causes**: Rules in Count mode, wrong WAF scope (regional vs global), WAF not associated with CloudFront
- **Solutions**: Verify rules are in Block mode, ensure Global CloudFront scope, check CloudFront security tab

**CloudFront Distribution Not Updating:**
- **Symptom**: Changes not reflected in browser testing
- **Causes**: Edge location caching, browser caching, deployment still in progress
- **Solutions**: Wait 15-20 minutes, use curl for testing, check distribution status

**S3 Origin Access Issues:**
- **Symptom**: 403 errors when accessing CloudFront distribution
- **Causes**: Incorrect bucket policy, OAC not properly configured
- **Solutions**: Copy exact bucket policy from CloudFront console, verify OAC settings

**Rate Limiting Not Triggering:**
- **Symptom**: Rapid requests not being blocked
- **Causes**: Rate limit threshold too high, rule not deployed, wrong IP aggregation
- **Solutions**: Lower rate limit for testing, verify rule priority and action

### Validation Commands

```bash
# Check WAF Web ACL status
aws wafv2 get-web-acl \
  --scope CLOUDFRONT \
  --id YOUR-WEB-ACL-ID \
  --name WebApp-Protection-WAF \
  --region us-east-1 \
  --profile prod-admin

# Verify CloudFront distribution configuration
aws cloudfront get-distribution-config \
  --id YOUR-DISTRIBUTION-ID \
  --profile prod-admin | grep -A 5 "WebACLId"

# Check WAF logging status
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/waf/" \
  --profile prod-admin

# Test end-to-end connectivity
curl -I -s "https://YOUR-DOMAIN.cloudfront.net/" | head -10
```

---

## Security Best Practices

### WAF Rule Management
- **Start with Count mode** for new rules to identify false positives
- **Monitor sampled requests** regularly for attack pattern analysis
- **Update managed rules** automatically through AWS-managed rule groups
- **Document custom rules** with clear business justification and testing procedures

### Performance Optimization
- **Use CloudFront caching** strategically to reduce origin load
- **Monitor WAF capacity units** to ensure efficient rule processing
- **Implement request filtering** at edge locations to minimize origin impact
- **Regular performance testing** to ensure security doesn't degrade user experience

### Compliance and Audit
- **Enable comprehensive logging** for all WAF rules and CloudFront requests
- **Implement log retention** policies meeting regulatory requirements
- **Document security decisions** including rule configurations and exception rationale
- **Regular security assessments** through penetration testing and vulnerability scanning

---

## Cost Optimization

### WAF Pricing Considerations
- **Web ACL**: $1.00 per month per Web ACL
- **Rule evaluation**: $0.60 per million requests evaluated
- **Rule capacity**: Additional charges for complex custom rules
- **Logging**: CloudWatch Logs charges for ingestion and storage

### CloudFront Cost Management
- **Price class optimization**: Consider regional vs global edge location coverage
- **Caching strategy**: Optimize cache hit ratios to reduce origin requests
- **Data transfer**: Monitor outbound data transfer charges
- **Request routing**: Use CloudFront behaviors for cost-effective content delivery

---

## Success Validation Checklist

**Infrastructure Validation:**
✅ **S3 bucket secured** with OAC preventing direct access  
✅ **CloudFront distribution** operational with global edge coverage  
✅ **SSL/TLS encryption** enforced with automatic HTTP to HTTPS redirection  

**WAF Protection Validation:**
✅ **Managed rule groups** blocking OWASP Top 10 vulnerabilities  
✅ **Custom rate limiting** preventing DDoS attacks at configured thresholds  
✅ **Geographic restrictions** blocking access from specified high-risk countries  

**Security Headers Validation:**
✅ **HSTS headers** enforcing secure transport with preload inclusion  
✅ **CSP headers** preventing XSS through content loading restrictions  
✅ **Frame options** preventing clickjacking attacks  

**Attack Simulation Validation:**
✅ **XSS attempts blocked** with 403 Forbidden responses  
✅ **SQL injection blocked** before reaching origin systems  
✅ **Rate limiting triggered** at configured request thresholds  
✅ **Legitimate traffic unaffected** maintaining normal application functionality  

**Integration Validation:**
✅ **WAF logging operational** with comprehensive request analysis  
✅ **CloudWatch metrics** providing security monitoring capabilities  
✅ **Security Hub integration** ready for incident response automation  

This implementation provides production-ready web application security architecture that demonstrates enterprise-grade security engineering capabilities while preparing comprehensively for AWS Security Specialty exam Domain 3 scenarios.