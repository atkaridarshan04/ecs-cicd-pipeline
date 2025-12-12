# VPC Endpoints Migration

## Overview
This project has been updated to use **VPC Endpoints** instead of **NAT Gateways** for AWS service communication from private subnets. This change provides better security, performance, and cost optimization.

## Changes Made

### 1. New VPC Endpoints Configuration (`vpc_endpoints.tf`)
- **ECR API Endpoint**: For ECR API calls
- **ECR DKR Endpoint**: For Docker registry operations  
- **S3 Gateway Endpoint**: For S3 access (ECR layers stored in S3)
- **CloudWatch Logs Endpoint**: For ECS task logging
- **VPC Endpoints Security Group**: Allows HTTPS traffic from private subnets

### 2. Updated Network Configuration (`main.tf`)
- **Commented out NAT Gateway resources** with clear instructions for re-enabling
- **Removed default internet route** from private route tables
- **Updated ECS security group** to allow outbound HTTPS to VPC endpoints only

### 3. Benefits of VPC Endpoints

| Aspect | NAT Gateway | VPC Endpoints |
|--------|-------------|---------------|
| **Cost** | ~$45/month per AZ | ~$7/month per endpoint |
| **Security** | Internet routing | Private AWS network |
| **Performance** | Internet latency | Direct AWS backbone |
| **Availability** | Single point of failure | Highly available |

## Architecture Changes

### Before (NAT Gateway)
```
ECS Tasks → NAT Gateway → Internet → ECR/S3/CloudWatch
```

### After (VPC Endpoints)
```
ECS Tasks → VPC Endpoints → AWS Services (private network)
```

## When to Re-enable NAT Gateway

Uncomment the NAT Gateway resources in `main.tf` if you need:
- Access to 3rd party APIs without VPC endpoints
- Software package downloads (apt, yum, npm, etc.)
- External service integrations
- Internet-based monitoring tools

## Cost Savings

**Monthly cost reduction**: ~$80-90 (2 NAT Gateways) vs ~$28 (4 VPC Endpoints)
**Annual savings**: ~$600-750

## Security Improvements

- ✅ No internet routing for AWS service communication
- ✅ Traffic stays within AWS private network
- ✅ Reduced attack surface
- ✅ Better compliance with security frameworks

## Deployment Notes

1. **Existing deployments**: Run `terraform plan` to see changes before applying
2. **New deployments**: VPC endpoints will be created automatically
3. **Rollback**: Uncomment NAT Gateway resources if needed
4. **Monitoring**: Check VPC endpoint metrics in CloudWatch

## Troubleshooting

If ECS tasks fail to start after migration:
1. Verify VPC endpoints are in "Available" state
2. Check security group rules allow HTTPS (443) traffic
3. Ensure DNS resolution is enabled in VPC
4. Review CloudWatch logs for specific errors
