# Lambda Functions

## Contents

- [Function Creation](#function-creation)
- [Code & Configuration Updates](#code--configuration-updates)
- [Layers](#layers)
- [Versions & Aliases](#versions--aliases)
  - [Safe Deployment Checklist](#safe-deployment-checklist-blue-green-with-aliases)
- [Invocation](#invocation)
- [Event Source Mappings](#event-source-mappings)
- [Function URLs](#function-urls)
- [Concurrency](#concurrency)
- [Permissions (Resource Policies)](#permissions-resource-policies)
- [Useful Queries](#useful-queries)
- [Best Practices](#best-practices)

---

## Function Creation

### From ZIP (Python/Node.js)
```bash
# Python with ARM64 architecture (Graviton2 - cheaper & often faster)
aws lambda create-function \
    --function-name my-processor \
    --runtime python3.12 \
    --architectures arm64 \
    --handler lambda_function.handler \
    --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
    --zip-file fileb://function.zip \
    --memory-size 512 \
    --timeout 30 \
    --environment Variables='{LOG_LEVEL=INFO,DB_HOST=prod-db.example.com}'

# Node.js with x86_64
aws lambda create-function \
    --function-name my-api \
    --runtime nodejs22.x \
    --architectures x86_64 \
    --handler index.handler \
    --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
    --zip-file fileb://function.zip
```

### From S3
```bash
aws lambda create-function \
    --function-name my-function \
    --runtime python3.12 \
    --code S3Bucket=my-deployment-bucket,S3Key=deployments/function.zip,S3ObjectVersion=abc123 \
    --handler app.lambda_handler \
    --role arn:aws:iam::123456789012:role/LambdaExecutionRole
```

### From Container Image (ECR)
```bash
aws lambda create-function \
    --function-name my-container-func \
    --package-type Image \
    --code ImageUri=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-image:latest \
    --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
    --architectures arm64 \
    --memory-size 1024 \
    --timeout 60
```

## Code & Configuration Updates

### Update Code
```bash
# From local ZIP
aws lambda update-function-code \
    --function-name my-function \
    --zip-file fileb://new-code.zip

# From S3
aws lambda update-function-code \
    --function-name my-function \
    --s3-bucket my-bucket \
    --s3-key deployments/new-code.zip

# From ECR image
aws lambda update-function-code \
    --function-name my-container-func \
    --image-uri 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-image:v2

# Publish version immediately
aws lambda update-function-code \
    --function-name my-function \
    --zip-file fileb://new-code.zip \
    --publish
```

### Update Configuration
```bash
# Memory and timeout
aws lambda update-function-configuration \
    --function-name my-function \
    --memory-size 1024 \
    --timeout 60

# Environment variables
aws lambda update-function-configuration \
    --function-name my-function \
    --environment Variables='{KEY1=value1,KEY2=value2}'

# Add layers
aws lambda update-function-configuration \
    --function-name my-function \
    --layers arn:aws:lambda:us-east-1:123456789012:layer:my-layer:3 \
             arn:aws:lambda:us-east-1:123456789012:layer:other-layer:1

# VPC configuration
aws lambda update-function-configuration \
    --function-name my-function \
    --vpc-config SubnetIds=subnet-1,subnet-2,SecurityGroupIds=sg-1

# Remove VPC
aws lambda update-function-configuration \
    --function-name my-function \
    --vpc-config SubnetIds=[],SecurityGroupIds=[]
```

### Wait for Update Completion
```bash
# Wait for function update to complete
aws lambda wait function-updated --function-name my-function

# Wait for function to be active
aws lambda wait function-active --function-name my-function
```

## Layers

### Create and Publish Layer
```bash
# Publish layer version
aws lambda publish-layer-version \
    --layer-name common-libs \
    --description "Shared Python libraries" \
    --zip-file fileb://layer.zip \
    --compatible-runtimes python3.11 python3.12 \
    --compatible-architectures arm64 x86_64

# Grant layer usage to another account
aws lambda add-layer-version-permission \
    --layer-name common-libs \
    --version-number 1 \
    --statement-id xaccount \
    --action lambda:GetLayerVersion \
    --principal 987654321098

# Grant to all accounts (public layer)
aws lambda add-layer-version-permission \
    --layer-name common-libs \
    --version-number 1 \
    --statement-id public \
    --action lambda:GetLayerVersion \
    --principal "*"
```

### Manage Layers
```bash
# List layers
aws lambda list-layers

# List layer versions
aws lambda list-layer-versions --layer-name common-libs

# Get layer version details
aws lambda get-layer-version \
    --layer-name common-libs \
    --version-number 1

# Delete layer version
aws lambda delete-layer-version \
    --layer-name common-libs \
    --version-number 1
```

## Versions & Aliases

### Publish Version
```bash
# Publish immutable version from $LATEST
aws lambda publish-version \
    --function-name my-function \
    --description "Release v1.2.0"

# List versions
aws lambda list-versions-by-function --function-name my-function
```

### Create and Manage Aliases
```bash
# Create alias pointing to version
aws lambda create-alias \
    --function-name my-function \
    --name PROD \
    --function-version 5 \
    --description "Production traffic"

# Update alias to new version
aws lambda update-alias \
    --function-name my-function \
    --name PROD \
    --function-version 6

# Weighted routing (canary deployment - 10% to new version)
aws lambda update-alias \
    --function-name my-function \
    --name PROD \
    --function-version 5 \
    --routing-config AdditionalVersionWeights='{"6": 0.1}'

# Complete traffic shift (100% to new version)
aws lambda update-alias \
    --function-name my-function \
    --name PROD \
    --function-version 6 \
    --routing-config AdditionalVersionWeights='{}'

# List aliases
aws lambda list-aliases --function-name my-function

# Delete alias
aws lambda delete-alias --function-name my-function --name STAGING
```

### Safe Deployment Checklist (Blue-Green with Aliases)

```
[ ] 1. Update code: aws lambda update-function-code --function-name my-function --zip-file fileb://code.zip
[ ] 2. Wait for update: aws lambda wait function-updated --function-name my-function
[ ] 3. Test $LATEST: aws lambda invoke --function-name my-function --qualifier '$LATEST' test-response.json
[ ] 4. Publish version: aws lambda publish-version --function-name my-function --description "v1.2.0"
[ ] 5. Canary (10%): aws lambda update-alias --function-name my-function --name PROD --routing-config AdditionalVersionWeights='{"NEW_VERSION": 0.1}'
[ ] 6. Monitor CloudWatch for errors (wait 5-10 min)
[ ] 7. Full shift: aws lambda update-alias --function-name my-function --name PROD --function-version NEW_VERSION --routing-config AdditionalVersionWeights='{}'
[ ] 8. Verify: aws lambda get-alias --function-name my-function --name PROD
```

## Invocation

### Synchronous Invocation
```bash
# Basic invocation
aws lambda invoke \
    --function-name my-function \
    response.json

# With payload
aws lambda invoke \
    --function-name my-function \
    --cli-binary-format raw-in-base64-out \
    --payload '{"key": "value", "items": [1, 2, 3]}' \
    response.json

# Invoke specific version or alias
aws lambda invoke \
    --function-name my-function \
    --qualifier PROD \
    response.json

# With logs in response
aws lambda invoke \
    --function-name my-function \
    --log-type Tail \
    --query 'LogResult' \
    --output text \
    response.json | base64 -d
```

### Asynchronous Invocation
```bash
# Fire-and-forget
aws lambda invoke \
    --function-name my-function \
    --invocation-type Event \
    --payload '{"event": "data"}' \
    response.json

# Dry run (validation only)
aws lambda invoke \
    --function-name my-function \
    --invocation-type DryRun \
    response.json
```

## Event Source Mappings

### SQS Trigger
```bash
aws lambda create-event-source-mapping \
    --function-name my-function \
    --event-source-arn arn:aws:sqs:us-east-1:123456789012:my-queue \
    --batch-size 10 \
    --maximum-batching-window-in-seconds 5
```

### DynamoDB Streams
```bash
aws lambda create-event-source-mapping \
    --function-name my-function \
    --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/MyTable/stream/2024-01-01T00:00:00.000 \
    --batch-size 100 \
    --starting-position LATEST \
    --maximum-retry-attempts 3 \
    --bisect-batch-on-function-error
```

### Kinesis Stream
```bash
aws lambda create-event-source-mapping \
    --function-name my-function \
    --event-source-arn arn:aws:kinesis:us-east-1:123456789012:stream/my-stream \
    --batch-size 100 \
    --starting-position LATEST \
    --parallelization-factor 2 \
    --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123456789012:dlq"}}'
```

### MSK (Kafka)
```bash
aws lambda create-event-source-mapping \
    --function-name my-function \
    --event-source-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/abc123 \
    --batch-size 100 \
    --starting-position LATEST \
    --topics my-topic
```

### Event Filtering
```bash
# Filter events before invoking Lambda
aws lambda create-event-source-mapping \
    --function-name my-function \
    --event-source-arn arn:aws:sqs:us-east-1:123456789012:my-queue \
    --filter-criteria '{"Filters": [{"Pattern": "{\"body\": {\"type\": [\"order\"]}}"}]}'

# Update existing mapping with filter
aws lambda update-event-source-mapping \
    --uuid abc123-uuid \
    --filter-criteria '{"Filters": [{"Pattern": "{\"eventName\": [\"INSERT\", \"MODIFY\"]}"}]}'
```

### Manage Event Source Mappings
```bash
# List all mappings
aws lambda list-event-source-mappings --function-name my-function

# Get mapping details
aws lambda get-event-source-mapping --uuid abc123-uuid

# Enable/disable mapping
aws lambda update-event-source-mapping \
    --uuid abc123-uuid \
    --enabled

aws lambda update-event-source-mapping \
    --uuid abc123-uuid \
    --no-enabled

# Delete mapping
aws lambda delete-event-source-mapping --uuid abc123-uuid
```

## Function URLs

### Create Function URL
```bash
# Public endpoint (no auth)
aws lambda create-function-url-config \
    --function-name my-function \
    --auth-type NONE

# IAM auth required
aws lambda create-function-url-config \
    --function-name my-function \
    --auth-type AWS_IAM

# With CORS
aws lambda create-function-url-config \
    --function-name my-function \
    --auth-type NONE \
    --cors '{
        "AllowOrigins": ["https://example.com"],
        "AllowMethods": ["GET", "POST"],
        "AllowHeaders": ["Content-Type"],
        "MaxAge": 86400
    }'
```

### Grant Public Access (for NONE auth type)
```bash
aws lambda add-permission \
    --function-name my-function \
    --statement-id FunctionURLAllowPublicAccess \
    --action lambda:InvokeFunctionUrl \
    --principal "*" \
    --function-url-auth-type NONE
```

### Manage Function URLs
```bash
# Get URL configuration
aws lambda get-function-url-config --function-name my-function

# Update CORS
aws lambda update-function-url-config \
    --function-name my-function \
    --cors '{"AllowOrigins": ["*"]}'

# Delete function URL
aws lambda delete-function-url-config --function-name my-function
```

## Concurrency

### Reserved Concurrency
```bash
# Set reserved concurrency (guarantees capacity, also limits max)
aws lambda put-function-concurrency \
    --function-name my-function \
    --reserved-concurrent-executions 100

# Get concurrency configuration
aws lambda get-function-concurrency --function-name my-function

# Remove reserved concurrency (use unreserved pool)
aws lambda delete-function-concurrency --function-name my-function
```

### Provisioned Concurrency
```bash
# Provision warm instances for alias/version (eliminates cold starts)
aws lambda put-provisioned-concurrency-config \
    --function-name my-function \
    --qualifier PROD \
    --provisioned-concurrent-executions 50

# Get provisioned concurrency status
aws lambda get-provisioned-concurrency-config \
    --function-name my-function \
    --qualifier PROD

# List all provisioned concurrency configs
aws lambda list-provisioned-concurrency-configs --function-name my-function

# Delete provisioned concurrency
aws lambda delete-provisioned-concurrency-config \
    --function-name my-function \
    --qualifier PROD
```

## Permissions (Resource Policies)

```bash
# Add permission for service to invoke
aws lambda add-permission \
    --function-name my-function \
    --statement-id AllowS3Invoke \
    --action lambda:InvokeFunction \
    --principal s3.amazonaws.com \
    --source-arn arn:aws:s3:::my-bucket \
    --source-account 123456789012

# Add permission for API Gateway
aws lambda add-permission \
    --function-name my-function \
    --statement-id AllowAPIGatewayInvoke \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:us-east-1:123456789012:api-id/*"

# Add permission for CloudWatch Events
aws lambda add-permission \
    --function-name my-function \
    --statement-id AllowCloudWatchEventsInvoke \
    --action lambda:InvokeFunction \
    --principal events.amazonaws.com \
    --source-arn arn:aws:events:us-east-1:123456789012:rule/my-rule

# Get resource policy
aws lambda get-policy --function-name my-function

# Remove permission
aws lambda remove-permission \
    --function-name my-function \
    --statement-id AllowS3Invoke
```

## Useful Queries

```bash
# List all functions with memory > 512MB
aws lambda list-functions \
    --query 'Functions[?MemorySize > `512`].[FunctionName, MemorySize, Runtime]' \
    --output table

# Find functions by runtime
aws lambda list-functions \
    --query 'Functions[?Runtime==`python3.12`].FunctionName'

# Get function ARN
aws lambda get-function \
    --function-name my-function \
    --query 'Configuration.FunctionArn' \
    --output text

# Find functions with specific tag
aws lambda list-functions \
    --query 'Functions[?Tags.Environment==`production`].FunctionName'
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **ARM64** | Use Graviton2 (`arm64`) for better price/performance |
| **Layers** | Share common code via layers to reduce package size |
| **Aliases** | Use aliases (PROD, STAGING) for safe deployments |
| **Reserved concurrency** | Set limits to prevent runaway costs |
| **Provisioned concurrency** | Use for latency-sensitive workloads |
| **Environment variables** | Externalize configuration, never hardcode |
| **IAM** | Use minimal permissions, function-specific roles |
| **VPC** | Only use VPC if accessing private resources |
| **Event filtering** | Filter at source to reduce invocations |
| **Dead-letter queues** | Configure DLQ for async invocations |
