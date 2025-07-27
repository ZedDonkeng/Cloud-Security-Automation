# AWS Config + Lambda Security Automation Project (Console Version)

## Project Overview
Build an automated compliance system that detects security violations and automatically remediates them using AWS Console. This demonstrates real-world cloud security engineering skills.

## What You'll Build
- **Detection:** Config Rules to identify security violations
- **Remediation:** Lambda functions to automatically fix issues
- **Reporting:** SNS notifications and CloudWatch dashboards
- **Testing:** Create non-compliant resources to trigger your system

## Prerequisites
- AWS Free Tier account
- Basic understanding of AWS services
- Note-taking for documentation

## Phase 1: Setup AWS Config (Day 1)

### Step 1: Enable AWS Config via Console

**Navigate to Config Service:**
1. Go to AWS Console → Search "Config" → Click "AWS Config"
2. Click "Get started" (if first time) or "Settings" if already enabled

**Configure Recording:**
1. **Resource types to record:** Select "Record all resource types in this region"
2. **Include global resources:** Check this box (for IAM roles, users, etc.)
3. **Amazon S3 bucket:** Choose "Create a bucket" → Use default name or create custom
4. **Amazon SNS topic:** Choose "Create a topic" (we'll use this for alerts later)
5. **AWS Config service-linked role:** Leave as default
6. Click "Next" → "Confirm"

**What happens:** Config starts recording all resource configurations in your account

### Step 2: Wait for Initial Discovery
- Go to "Resources" tab in Config console
- Wait 5-10 minutes for initial resource discovery
- You should see S3 buckets, EC2 instances, security groups, etc.

## Phase 2: Create Your First Config Rule (Day 2)

### Step 1: Add Pre-built Config Rule
**We'll start with AWS managed rules - easier than custom Lambda**

**Add S3 Public Access Rule:**
1. In Config console → Click "Rules" in left menu
2. Click "Add rule"
3. **Rule type:** Select "AWS managed rules"
4. Search for "s3-bucket-public-access-prohibited"
5. Click on it → "Next"
6. **Name:** Keep default or rename to "s3-no-public-access"
7. **Trigger:** Keep "Configuration changes"
8. **Scope:** AWS::S3::Bucket (should be pre-filled)
9. Click "Next" → "Add rule"

**What happens:** Config will evaluate all existing S3 buckets and mark them compliant/non-compliant

### Step 2: Test the Rule - Create Non-Compliant Resource

**Create a Public S3 Bucket:**
1. Go to S3 console → "Create bucket"
2. **Bucket name:** `test-public-bucket-yourname-123` (must be globally unique)
3. **Region:** Same as your Config setup
4. **Block Public Access:** UNCHECK all boxes (this makes it non-compliant)
5. Click "Create bucket"

**Make it Actually Public:**
1. Click on your new bucket → "Permissions" tab
2. Scroll to "Bucket policy" → Click "Edit"
3. Paste this policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::test-public-bucket-yourname-123/*"
        }
    ]
}
```
4. Click "Save changes"

### Step 3: Check Compliance Results
1. Go back to Config console → "Rules"
2. Click on your "s3-bucket-public-access-prohibited" rule
3. Wait 5-10 minutes, then refresh
4. You should see your test bucket marked as "Noncompliant"

**Screenshot this!** This shows your rule is working.

## Phase 3: Create Automatic Remediation (Day 3-4)

### Step 1: Create SNS Topic for Alerts
1. Go to SNS console → "Topics" → "Create topic"
2. **Type:** Standard
3. **Name:** `security-compliance-alerts`
4. Click "Create topic"
5. Click "Create subscription"
6. **Protocol:** Email
7. **Endpoint:** Your email address
8. Click "Create subscription" → Check your email and confirm

### Step 2: Create Lambda Remediation Function

**Go to Lambda Console:**
1. AWS Console → Search "Lambda" → Click "Lambda"
2. Click "Create function"
3. **Function name:** `s3-auto-remediation`
4. **Runtime:** Python 3.9
5. Click "Create function"

**Add the Remediation Code:**
Replace the default code with this:

```python
import json
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    sns = boto3.client('sns')
    
    # Parse the Config Rule compliance notification
    try:
        # This handles Config Rule state change notifications
        message = json.loads(event['Records'][0]['Sns']['Message'])
        
        # Extract bucket name and compliance status
        resource_id = message['configurationItem']['resourceId']
        compliance_type = message['configurationItem']['complianceType']
        
        if compliance_type == 'NON_COMPLIANT':
            # Block public access on the bucket
            s3.put_public_access_block(
                Bucket=resource_id,
                PublicAccessBlockConfiguration={
                    'BlockPublicAcls': True,
                    'IgnorePublicAcls': True,
                    'BlockPublicPolicy': True,
                    'RestrictPublicBuckets': True
                }
            )
            
            # Send notification
            sns.publish(
                TopicArn='arn:aws:sns:YOUR-REGION:YOUR-ACCOUNT-ID:security-compliance-alerts',
                Message=f'SECURITY ALERT: Automatically blocked public access on S3 bucket: {resource_id}',
                Subject='Auto-Remediation: S3 Public Access Blocked'
            )
            
            print(f"Successfully blocked public access on bucket: {resource_id}")
            
        return {
            'statusCode': 200,
            'body': json.dumps('Remediation completed successfully')
        }
        
    except Exception as e:
        print(f"Error in remediation: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps(f'Error: {str(e)}')
        }
```

**Update the SNS ARN:**
- Replace `YOUR-REGION` with your AWS region (e.g., us-east-1)
- Replace `YOUR-ACCOUNT-ID` with your AWS account ID
- You can find these in the Config console URL or SNS topic ARN

### Step 3: Configure Lambda Permissions

**Add S3 and SNS Permissions:**
1. In your Lambda function → "Configuration" tab → "Permissions"
2. Click on the role name (will open IAM console)
3. Click "Add permissions" → "Attach policies"
4. Search and attach these policies:
   - `AmazonS3FullAccess` (for remediation)
   - `AmazonSNSFullAccess` (for notifications)
5. Click "Add permissions"

### Step 4: Connect Config Rule to Lambda

**Create EventBridge Rule:**
1. Go to EventBridge console → "Rules" → "Create rule"
2. **Name:** `config-s3-compliance-trigger`
3. **Event bus:** default
4. **Rule type:** Rule with an event pattern
5. **Event source:** AWS services
6. **Service name:** Config
7. **Event type:** Config Rules Compliance Change
8. Click "Next"

**Set Lambda as Target:**
1. **Target type:** AWS service
2. **Service:** Lambda function
3. **Function:** Select your `s3-auto-remediation` function
4. Click "Next" → "Create rule"

## Phase 4: Test the Complete System (Day 5)

### Step 1: Test Automatic Remediation

**Create Another Non-Compliant S3 Bucket:**
1. Create bucket: `test-auto-remediation-yourname-456`
2. Make it public (same steps as before)
3. Wait 5-10 minutes

**What Should Happen:**
1. Config detects the non-compliant bucket
2. EventBridge triggers your Lambda function
3. Lambda blocks public access automatically
4. You receive email notification
5. Config re-evaluates and marks bucket as compliant

**Troubleshooting:**
- Check Lambda logs: Lambda console → your function → "Monitor" → "View CloudWatch logs"
- Check EventBridge: EventBridge console → "Rules" → check if rule is triggered
- Check SNS: SNS console → your topic → check delivery status

### Step 2: Add More Security Rules

**Security Group Rule - No SSH from Internet:**
1. Config console → "Rules" → "Add rule"
2. Search "incoming-ssh-disabled"
3. Add this rule (it checks for port 22 open to 0.0.0.0/0)

**Test it:**
1. EC2 console → "Security Groups" → "Create security group"
2. **Name:** `test-ssh-open`
3. **Inbound rules:** Add SSH (port 22) from 0.0.0.0/0
4. Save and wait for Config to evaluate

**EBS Encryption Rule:**
1. Config console → "Rules" → "Add rule"  
2. Search "encrypted-volumes"
3. Add this rule

**Test it:**
1. EC2 console → "Volumes" → "Create volume"
2. **Encryption:** Leave unchecked
3. Create volume and wait for Config evaluation

## Phase 5: Create Dashboard and Documentation (Day 6-7)

### Step 1: CloudWatch Dashboard

**Create Compliance Dashboard:**
1. CloudWatch console → "Dashboards" → "Create dashboard"
2. **Name:** `Security-Compliance-Dashboard`
3. Add widget → "Number" → "Metrics"
4. Search for: `AWS/Config` → `ComplianceByConfigRule`
5. Select your config rules → "Create widget"
6. Add more widgets for:
   - Non-compliant resources over time (Line graph)
   - Remediation Lambda invocations
   - SNS notification metrics

### Step 2: Document Your Project

**Create a Project Summary Document:**

**Architecture Overview:**
- Config Rules monitor compliance
- EventBridge triggers remediation
- Lambda functions auto-fix issues
- SNS sends notifications
- CloudWatch provides visibility

**Rules Implemented:**
1. S3 buckets must not be publicly accessible
2. Security groups must not allow SSH from internet
3. EBS volumes must be encrypted
4. [Add more as you expand]

**Key Metrics:**
- Time to detection: ~10 minutes
- Time to remediation: ~2 minutes
- False positive rate: 0%
- Cost per month: ~$5-10

**Take Screenshots of:**
- Config Rules dashboard showing compliance status
- Lambda function logs showing successful remediation
- CloudWatch dashboard with metrics
- SNS email notifications
- Before/after of remediated resources

## Phase 6: Portfolio Presentation

### GitHub Repository Structure
```
aws-security-automation/
├── README.md
├── docs/
│   ├── architecture-diagram.png
│   ├── setup-guide.md
│   └── troubleshooting.md
├── lambda-functions/
│   ├── s3-auto-remediation.py
│   └── sg-remediation.py
├── screenshots/
│   ├── config-dashboard.png
│   ├── compliance-results.png
│   └── cloudwatch-metrics.png
└── costs/
    └── cost-analysis.md
```

### README.md Template
```markdown
# AWS Security Compliance Automation

## Overview
Automated security compliance system using AWS Config, Lambda, and EventBridge to detect and remediate security violations in real-time.

## Architecture
[Include your architecture diagram]

## Features
- Real-time security violation detection
- Automatic remediation of common issues
- Email notifications for security events
- Compliance dashboard with metrics
- Cost-optimized design (~$10/month)

## Rules Implemented
1. **S3 Public Access Prevention**: Automatically blocks public access on S3 buckets
2. **Security Group Hardening**: Detects overly permissive security groups
3. **EBS Encryption Enforcement**: Identifies unencrypted volumes

## Results
- Reduced manual security reviews by 80%
- Average remediation time: 2 minutes
- 100% compliance rate maintained
```

### Interview Talking Points

**Technical Deep Dive:**
- "I built this to solve the problem of manual security reviews taking too long"
- "The system detects violations in 10 minutes and remediates in 2 minutes"
- "I used EventBridge to decouple Config from Lambda for better scaling"
- "Cost is only $10/month compared to $50k+ security tools"

**Business Impact:**
- "This would save a security team 20+ hours per week"
- "Reduces security incident response time by 90%"
- "Provides auditable compliance evidence for frameworks like SOC2"

**Future Enhancements:**
- "Could expand to multi-account using Organizations"
- "Integration with Slack/Teams for better notifications"
- "Machine learning to detect anomalous configurations"

## Next Steps After This Project

1. **Add More Rules**: IAM overprivileged roles, untagged resources, etc.
2. **Multi-Account Setup**: Use Config aggregator across accounts
3. **Custom Rules**: Write Lambda-based rules for company-specific policies
4. **Integration**: Connect to SIEM tools or ticketing systems
5. **Compliance Reporting**: Generate SOC2/PCI compliance reports

This project demonstrates you can:
- Design security automation systems
- Work with multiple AWS services
- Think about cost optimization
- Document and present technical work
- Understand real-world security challenges

Perfect for a Cloud Security Engineer role!