
# AWS Terraform Infrastructure Management Lab

## Overview
This hands-on lab demonstrates Infrastructure as Code (IaC) principles using Terraform to manage AWS resources. You'll learn to provision, manage, and import cloud infrastructure programmatically.

## Lab Environment Setup

### Connecting to Your Development Environment
Your AWS EC2 instance hosts a cloud-based VS Code environment for this lab.

**Connection Steps:**
1. Navigate to AWS Management Console
2. Select **EC2** service from the services menu
3. Click **Running Instances** from the left sidebar
4. Locate your assigned instance and copy its **Public IPv4 DNS**
5. Launch a new browser window and navigate to: `https://[your-public-ip]:8080`
6. Accept the certificate warning (development environment uses self-signed certificates)
7. Enter credentials when prompted: **Password:** `Linux4All!`

### Workspace Preparation
Once VS Code loads in your browser:
1. Access the integrated terminal: **View** → **Terminal** (or `Ctrl+Shift+Backtick`)
2. Create your project directory:
   ```bash
   mkdir aws-terraform-lab && cd aws-terraform-lab
   ```
3. Open this directory as your workspace: **File** → **Open Folder**

---

## Module 1: Terraform Foundation & Provider Configuration

### Infrastructure Definition File
Create your primary Terraform configuration file `infrastructure.tf`:

```hcl
# Configure Terraform and required providers
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

# Configure AWS provider
provider "aws" {
  region = "us-east-1"
}
```

### Initialize Terraform Environment
Execute the initialization command to download required providers:
```bash
terraform init
```

**Expected Output:** Terraform will download and install the AWS and Random providers, creating a `.terraform` directory and lock file.

---

## Module 2: Resource Provisioning with S3 Storage

### Dynamic Resource Naming
Create a unique identifier for resource naming to avoid conflicts:

```hcl
# Generate random string for unique resource naming
resource "random_string" "bucket_identifier" {
  length  = 6
  special = false
  upper   = false
}
```

### S3 Bucket Resource Definition
Add the S3 bucket configuration to your `infrastructure.tf`:

```hcl
# Primary S3 storage bucket
resource "aws_s3_bucket" "primary_storage" {
  bucket = "terraform-lab-storage-${random_string.bucket_identifier.result}"
  
  tags = {
    Project     = "TerraformLab"
    ManagedBy   = "Terraform"
    CreatedDate = timestamp()
  }
}

# Bucket versioning configuration
resource "aws_s3_bucket_versioning" "primary_versioning" {
  bucket = aws_s3_bucket.primary_storage.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### Deployment Execution
1. **Review planned changes:**
   ```bash
   terraform plan -out=deployment.tfplan
   ```

2. **Apply the configuration:**
   ```bash
   terraform apply deployment.tfplan
   ```

### Verification Process
Confirm resource creation in AWS Console:
- Navigate to **S3** service
- Verify your bucket appears in the bucket list
- Check the tags match your configuration

---

## Module 3: Multi-Environment Management with Workspaces

### Environment Workspace Creation
Terraform workspaces enable multiple environment management from a single configuration.

```bash
# Create environment-specific workspaces
terraform workspace new development
terraform workspace new testing  
terraform workspace new production

# View available workspaces
terraform workspace list

# Switch between environments
terraform workspace select development
```

### Environment-Aware Configuration
Modify your bucket resource to include workspace context:

```hcl
resource "aws_s3_bucket" "primary_storage" {
  bucket = "terraform-lab-${terraform.workspace}-${random_string.bucket_identifier.result}"
  
  tags = {
    Project     = "TerraformLab"
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
    CreatedDate = timestamp()
  }
}
```

### Multi-Environment Deployment
Deploy to each environment:

```bash
# Deploy to development
terraform workspace select development
terraform apply -auto-approve

# Deploy to testing
terraform workspace select testing  
terraform apply -auto-approve

# Deploy to production
terraform workspace select production
terraform apply -auto-approve
```

**Result:** Three separate S3 buckets, one for each environment, with appropriate naming and tagging.

---

## Module 4: External Resource Integration

### Identifying Existing Resources
Before importing, locate existing AWS resources that need Terraform management.

1. In AWS Console, navigate to **S3**
2. Identify any pre-existing buckets (look for names like `existing-company-bucket-*`)
3. Note the exact bucket name for import

### Import Configuration Block
Add an import configuration to your `infrastructure.tf`:

```hcl
# Import existing S3 bucket into Terraform management
import {
  to = aws_s3_bucket.legacy_storage
  id = "existing-company-bucket-abc123"  # Replace with actual bucket name
}

# Define the imported resource
resource "aws_s3_bucket" "legacy_storage" {
  bucket = "existing-company-bucket-abc123"  # Must match imported resource
  
  tags = {
    Project     = "TerraformLab"
    Environment = "imported"
    ManagedBy   = "Terraform"
    Status      = "Migrated"
  }
}
```

### Execute Import Process
1. **Plan the import:**
   ```bash
   terraform plan
   ```

2. **Execute the import:**
   ```bash
   terraform apply -auto-approve
   ```

### State Synchronization
After manual changes are made to AWS resources outside Terraform:

```bash
# Detect configuration drift
terraform plan -refresh-only

# Synchronize state with actual infrastructure
terraform apply -refresh-only -auto-approve
```

---

## Module 5: Advanced Configuration Management

### Static Website Hosting Setup
Enhance the imported bucket with web hosting capabilities:

```hcl
resource "aws_s3_bucket_website_configuration" "legacy_website" {
  bucket = aws_s3_bucket.legacy_storage.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

# Output the website endpoint
output "website_url" {
  description = "S3 bucket website endpoint"
  value       = aws_s3_bucket_website_configuration.legacy_website.website_endpoint
}
```

### Public Access Configuration
Configure bucket for public website hosting:

```hcl
resource "aws_s3_bucket_public_access_block" "legacy_public_access" {
  bucket = aws_s3_bucket.legacy_storage.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

---

## Lab Validation and Cleanup

### Resource Verification
Confirm all resources are properly managed:

```bash
# Show current state
terraform show

# List all resources
terraform state list

# Validate configuration
terraform validate
```

### Environment Cleanup
Remove resources when lab is complete:

```bash
# Destroy resources in current workspace
terraform destroy -auto-approve

# Clean up all workspaces
terraform workspace select default
terraform workspace delete development
terraform workspace delete testing
terraform workspace delete production
```

---

## Key Learning Outcomes

This lab demonstrates essential Infrastructure as Code concepts:

- **Declarative Infrastructure:** Define desired state rather than step-by-step procedures
- **Environment Isolation:** Use workspaces to maintain separate environments
- **State Management:** Track resource lifecycles and detect configuration drift  
- **Resource Import:** Bring existing infrastructure under Terraform control
- **Configuration Synchronization:** Align Terraform state with actual infrastructure

### Best Practices Demonstrated
- Unique resource naming to prevent conflicts
- Consistent tagging for resource organization
- Environment-specific configurations using workspace variables
- Import workflows for legacy resource adoption
- State refresh procedures for drift detection

---

## Troubleshooting Guide

**Common Issues:**
- **Provider Download Failures:** Ensure internet connectivity and retry `terraform init`
- **Permission Errors:** Verify AWS credentials and IAM permissions
- **Resource Conflicts:** Check for naming collisions with existing resources
- **State Lock Issues:** Use `terraform force-unlock` if state becomes locked

**Additional Resources:**
- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Workspace Management](https://developer.hashicorp.com/terraform/language/state/workspaces)
- [AWS S3 Service Documentation](https://docs.aws.amazon.com/s3/)

---

*This lab provides hands-on experience with enterprise Infrastructure as Code practices using Terraform and AWS services.*
