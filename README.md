# AWS High Availability Web Server Infrastructure

This Terraform configuration deploys a highly available web server infrastructure on AWS using Auto Scaling Groups, Application Load Balancer, and Launch Templates.

## Architecture Overview

The infrastructure creates a fault-tolerant web application deployment across multiple availability zones in the `eu-north-1` region with the following components:

- **Application Load Balancer (ALB)** - Distributes traffic across multiple instances
- **Auto Scaling Group (ASG)** - Maintains desired number of instances and handles scaling
- **Launch Template** - Defines instance configuration for consistent deployments
- **Security Group** - Controls network access to the instances
- **Target Group** - Health checks and routing for the load balancer

## Infrastructure Components

### 1. Provider Configuration
```hcl
provider "aws" {
  region = "eu-north-1"
  default_tags {
    tags = {
      Owner     = "Denis"
      CreatedBY = "Terraform"
    }
  }
}
```
- Deploys to EU North (Stockholm) region
- Applies default tags to all resources for cost tracking and ownership

### 2. Data Sources
- **Availability Zones**: Dynamically fetches available AZs in the region
- **AMI**: Gets the latest Amazon Linux 2 AMI for consistent base images

### 3. Networking
- Uses the **default VPC** for simplicity
- Utilizes **default subnets** in two availability zones for high availability
- Ensures cross-AZ redundancy for fault tolerance

### 4. Security Group
```hcl
resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = ["80", "443", "22"]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```
**Allowed Inbound Traffic:**
- Port 80 (HTTP) - Web traffic
- Port 443 (HTTPS) - Secure web traffic  
- Port 22 (SSH) - Administrative access

**Security Note:** SSH access (port 22) is open to the world (0.0.0.0/0). Consider restricting this to specific IP ranges for production environments.

### 5. Launch Template
- **Instance Type**: t3.micro (suitable for testing/small workloads)
- **Key Pair**: "DevOps" (ensure this exists in your AWS account)
- **User Data**: Executes `user_data.sh` script during instance initialization
- **AMI**: Latest Amazon Linux 2

### 6. Auto Scaling Group
```hcl
resource "aws_autoscaling_group" "web" {
  min_size            = 2
  max_size            = 2
  min_elb_capacity    = 2
  health_check_type   = "ELB"
}
```
**Configuration:**
- Maintains exactly 2 instances across 2 AZs
- Uses ELB health checks (more sophisticated than EC2 health checks)
- Implements `create_before_destroy` lifecycle for zero-downtime updates

### 7. Application Load Balancer Setup
- **ALB**: Distributes incoming requests across healthy instances
- **Target Group**: Defines health check parameters and routing rules
- **Listener**: Routes HTTP traffic on port 80 to the target group
- **Deregistration Delay**: Set to 10 seconds for faster deployments

## Deployment Strategy

This configuration implements a **rolling deployment** strategy rather than true blue-green deployment:

1. When launch template changes, ASG creates new instances
2. New instances are health-checked before receiving traffic
3. Old instances are terminated after new ones are healthy
4. `create_before_destroy` ensures no downtime during updates

## Prerequisites

Before deploying this infrastructure, ensure you have:

1. **AWS CLI configured** with appropriate credentials
2. **Terraform installed** (version 1.0+)
3. **EC2 Key Pair** named "DevOps" in the eu-north-1 region
4. **user_data.sh script** in the same directory as your Terraform files

### Sample user_data.sh
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Web Server $(hostname -f)</h1>" > /var/www/html/index.html
```

## Deployment Commands

```bash
# Initialize Terraform
terraform init

# Plan the deployment
terraform plan

# Apply the configuration
terraform apply

# Get the load balancer URL
terraform output web_loadbalancer_url
```

## Accessing Your Application

After deployment, access your web application using the load balancer DNS name:
```
http://<load-balancer-dns-name>
```

The DNS name is provided as an output variable after successful deployment.

## High Availability Features

- **Multi-AZ Deployment**: Instances span across 2 availability zones
- **Auto Scaling**: Automatically replaces failed instances
- **Load Balancing**: Distributes traffic and performs health checks
- **ELB Health Checks**: More comprehensive than basic EC2 health checks
- **Zero Downtime Updates**: Rolling deployments with `create_before_destroy`

## Cost Considerations

- **t3.micro instances**: Eligible for AWS Free Tier
- **Application Load Balancer**: ~$16-20/month
- **Data Transfer**: Additional costs for outbound traffic
- **Default tags**: Help track costs by owner and creation method

## Security Recommendations

For production environments, consider:

1. **Restrict SSH access** to specific IP ranges or use Systems Manager Session Manager
2. **Enable HTTPS** with SSL/TLS certificates
3. **Use private subnets** for instances with NAT Gateway for outbound traffic
4. **Implement WAF** for additional application-layer protection
5. **Enable CloudTrail** for audit logging

## Monitoring and Troubleshooting

- **CloudWatch Logs**: Monitor application and system logs
- **ALB Access Logs**: Track request patterns and errors
- **CloudWatch Metrics**: Monitor ASG scaling activities and ALB performance
- **Health Check Status**: Monitor target group health in EC2 console

## Cleanup

To avoid ongoing costs, destroy the infrastructure when no longer needed:
```bash
terraform destroy
```

## Limitations

- Uses default VPC (not recommended for production)
- SSH access open to the world
- No HTTPS/SSL configuration
- Single region deployment
- Fixed scaling (min=max=2)

## True Blue-Green Deployment

For actual blue-green deployments, consider:
- Separate ASGs for blue/green environments
- Route 53 weighted routing or ALB listener rules
- Database migration strategies
- Automated rollback mechanisms