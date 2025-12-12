# CloudWatch Logging Analysis - ECS CI/CD Pipeline

## Overview
This document details all services in your architecture that send logs to CloudWatch and their configurations.

## Services Sending Logs to CloudWatch

### 1. ECS Fargate Tasks ✅ **CONFIGURED**

**Configuration Location**: `ecs.tf`
```hcl
# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.project_name}"
  retention_in_days = 7
}

# Container logging configuration
logConfiguration = {
  logDriver = "awslogs"
  options = {
    awslogs-group         = aws_cloudwatch_log_group.app.name
    awslogs-region        = var.aws_region
    awslogs-stream-prefix = "ecs"
  }
}
```

**Log Details**:
- **Log Group**: `/ecs/ecs-cicd-pipeline`
- **Log Streams**: `ecs/app-container/{task-id}`
- **Retention**: 7 days
- **Content**: Application logs, container stdout/stderr
- **Access**: Via VPC Endpoint for CloudWatch Logs

### 2. ECS Cluster (Container Insights) ✅ **CONFIGURED**

**Configuration Location**: `ecs.tf`
```hcl
# Enable CloudWatch Container Insights
setting {
  name  = "containerInsights"
  value = "enabled"
}
```

**Metrics Collected**:
- CPU and memory utilization
- Network metrics
- Task and service metrics
- Performance monitoring data

### 3. CodeBuild ✅ **CONFIGURED** (Default)

**Configuration**: Uses AWS default CloudWatch logging
- **Log Group**: `/aws/codebuild/{project-name}`
- **Log Streams**: Auto-generated per build
- **Content**: Build process logs, Docker build output
- **Retention**: AWS default (indefinite)

**IAM Permissions** (in `iam.tf`):
```hcl
# CodeBuild CloudWatch permissions
"logs:CreateLogGroup",
"logs:CreateLogStream", 
"logs:PutLogEvents"
```

### 4. Application Load Balancer ❌ **NOT CONFIGURED**

**Current Status**: No access logs configured
**Recommendation**: Enable ALB access logs to S3

### 5. CodePipeline ❌ **NOT CONFIGURED**

**Current Status**: Uses CloudTrail for API calls only
**Recommendation**: Pipeline execution logs available in console only

## Missing CloudWatch Integrations

### 1. ALB Access Logs (Recommended)
```hcl
# Add to alb.tf
resource "aws_lb" "main" {
  # ... existing config ...
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    prefix  = "alb-logs"
    enabled = true
  }
}
```

### 2. VPC Flow Logs (Optional)
```hcl
# Add to main.tf for network monitoring
resource "aws_flow_log" "vpc" {
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
}
```

### 3. CodeBuild Custom Log Group (Optional)
```hcl
# Add to codebuild.tf for better log management
logs_config {
  cloudwatch_logs {
    group_name  = "/aws/codebuild/${var.project_name}"
    stream_name = "build-logs"
  }
}
```

## Current Log Flow Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   ECS Tasks     │───▶│  VPC Endpoint    │───▶│   CloudWatch    │
│  (App Logs)     │    │    (Logs)        │    │   Log Groups    │
└─────────────────┘    └──────────────────┘    └─────────────────┘

┌─────────────────┐                             ┌─────────────────┐
│   CodeBuild     │────────────────────────────▶│   CloudWatch    │
│  (Build Logs)   │         Internet            │   Log Groups    │
└─────────────────┘                             └─────────────────┘

┌─────────────────┐                             ┌─────────────────┐
│ ECS Container   │────────────────────────────▶│   CloudWatch    │
│   Insights      │      AWS Backbone           │    Metrics      │
└─────────────────┘                             └─────────────────┘
```

## Log Retention & Costs

| Service | Log Group | Retention | Est. Monthly Cost |
|---------|-----------|-----------|-------------------|
| ECS Tasks | `/ecs/ecs-cicd-pipeline` | 7 days | $2-5 |
| CodeBuild | `/aws/codebuild/ecs-cicd-pipeline-build` | Indefinite | $1-3 |
| Container Insights | Multiple metric streams | 15 months | $5-10 |

**Total Estimated**: $8-18/month

## Monitoring Recommendations

### 1. CloudWatch Alarms
- ECS task failure rate
- High CPU/memory usage
- CodeBuild failure rate
- ALB 5xx error rate

### 2. CloudWatch Dashboards
- ECS service health
- Pipeline execution status
- Application performance metrics

### 3. Log Insights Queries
```sql
-- ECS application errors
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc

-- CodeBuild failures
fields @timestamp, @message
| filter @message like /FAILED/
| stats count() by bin(5m)
```

## Security Considerations

- ✅ **VPC Endpoint**: Logs sent via private network
- ✅ **IAM Roles**: Least privilege access
- ✅ **Encryption**: CloudWatch logs encrypted at rest
- ⚠️ **Log Retention**: Consider compliance requirements

## Troubleshooting Log Issues

1. **ECS logs not appearing**:
   - Check VPC endpoint status
   - Verify security group allows HTTPS (443)
   - Confirm task execution role permissions

2. **CodeBuild logs missing**:
   - Check IAM permissions for logs
   - Verify build project configuration

3. **High CloudWatch costs**:
   - Reduce log retention periods
   - Filter verbose application logs
   - Use log sampling for high-volume apps
