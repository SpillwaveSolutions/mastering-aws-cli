# Elastic Container Service (ECS)

## Clusters

### Create Cluster
```bash
# Fargate-only cluster with Container Insights
aws ecs create-cluster \
    --cluster-name production \
    --capacity-providers FARGATE FARGATE_SPOT \
    --default-capacity-provider-strategy \
        capacityProvider=FARGATE,weight=1,base=1 \
        capacityProvider=FARGATE_SPOT,weight=4 \
    --settings name=containerInsights,value=enabled

# EC2 cluster with managed scaling
aws ecs create-cluster \
    --cluster-name ec2-cluster \
    --settings name=containerInsights,value=enabled
```

### Manage Clusters
```bash
# List clusters
aws ecs list-clusters

# Describe cluster
aws ecs describe-clusters --clusters production

# Delete cluster (must be empty)
aws ecs delete-cluster --cluster production
```

### Capacity Providers
```bash
# Create capacity provider for EC2 Auto Scaling group
aws ecs create-capacity-provider \
    --name my-ec2-capacity \
    --auto-scaling-group-provider \
        autoScalingGroupArn=arn:aws:autoscaling:us-east-1:123456789012:autoScalingGroup:abc123:autoScalingGroupName/my-asg,\
managedScaling='{status=ENABLED,targetCapacity=100}',\
managedTerminationProtection=ENABLED

# Update cluster capacity providers
aws ecs put-cluster-capacity-providers \
    --cluster production \
    --capacity-providers FARGATE FARGATE_SPOT my-ec2-capacity \
    --default-capacity-provider-strategy \
        capacityProvider=FARGATE,weight=1
```

## Task Definitions

### Register Task Definition
```bash
aws ecs register-task-definition \
    --cli-input-json file://task-def.json
```

**task-def.json (Fargate):**
```json
{
    "family": "web-api",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
    "containerDefinitions": [
        {
            "name": "app",
            "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
            "essential": true,
            "portMappings": [
                {"containerPort": 8080, "protocol": "tcp"}
            ],
            "environment": [
                {"name": "NODE_ENV", "value": "production"}
            ],
            "secrets": [
                {
                    "name": "DB_PASSWORD",
                    "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-password"
                },
                {
                    "name": "API_KEY",
                    "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/api-key"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/web-api",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "healthCheck": {
                "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
                "interval": 30,
                "timeout": 5,
                "retries": 3,
                "startPeriod": 60
            }
        }
    ]
}
```

### Manage Task Definitions
```bash
# List task definition families
aws ecs list-task-definition-families

# List task definitions
aws ecs list-task-definitions --family-prefix web-api

# Describe task definition
aws ecs describe-task-definition --task-definition web-api:1

# Deregister (soft delete)
aws ecs deregister-task-definition --task-definition web-api:1
```

## Services

### Create Service
```bash
# Fargate service with ALB
aws ecs create-service \
    --cluster production \
    --service-name web-api-svc \
    --task-definition web-api:1 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration \
        "awsvpcConfiguration={subnets=[subnet-1,subnet-2],securityGroups=[sg-1],assignPublicIp=DISABLED}" \
    --load-balancers \
        "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/abc123,containerName=app,containerPort=8080" \
    --health-check-grace-period-seconds 60 \
    --enable-execute-command

# With capacity provider strategy
aws ecs create-service \
    --cluster production \
    --service-name web-api-svc \
    --task-definition web-api:1 \
    --desired-count 4 \
    --capacity-provider-strategy \
        capacityProvider=FARGATE,weight=1,base=1 \
        capacityProvider=FARGATE_SPOT,weight=4 \
    --network-configuration \
        "awsvpcConfiguration={subnets=[subnet-1,subnet-2],securityGroups=[sg-1]}"
```

### Update Service
```bash
# Deploy new task definition
aws ecs update-service \
    --cluster production \
    --service web-api-svc \
    --task-definition web-api:2

# Force new deployment (pull latest image)
aws ecs update-service \
    --cluster production \
    --service web-api-svc \
    --force-new-deployment

# Scale
aws ecs update-service \
    --cluster production \
    --service web-api-svc \
    --desired-count 5

# Update deployment configuration
aws ecs update-service \
    --cluster production \
    --service web-api-svc \
    --deployment-configuration \
        "minimumHealthyPercent=50,maximumPercent=200,deploymentCircuitBreaker={enable=true,rollback=true}"
```

### List and Describe Services
```bash
# List services
aws ecs list-services --cluster production

# Describe service
aws ecs describe-services \
    --cluster production \
    --services web-api-svc

# Get service events
aws ecs describe-services \
    --cluster production \
    --services web-api-svc \
    --query 'services[0].events[:10]'
```

### Delete Service
```bash
# Scale to 0 first
aws ecs update-service \
    --cluster production \
    --service web-api-svc \
    --desired-count 0

# Then delete
aws ecs delete-service \
    --cluster production \
    --service web-api-svc

# Or force delete (stops tasks immediately)
aws ecs delete-service \
    --cluster production \
    --service web-api-svc \
    --force
```

## Tasks

### Run Standalone Task
```bash
aws ecs run-task \
    --cluster production \
    --task-definition migration-task:1 \
    --launch-type FARGATE \
    --network-configuration \
        "awsvpcConfiguration={subnets=[subnet-1],securityGroups=[sg-1],assignPublicIp=ENABLED}" \
    --overrides \
        '{"containerOverrides":[{"name":"app","command":["python","migrate.py"]}]}'
```

### List and Stop Tasks
```bash
# List running tasks
aws ecs list-tasks --cluster production

# List tasks for service
aws ecs list-tasks --cluster production --service-name web-api-svc

# Describe tasks
aws ecs describe-tasks \
    --cluster production \
    --tasks arn:aws:ecs:us-east-1:123456789012:task/production/abc123

# Stop task
aws ecs stop-task \
    --cluster production \
    --task arn:aws:ecs:us-east-1:123456789012:task/production/abc123 \
    --reason "Debugging"
```

## ECS Exec (Container Debugging)

### Enable ECS Exec
Service must be created with `--enable-execute-command`:
```bash
aws ecs update-service \
    --cluster production \
    --service web-api-svc \
    --enable-execute-command
```

### Execute Commands
```bash
# Interactive shell
aws ecs execute-command \
    --cluster production \
    --task arn:aws:ecs:us-east-1:123456789012:task/production/abc123 \
    --container app \
    --interactive \
    --command "/bin/sh"

# Run single command
aws ecs execute-command \
    --cluster production \
    --task arn:aws:ecs:us-east-1:123456789012:task/production/abc123 \
    --container app \
    --interactive \
    --command "cat /app/config.json"
```

### Check ECS Exec Readiness
```bash
# Install amazon-ecs-exec-checker
aws ecs describe-tasks \
    --cluster production \
    --tasks abc123 \
    --query 'tasks[0].containers[0].managedAgents'
```

## Service Auto Scaling

### Register Scalable Target
```bash
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/production/web-api-svc \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 2 \
    --max-capacity 10
```

### Target Tracking Policy
```bash
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/production/web-api-svc \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name cpu-scaling \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration \
        "TargetValue=70.0,PredefinedMetricSpecification={PredefinedMetricType=ECSServiceAverageCPUUtilization},ScaleOutCooldown=60,ScaleInCooldown=60"
```

### Step Scaling Policy
```bash
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/production/web-api-svc \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name memory-step-scaling \
    --policy-type StepScaling \
    --step-scaling-policy-configuration \
        "AdjustmentType=ChangeInCapacity,StepAdjustments=[{MetricIntervalLowerBound=0,ScalingAdjustment=2}],Cooldown=60"
```

## Blue-Green Deployments (with CodeDeploy)

### Create Deployment Group
```bash
aws deploy create-deployment-group \
    --application-name my-ecs-app \
    --deployment-group-name my-dg \
    --service-role-arn arn:aws:iam::123456789012:role/CodeDeployRole \
    --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
    --ecs-services clusterName=production,serviceName=web-api-svc \
    --load-balancer-info \
        "targetGroupPairInfoList=[{targetGroups=[{name=blue-tg},{name=green-tg}],prodTrafficRoute={listenerArns=[arn:aws:elasticloadbalancing:...]}}]" \
    --blue-green-deployment-configuration \
        "terminateBlueInstancesOnDeploymentSuccess={action=TERMINATE,terminationWaitTimeInMinutes=5},deploymentReadyOption={actionOnTimeout=CONTINUE_DEPLOYMENT}"
```

## Useful Queries

```bash
# Get task private IPs
aws ecs describe-tasks \
    --cluster production \
    --tasks $(aws ecs list-tasks --cluster production --service-name web-api-svc --query 'taskArns[*]' --output text) \
    --query 'tasks[*].attachments[*].details[?name==`privateIPv4Address`].value' \
    --output text

# Find stopped tasks with reason
aws ecs list-tasks --cluster production --desired-status STOPPED
aws ecs describe-tasks \
    --cluster production \
    --tasks TASK_ARN \
    --query 'tasks[*].{id:taskArn,reason:stoppedReason,code:stopCode}'

# Get container instance ARNs (EC2 launch type)
aws ecs list-container-instances --cluster ec2-cluster
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Fargate** | Use for reduced ops overhead unless GPU/specific instance needed |
| **Fargate Spot** | Use for fault-tolerant workloads (up to 70% savings) |
| **Secrets** | Use Secrets Manager or SSM Parameter Store, never env vars |
| **Logging** | Always configure awslogs driver to CloudWatch |
| **Health checks** | Define container health checks for ALB integration |
| **ECS Exec** | Enable for debugging, disable in highly secure environments |
| **Task roles** | Separate execution role (ECR/logs) from task role (app perms) |
| **Circuit breaker** | Enable deployment circuit breaker for automatic rollback |
