# Advanced Patterns

## JMESPath Querying

JMESPath is a query language for JSON. Master these patterns to filter and transform AWS CLI output efficiently.

### Query Patterns by Complexity

```bash
# SELECTION: Extract fields from results
--query 'Reservations[*].Instances[*].[InstanceId,State.Name]'           # Array output
--query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name}'  # Named objects
--query 'Reservations[0].Instances[0].InstanceId'                        # Single value

# FILTERING: Match conditions
--query 'Reservations[*].Instances[?State.Name==`running`].InstanceId'   # Exact match
--query 'Contents[?Size > `1048576`].Key'                                # Comparison
--query 'Roles[?starts_with(RoleName, `Lambda`)].RoleName'               # String functions
--query 'Instances[?contains(Tags[?Key==`Name`].Value|[0], `prod`)]'     # Nested + contains

# SORTING & AGGREGATION: Transform results
--query 'sort_by(Images, &CreationDate)[-1].ImageId'                     # Sort, get last
--query 'reverse(sort_by(Images, &CreationDate))[0:5]'                   # Newest 5
--query 'length(Reservations[*].Instances[*][])'                         # Count
--query 'sum(Volumes[*].Size)'                                           # Sum values
--query 'max_by(Instances[], &LaunchTime).InstanceId'                    # Max by field

# TRANSFORMATION: Reshape output
--query 'Reservations[*].Instances[*].InstanceId[]'                      # Flatten arrays
--query 'Reservations[*].Instances[*].InstanceId | join(`,`, @)'         # Join to string
--query '{ID:InstanceId,HasIP:PublicIpAddress!=null}'                    # Conditional
--query '{Name:FunctionName,Timeout:Timeout||`3`}'                       # Default values
--query '{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0]}'              # Extract tag
```

### Practical Examples
```bash
# List running instances with names
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?State.Name==`running`].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0]}' \
    --output table

# Chain filters: running t3.micro instances
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?State.Name==`running`] | [?InstanceType==`t3.micro`] | [].{ID:InstanceId,AZ:Placement.AvailabilityZone}'
```

## Pagination & Limiting

### Client-side Pagination
```bash
# Control page size (memory management)
aws s3api list-objects-v2 --bucket my-bucket --page-size 100

# Limit total items
aws s3api list-objects-v2 --bucket my-bucket --max-items 50

# Starting token for manual pagination
aws s3api list-objects-v2 --bucket my-bucket \
    --max-items 100 \
    --starting-token eyJNYXJrZXIiOiBudWxsLCAiYm90b190cnVuY2F0ZV9hbW91bnQiOiAxMDB9

# No pager (direct stdout)
aws s3 ls --no-cli-pager

# Disable pager globally
export AWS_PAGER=""
```

### Server-side Pagination
```bash
# Use service-specific pagination
aws dynamodb scan \
    --table-name MyTable \
    --limit 25 \
    --exclusive-start-key '{"pk": {"S": "last-key"}}'

# List all with loop
aws s3api list-objects-v2 --bucket my-bucket --output json | \
    jq -r '.Contents[].Key' > all-keys.txt
```

## Skeletons & Input JSON

### Generate Skeletons
```bash
# Generate input skeleton
aws ec2 run-instances --generate-cli-skeleton > run-instance-input.json

# Generate output skeleton (shows expected response)
aws ec2 run-instances --generate-cli-skeleton output > run-instance-output.json

# Generate for complex commands
aws ecs create-service --generate-cli-skeleton > ecs-service.json
aws lambda create-function --generate-cli-skeleton > lambda-function.json
```

### Use Input JSON
```bash
# Execute with input file
aws ec2 run-instances --cli-input-json file://run-instance-input.json

# Combine with overrides
aws lambda create-function \
    --cli-input-json file://lambda-base.json \
    --function-name override-name

# YAML input (requires yq or conversion)
yq -o=json input.yaml | aws ecs create-service --cli-input-json file:///dev/stdin
```

## Waiters

### Built-in Waiters
```bash
# Wait for instance running
aws ec2 start-instances --instance-ids i-123
aws ec2 wait instance-running --instance-ids i-123
echo "Instance is running"

# Wait for instance stopped
aws ec2 wait instance-stopped --instance-ids i-123

# Wait for instance terminated
aws ec2 wait instance-terminated --instance-ids i-123

# Wait for CloudFormation stack
aws cloudformation wait stack-create-complete --stack-name my-stack
aws cloudformation wait stack-update-complete --stack-name my-stack
aws cloudformation wait stack-delete-complete --stack-name my-stack

# Wait for RDS available
aws rds wait db-instance-available --db-instance-identifier mydb

# Wait for ECS service stable
aws ecs wait services-stable \
    --cluster my-cluster \
    --services my-service

# Wait for Lambda function active
aws lambda wait function-active --function-name my-function

# Wait for S3 bucket exists
aws s3api wait bucket-exists --bucket my-bucket

# Wait for image available
aws ec2 wait image-available --image-ids ami-123
```

### Custom Waiter with Loop
```bash
#!/bin/bash
# Custom waiter for any condition

wait_for_condition() {
    local max_attempts=60
    local attempt=0
    local sleep_time=5

    while [ $attempt -lt $max_attempts ]; do
        status=$(aws ecs describe-services \
            --cluster my-cluster \
            --services my-service \
            --query 'services[0].deployments[0].rolloutState' \
            --output text)

        if [ "$status" = "COMPLETED" ]; then
            echo "Deployment completed"
            return 0
        elif [ "$status" = "FAILED" ]; then
            echo "Deployment failed"
            return 1
        fi

        echo "Status: $status (attempt $((attempt+1))/$max_attempts)"
        sleep $sleep_time
        ((attempt++))
    done

    echo "Timeout waiting for deployment"
    return 1
}

wait_for_condition
```

## CLI Aliases

### Configure Aliases
Create `~/.aws/cli/alias`:

```ini
[toplevel]
# Identity shortcuts
whoami = sts get-caller-identity
account = sts get-caller-identity --query Account --output text
region = configure get region

# S3 shortcuts
mkbucket = s3 mb
rmbucket = s3 rb --force

# Quick list commands
instances = ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,Tags[?Key==`Name`].Value|[0]]' --output table
volumes = ec2 describe-volumes --query 'Volumes[*].[VolumeId,State,Size,VolumeType]' --output table
buckets = s3api list-buckets --query 'Buckets[*].[Name,CreationDate]' --output table
functions = lambda list-functions --query 'Functions[*].[FunctionName,Runtime,MemorySize]' --output table

# Running instances
running = ec2 describe-instances --filters Name=instance-state-name,Values=running --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]' --output table

# Recent logs
logs = logs tail --follow

[command ec2]
# EC2-specific aliases
top = describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' --output table
myami = describe-images --owners self --query 'sort_by(Images,&CreationDate)[-5:].{ID:ImageId,Name:Name,Date:CreationDate}' --output table

[command s3]
# S3-specific aliases
disk = api list-objects-v2 --summarize --human-readable --query '{Objects:length(Contents),Size:Size}'

[command ecs]
# ECS shortcuts
tasks = list-tasks
services = list-services
```

**Usage:**
```bash
aws whoami
aws running
aws ec2 top
aws s3 disk --bucket my-bucket
```

## Debugging

### Dry Run
```bash
# Check permissions without executing
aws ec2 run-instances --dry-run \
    --image-id ami-123 \
    --instance-type t3.micro

# Returns error if no permission, success if allowed
aws ec2 terminate-instances --dry-run --instance-ids i-123
```

### Debug Mode
```bash
# Full debug output (HTTP requests/responses)
aws s3 ls --debug

# Debug to file
aws s3 ls --debug 2>&1 | tee debug.log

# Show just the HTTP traffic
aws s3 ls --debug 2>&1 | grep -E "(HTTP|Request|Response)"
```

### Verbose Credential Info
```bash
# Show credential source
aws sts get-caller-identity --debug 2>&1 | grep -i credential

# Check credential chain
aws configure list
```

## Scripting Patterns

### Error Handling
```bash
#!/bin/bash
set -euo pipefail

# Capture output and status
if output=$(aws s3 ls s3://my-bucket 2>&1); then
    echo "Success: $output"
else
    echo "Failed: $output"
    exit 1
fi

# Check specific error
create_output=$(aws s3 mb s3://my-bucket 2>&1) || {
    if [[ "$create_output" == *"BucketAlreadyOwnedByYou"* ]]; then
        echo "Bucket already exists, continuing..."
    else
        echo "Error: $create_output"
        exit 1
    fi
}

# Retry pattern
retry() {
    local max_attempts=3
    local attempt=1
    local delay=5

    while [ $attempt -le $max_attempts ]; do
        if "$@"; then
            return 0
        fi
        echo "Attempt $attempt failed, retrying in ${delay}s..."
        sleep $delay
        ((attempt++))
        ((delay*=2))
    done

    echo "All $max_attempts attempts failed"
    return 1
}

retry aws lambda invoke --function-name my-function output.json
```

### Parallel Execution
```bash
#!/bin/bash
# Process instances in parallel

instances=$(aws ec2 describe-instances \
    --filters Name=instance-state-name,Values=running \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text)

# Using xargs for parallel
echo "$instances" | xargs -n1 -P4 -I{} aws ssm send-command \
    --instance-ids {} \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["uptime"]'

# Using GNU parallel
echo "$instances" | parallel -j4 aws ssm send-command \
    --instance-ids {} \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["df -h"]'
```

### Batch Operations
```bash
#!/bin/bash
# Process in batches (API limits)

batch_size=25
items=($(aws sqs list-queues --query 'QueueUrls[*]' --output text))

for ((i=0; i<${#items[@]}; i+=batch_size)); do
    batch=("${items[@]:i:batch_size}")
    echo "Processing batch: ${batch[*]}"

    for queue in "${batch[@]}"; do
        aws sqs get-queue-attributes \
            --queue-url "$queue" \
            --attribute-names ApproximateNumberOfMessages
    done
done
```

### Output Processing
```bash
# CSV output
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name]' \
    --output text | tr '\t' ','

# JSON to CSV with jq
aws ec2 describe-instances | \
    jq -r '.Reservations[].Instances[] | [.InstanceId, .InstanceType, .State.Name] | @csv'

# Process JSON output
aws lambda list-functions | jq -r '.Functions[] |
    select(.Runtime | startswith("python")) |
    "\(.FunctionName): \(.Runtime) - \(.MemorySize)MB"'

# Create report
aws ec2 describe-instances --output json | jq -r '
    ["Instance ID","Name","Type","State"],
    (.Reservations[].Instances[] | [
        .InstanceId,
        (.Tags // [] | map(select(.Key == "Name")) | .[0].Value // "N/A"),
        .InstanceType,
        .State.Name
    ]) | @tsv
' | column -t -s $'\t'
```

### Configuration Management
```bash
#!/bin/bash
# Environment-specific config

ENV=${1:-dev}
CONFIG_FILE="config-${ENV}.json"

if [[ ! -f "$CONFIG_FILE" ]]; then
    echo "Config file not found: $CONFIG_FILE"
    exit 1
fi

# Read config
BUCKET=$(jq -r '.bucket' "$CONFIG_FILE")
REGION=$(jq -r '.region' "$CONFIG_FILE")
PROFILE=$(jq -r '.profile // "default"' "$CONFIG_FILE")

# Use in commands
aws s3 sync ./dist "s3://${BUCKET}/" \
    --region "$REGION" \
    --profile "$PROFILE"
```

## Useful Queries Collection

```bash
# Find untagged resources
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?!not_null(Tags)].InstanceId'

# Get total EBS storage
aws ec2 describe-volumes \
    --query 'sum(Volumes[*].Size)' \
    --output text

# Find public S3 buckets
aws s3api list-buckets --query 'Buckets[*].Name' --output text | \
    xargs -n1 -P4 -I{} sh -c \
    'aws s3api get-bucket-acl --bucket {} 2>/dev/null | grep -q AllUsers && echo {}'

# Lambda functions by runtime
aws lambda list-functions \
    --query 'Functions | group_by(@, &Runtime) | [*].{Runtime: [0].Runtime, Count: length(@)}'

# Cost estimation (running instances)
aws ec2 describe-instances \
    --filters Name=instance-state-name,Values=running \
    --query 'Reservations[*].Instances[*].InstanceType' \
    --output text | sort | uniq -c | sort -rn
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Use --query** | Filter on client side to reduce data transfer |
| **Use --output text** | For scripting (avoids JSON parsing) |
| **Use waiters** | Instead of sleep loops for async operations |
| **Use --dry-run** | Test permissions before executing |
| **Use aliases** | Create shortcuts for common operations |
| **Enable debug** | Use --debug for troubleshooting |
| **Handle pagination** | Account for paginated responses |
| **Retry on throttling** | Implement exponential backoff |
| **Use input files** | For complex parameters (--cli-input-json) |
| **Parallel processing** | Use xargs/parallel for batch operations |
