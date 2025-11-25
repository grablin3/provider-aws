# AWS Infrastructure Provider

This module sets up complete AWS infrastructure using Terraform, including EC2 deployment, RDS databases, networking, and CI/CD automation via GitHub Actions.

## What This Does

This provider module creates:
- AWS infrastructure with Terraform
- EC2 instances for application deployment
- RDS database instances (optional)
- VPC, subnets, security groups, and networking
- Load balancers and auto-scaling
- GitHub Actions workflows for deployment
- Multi-environment support (dev, staging, production)

---

## Prerequisites

Before configuring this module, you need:
1. An AWS account (sign up at https://aws.amazon.com)
2. AWS IAM user with admin permissions for GitHub Actions
3. Basic understanding of AWS regions

---

## Configuration Fields

### AWS Region `awsRegion`
**What it is**: The geographic region where your AWS resources will be deployed.

**Options**:

| Region Code | Name | Notes |
|-------------|------|-------|
| `us-east-1` | US East (N. Virginia) | Most services, lowest cost |
| `us-west-1` | US West (N. California) | Good for West Coast |
| `us-west-2` | US West (Oregon) | Good for West Coast |
| `eu-west-1` | Europe (Ireland) | Good for European users |
| `ap-southeast-1` | Asia Pacific (Singapore) | Good for Asian users |

**How to choose**:
- Choose a region closest to your users for lower latency
- Check AWS pricing - some regions are cheaper than others
- Ensure the region supports all services you need

**Default**: `us-west-1`

---

### AWS Account ID `awsAccountId`
**What it is**: Your 12-digit AWS account identifier.

**How to find it**:
1. Log into AWS Console (https://console.aws.amazon.com)
2. Click your account name in the top-right corner
3. Your Account ID is displayed (12 digits, e.g., `123456789012`)

**Alternative method**:
- Run AWS CLI command: `aws sts get-caller-identity --query Account --output text`

**Format**: 12-digit number (e.g., `123456789012`)

**Why it's needed**: Used for creating resource ARNs and IAM permissions.

---

### AWS Admin Access Key ID `awsAdminId`
**What it is**: The access key ID for an IAM user with administrator permissions, used by GitHub Actions to deploy infrastructure.

**How to create it**:

#### **Log into AWS Console**
   - Go to https://console.aws.amazon.com
   - Navigate to IAM service (search for "IAM" in the top search bar)

#### **Create IAM User** (if you don't have one):
   - Click "Users" in the left sidebar
   - Click "Create user"
   - Username: `github-actions-admin` (or any name you prefer)
   - Click "Next"

#### **Attach Administrator Policy**:
   - Select "Attach policies directly"
   - Search for `AdministratorAccess`
   - Check the box next to it
   - Click "Next" → "Create user"

#### **Create Access Key**:
   - Click on the user you just created
   - Go to "Security credentials" tab
   - Scroll to "Access keys" section
   - Click "Create access key"
   - Select "Application running outside AWS"
   - Click "Next" → "Create access key"
   - **IMPORTANT**: Copy the "Access key" (this is your `awsAdminId`)
   - Click "Download .csv file" to save both keys securely

**Format**: Starts with `AKIA...` (20 characters, e.g., `AKIAIOSFODNN7EXAMPLE`)

**Security Warning**: 
- Keep this secret! Never commit it to your code repository
- This key has full admin access to your AWS account
- Genesis will store it as a GitHub secret for CI/CD

---

### AWS Admin Secret Access Key `awsAdminSecret`
**What it is**: The secret key that pairs with the access key ID, used for authentication.

**How to get it**:
- When you create the access key (see previous step), AWS shows both:
  - Access key ID (→ `awsAdminId`)
  - Secret access key (→ `awsAdminSecret`)
- You can only see the secret access key **once** when creating it
- If you lose it, you must create a new access key

**Format**: 40-character alphanumeric string (e.g., `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`)

**Security Warning**: 
- **NEVER** share this or commit it to code
- If compromised, delete the access key immediately in IAM console
- Genesis will store it as a GitHub secret for CI/CD

---

### EC2 Instance Type `instanceType`
**What it is**: The size/capacity of the virtual server that will run your application.

**Options**:
- `t2.nano` - 1 vCPU, 0.5 GB RAM (~$4/month) - Minimal apps only
- `t2.micro` - 1 vCPU, 1 GB RAM (~$8/month) - **Recommended for dev/staging**
- `t2.small` - 1 vCPU, 2 GB RAM (~$17/month) - Good for small production
- `t2.medium` - 2 vCPU, 4 GB RAM (~$34/month) - Medium traffic production

**How to choose**:
- **Development/Testing**: `t2.micro` (AWS Free Tier eligible)
- **Small production** (< 100 concurrent users): `t2.small`
- **Medium production** (100-1000 users): `t2.medium`
- **High traffic**: Consider larger instances (t3.large, t3.xlarge, etc.)

**Pricing**: Check current prices at https://aws.amazon.com/ec2/pricing/on-demand/

**Default**: `t2.micro`

**Note**: AWS Free Tier includes 750 hours/month of t2.micro for the first 12 months.

---

### RDS Instance Class `databaseClass`
**What it is**: The size of your managed PostgreSQL/MySQL database instance (if using a database).

**Options**:
- `db.t3.micro` - 2 vCPU, 1 GB RAM (~$15/month) - **Recommended for dev/staging**
- `db.t3.small` - 2 vCPU, 2 GB RAM (~$30/month) - Small production
- `db.t3.medium` - 2 vCPU, 4 GB RAM (~$60/month) - Medium production

**How to choose**:
- **Development/Testing**: `db.t3.micro`
- **Small production**: `db.t3.small`
- **Medium production**: `db.t3.medium`

**When to use**:
- Only needed if you selected a database extension (e.g., `extension-rdbms`)
- If not using a database, this field is optional

**Default**: `db.t3.micro`

**Note**: AWS Free Tier includes 750 hours/month of db.t2.micro (older generation) for first 12 months.

---

## What Gets Created

When you deploy with this provider, AWS resources created include:

### Networking:
- **VPC** (Virtual Private Cloud) - Your isolated network
- **Subnets** - Public and private subnets in multiple availability zones
- **Internet Gateway** - Allow internet access
- **NAT Gateway** - Allow private instances to access internet
- **Route Tables** - Network routing configuration
- **Security Groups** - Firewall rules

### Compute:
- **EC2 Instances** - Virtual servers running your application
- **Auto Scaling Group** - Automatically scale instances based on load
- **Application Load Balancer** - Distribute traffic across instances
- **Target Groups** - Route traffic to healthy instances

### Database (if using database extension):
- **RDS Instance** - Managed PostgreSQL/MySQL database
- **DB Subnet Group** - Database networking
- **DB Security Group** - Database firewall rules

### Storage:
- **S3 Buckets** - Static file storage (if using frontend)
- **CloudFront Distribution** - CDN for frontend assets

### CI/CD:
- **GitHub Actions Workflows** - Automated deployment pipelines
- **GitHub Secrets** - Securely stored AWS credentials

---

## Environment-Specific Defaults

Genesis automatically adjusts instance sizes based on environment:

| Environment | EC2 Instance | RDS Instance | Purpose |
|-------------|--------------|--------------|---------|
| dev | `t2.nano` | `db.t3.micro` | Development/testing |
| staging | `t2.micro` | `db.t3.micro` | Pre-production validation |
| prod | `t2.small` | `db.t3.small` | Production workloads |

You can override these in your project configuration.

---

## Common Issues

### "Access Denied" Errors
**Problem**: AWS IAM permissions insufficient.

**Solutions**:
- Verify IAM user has `AdministratorAccess` policy attached
- Check access key is active in IAM console
- Ensure you're using the correct AWS account
- Try creating a new access key

### "Region Not Available"
**Problem**: Selected region doesn't support required services.

**Solutions**:
- Choose a major region (us-east-1, us-west-2, eu-west-1)
- Check AWS service availability: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/

### "Limit Exceeded" Errors
**Problem**: AWS account limits reached (e.g., max VPCs, EC2 instances).

**Solutions**:
- Check your account limits in AWS Console → Service Quotas
- Request limit increase: https://console.aws.amazon.com/servicequotas
- Delete unused resources to free up capacity
- New AWS accounts have lower limits initially

### Terraform State Issues
**Problem**: Terraform state file conflicts or corruption.

**Solutions**:
- Never manually edit `*.tfstate` files
- Use Terraform remote state (S3 backend)
- Run `terraform refresh` to sync state
- If corrupted, restore from backup or recreate

### High Costs
**Problem**: Unexpected AWS charges.

**Solutions**:
- Use t2/t3 instance types (burstable, cheaper)
- Stop/terminate unused instances
- Use AWS Free Tier when eligible
- Set up AWS Budget Alerts: https://console.aws.amazon.com/billing/home#/budgets
- Delete old EBS snapshots
- Use Reserved Instances for production (1-3 year commitment for discounts)

---

## Cost Estimation

Approximate monthly costs for a basic setup:

**Development** (t2.micro + db.t3.micro):
- EC2: ~$8/month
- RDS: ~$15/month
- Networking: ~$5/month
- **Total**: ~$28/month

**Production** (t2.small + db.t3.small + load balancer):
- EC2: ~$17/month
- RDS: ~$30/month
- Load Balancer: ~$20/month
- Networking: ~$10/month
- **Total**: ~$77/month

**Note**: 
- Costs vary by region
- AWS Free Tier can reduce costs significantly (first 12 months)
- Use AWS Pricing Calculator for accurate estimates: https://calculator.aws/

---

## Best Practices

1. **Security**:
   - Rotate AWS access keys every 90 days
   - Use IAM roles for EC2 instances (not hard-coded keys)
   - Enable MFA on AWS root account
   - Follow least-privilege principle

2. **Cost Management**:
   - Tag all resources with environment and project name
   - Set up AWS Budget Alerts
   - Review AWS Cost Explorer monthly
   - Delete unused resources promptly

3. **High Availability**:
   - Use multiple availability zones
   - Enable auto-scaling for production
   - Use RDS Multi-AZ for production databases
   - Implement health checks

4. **Monitoring**:
   - Enable CloudWatch monitoring
   - Set up alarms for critical metrics
   - Review logs regularly
   - Use AWS X-Ray for debugging

---

## Additional Resources

- **AWS Console**: https://console.aws.amazon.com
- **AWS Documentation**: https://docs.aws.amazon.com
- **AWS Free Tier**: https://aws.amazon.com/free/
- **AWS Pricing**: https://aws.amazon.com/pricing/
- **IAM Best Practices**: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- **EC2 Instance Types**: https://aws.amazon.com/ec2/instance-types/
- **RDS Pricing**: https://aws.amazon.com/rds/pricing/
- **Terraform AWS Provider**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
