# DynamoDB

## Table Management

### Create Table (On-Demand)
```bash
# Simple table with on-demand billing
aws dynamodb create-table \
    --table-name Users \
    --attribute-definitions \
        AttributeName=UserId,AttributeType=S \
    --key-schema \
        AttributeName=UserId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST

# With Global Secondary Index
aws dynamodb create-table \
    --table-name Users \
    --attribute-definitions \
        AttributeName=UserId,AttributeType=S \
        AttributeName=Email,AttributeType=S \
    --key-schema \
        AttributeName=UserId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --global-secondary-indexes '[
        {
            "IndexName": "EmailIndex",
            "KeySchema": [{"AttributeName": "Email", "KeyType": "HASH"}],
            "Projection": {"ProjectionType": "ALL"}
        }
    ]'

# Composite key (partition + sort)
aws dynamodb create-table \
    --table-name Orders \
    --attribute-definitions \
        AttributeName=CustomerId,AttributeType=S \
        AttributeName=OrderDate,AttributeType=S \
    --key-schema \
        AttributeName=CustomerId,KeyType=HASH \
        AttributeName=OrderDate,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST
```

### Create Table (Provisioned)
```bash
aws dynamodb create-table \
    --table-name Logs \
    --attribute-definitions AttributeName=LogId,AttributeType=S \
    --key-schema AttributeName=LogId,KeyType=HASH \
    --billing-mode PROVISIONED \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

### Manage Tables
```bash
# List tables
aws dynamodb list-tables

# Describe table
aws dynamodb describe-table --table-name Users

# Delete table
aws dynamodb delete-table --table-name Users

# Wait for table to be active
aws dynamodb wait table-exists --table-name Users
```

### Update Table
```bash
# Switch to provisioned capacity
aws dynamodb update-table \
    --table-name Users \
    --billing-mode PROVISIONED \
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10

# Switch to on-demand
aws dynamodb update-table \
    --table-name Users \
    --billing-mode PAY_PER_REQUEST

# Add GSI
aws dynamodb update-table \
    --table-name Users \
    --attribute-definitions AttributeName=Status,AttributeType=S \
    --global-secondary-index-updates '[
        {
            "Create": {
                "IndexName": "StatusIndex",
                "KeySchema": [{"AttributeName": "Status", "KeyType": "HASH"}],
                "Projection": {"ProjectionType": "ALL"}
            }
        }
    ]'

# Delete GSI
aws dynamodb update-table \
    --table-name Users \
    --global-secondary-index-updates '[
        {"Delete": {"IndexName": "StatusIndex"}}
    ]'
```

## Item Operations

### Put Item
```bash
# Basic put
aws dynamodb put-item \
    --table-name Users \
    --item '{
        "UserId": {"S": "u-123"},
        "Name": {"S": "Alice"},
        "Age": {"N": "30"},
        "Email": {"S": "alice@example.com"},
        "Tags": {"SS": ["admin", "active"]}
    }'

# Put with condition (only if not exists)
aws dynamodb put-item \
    --table-name Users \
    --item '{"UserId": {"S": "u-123"}, "Name": {"S": "Alice"}}' \
    --condition-expression "attribute_not_exists(UserId)"

# Return old values
aws dynamodb put-item \
    --table-name Users \
    --item '{"UserId": {"S": "u-123"}, "Name": {"S": "Bob"}}' \
    --return-values ALL_OLD
```

### Get Item
```bash
# Basic get
aws dynamodb get-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}'

# Get specific attributes only
aws dynamodb get-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --projection-expression "Name, Email"

# Consistent read
aws dynamodb get-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --consistent-read
```

### Update Item
```bash
# Update with expression
aws dynamodb update-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --update-expression "SET Age = :age, #status = :status" \
    --expression-attribute-names '{"#status": "Status"}' \
    --expression-attribute-values '{":age": {"N": "31"}, ":status": {"S": "active"}}'

# Increment counter
aws dynamodb update-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --update-expression "SET LoginCount = if_not_exists(LoginCount, :zero) + :inc" \
    --expression-attribute-values '{":zero": {"N": "0"}, ":inc": {"N": "1"}}'

# Add to set
aws dynamodb update-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --update-expression "ADD Tags :newTags" \
    --expression-attribute-values '{":newTags": {"SS": ["premium"]}}'

# Remove attribute
aws dynamodb update-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --update-expression "REMOVE TemporaryField"

# Conditional update
aws dynamodb update-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --update-expression "SET Balance = Balance - :amount" \
    --condition-expression "Balance >= :amount" \
    --expression-attribute-values '{":amount": {"N": "100"}}'
```

### Delete Item
```bash
# Basic delete
aws dynamodb delete-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}'

# Conditional delete
aws dynamodb delete-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --condition-expression "#status = :inactive" \
    --expression-attribute-names '{"#status": "Status"}' \
    --expression-attribute-values '{":inactive": {"S": "inactive"}}'

# Return deleted item
aws dynamodb delete-item \
    --table-name Users \
    --key '{"UserId": {"S": "u-123"}}' \
    --return-values ALL_OLD
```

## Query and Scan

### Query (Efficient)
```bash
# Query by partition key
aws dynamodb query \
    --table-name Orders \
    --key-condition-expression "CustomerId = :cid" \
    --expression-attribute-values '{":cid": {"S": "c-123"}}'

# Query with sort key condition
aws dynamodb query \
    --table-name Orders \
    --key-condition-expression "CustomerId = :cid AND OrderDate BETWEEN :start AND :end" \
    --expression-attribute-values '{
        ":cid": {"S": "c-123"},
        ":start": {"S": "2024-01-01"},
        ":end": {"S": "2024-12-31"}
    }'

# Query with filter
aws dynamodb query \
    --table-name Orders \
    --key-condition-expression "CustomerId = :cid" \
    --filter-expression "Amount > :min" \
    --expression-attribute-values '{":cid": {"S": "c-123"}, ":min": {"N": "100"}}'

# Query GSI
aws dynamodb query \
    --table-name Users \
    --index-name EmailIndex \
    --key-condition-expression "Email = :email" \
    --expression-attribute-values '{":email": {"S": "alice@example.com"}}'

# Reverse order (descending)
aws dynamodb query \
    --table-name Orders \
    --key-condition-expression "CustomerId = :cid" \
    --expression-attribute-values '{":cid": {"S": "c-123"}}' \
    --scan-index-forward false \
    --limit 10
```

### Scan (Use Sparingly)
```bash
# Basic scan
aws dynamodb scan --table-name Users

# Scan with filter
aws dynamodb scan \
    --table-name Users \
    --filter-expression "Age > :age" \
    --expression-attribute-values '{":age": {"N": "25"}}'

# Parallel scan (for large tables)
aws dynamodb scan \
    --table-name Users \
    --segment 0 \
    --total-segments 4
```

### PartiQL
```bash
# Select
aws dynamodb execute-statement \
    --statement "SELECT * FROM Users WHERE UserId = 'u-123'"

# Select with filter
aws dynamodb execute-statement \
    --statement "SELECT Name, Email FROM Users WHERE Age > 25"

# Insert
aws dynamodb execute-statement \
    --statement "INSERT INTO Users VALUE {'UserId': 'u-456', 'Name': 'Bob'}"

# Update
aws dynamodb execute-statement \
    --statement "UPDATE Users SET Age = 32 WHERE UserId = 'u-123'"

# Delete
aws dynamodb execute-statement \
    --statement "DELETE FROM Users WHERE UserId = 'u-123'"
```

## Batch Operations

### Batch Write
```bash
# Write multiple items (max 25)
aws dynamodb batch-write-item \
    --request-items '{
        "Users": [
            {"PutRequest": {"Item": {"UserId": {"S": "u-1"}, "Name": {"S": "Alice"}}}},
            {"PutRequest": {"Item": {"UserId": {"S": "u-2"}, "Name": {"S": "Bob"}}}},
            {"DeleteRequest": {"Key": {"UserId": {"S": "u-old"}}}}
        ]
    }'
```

### Batch Get
```bash
# Get multiple items (max 100)
aws dynamodb batch-get-item \
    --request-items '{
        "Users": {
            "Keys": [
                {"UserId": {"S": "u-1"}},
                {"UserId": {"S": "u-2"}},
                {"UserId": {"S": "u-3"}}
            ],
            "ProjectionExpression": "UserId, Name, Email"
        }
    }'
```

## Transactions

### TransactWrite (All-or-nothing writes)
```bash
aws dynamodb transact-write-items \
    --transact-items '[
        {
            "Put": {
                "TableName": "Orders",
                "Item": {"OrderId": {"S": "o-123"}, "CustomerId": {"S": "c-123"}, "Amount": {"N": "100"}}
            }
        },
        {
            "Update": {
                "TableName": "Customers",
                "Key": {"CustomerId": {"S": "c-123"}},
                "UpdateExpression": "SET Balance = Balance - :amount",
                "ConditionExpression": "Balance >= :amount",
                "ExpressionAttributeValues": {":amount": {"N": "100"}}
            }
        }
    ]'
```

### TransactGet (Consistent reads)
```bash
aws dynamodb transact-get-items \
    --transact-items '[
        {"Get": {"TableName": "Users", "Key": {"UserId": {"S": "u-123"}}}},
        {"Get": {"TableName": "Orders", "Key": {"OrderId": {"S": "o-123"}}}}
    ]'
```

## DynamoDB Streams

### Enable Streams
```bash
# Enable streams on table
aws dynamodb update-table \
    --table-name Users \
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# Stream view types:
# KEYS_ONLY - Only key attributes
# NEW_IMAGE - Item after modification
# OLD_IMAGE - Item before modification
# NEW_AND_OLD_IMAGES - Both before and after
```

### Read Streams
```bash
# Get stream ARN
aws dynamodb describe-table \
    --table-name Users \
    --query 'Table.LatestStreamArn' \
    --output text

# List stream shards
aws dynamodbstreams describe-stream \
    --stream-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/stream/2024-01-01T00:00:00.000

# Get shard iterator
aws dynamodbstreams get-shard-iterator \
    --stream-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/stream/2024-01-01T00:00:00.000 \
    --shard-id shardId-00000001 \
    --shard-iterator-type TRIM_HORIZON

# Get records
aws dynamodbstreams get-records \
    --shard-iterator <shard-iterator>
```

## Global Tables

### Create Global Table
```bash
# Create table in first region
aws dynamodb create-table \
    --table-name GlobalUsers \
    --attribute-definitions AttributeName=UserId,AttributeType=S \
    --key-schema AttributeName=UserId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
    --region us-east-1

# Wait for table
aws dynamodb wait table-exists --table-name GlobalUsers --region us-east-1

# Add replica in another region
aws dynamodb update-table \
    --table-name GlobalUsers \
    --replica-updates '[{"Create": {"RegionName": "eu-west-1"}}]' \
    --region us-east-1

# Add more replicas
aws dynamodb update-table \
    --table-name GlobalUsers \
    --replica-updates '[{"Create": {"RegionName": "ap-southeast-1"}}]' \
    --region us-east-1
```

### Manage Global Tables
```bash
# Describe global table
aws dynamodb describe-table \
    --table-name GlobalUsers \
    --query 'Table.Replicas'

# Remove replica
aws dynamodb update-table \
    --table-name GlobalUsers \
    --replica-updates '[{"Delete": {"RegionName": "ap-southeast-1"}}]' \
    --region us-east-1
```

## Time to Live (TTL)

### Enable TTL
```bash
# Enable TTL on attribute
aws dynamodb update-time-to-live \
    --table-name Sessions \
    --time-to-live-specification Enabled=true,AttributeName=ExpiresAt

# Check TTL status
aws dynamodb describe-time-to-live --table-name Sessions
```

### Use TTL
```bash
# Put item with TTL (Unix timestamp)
EXPIRES=$(date -d '+24 hours' +%s)
aws dynamodb put-item \
    --table-name Sessions \
    --item "{
        \"SessionId\": {\"S\": \"sess-123\"},
        \"UserId\": {\"S\": \"u-123\"},
        \"ExpiresAt\": {\"N\": \"$EXPIRES\"}
    }"
```

## Backups and Recovery

### On-Demand Backup
```bash
# Create backup
aws dynamodb create-backup \
    --table-name Users \
    --backup-name Users-2024-01-15

# List backups
aws dynamodb list-backups --table-name Users

# Describe backup
aws dynamodb describe-backup \
    --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/backup/01234567890123-abc123

# Restore from backup
aws dynamodb restore-table-from-backup \
    --target-table-name Users-Restored \
    --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/backup/01234567890123-abc123

# Delete backup
aws dynamodb delete-backup \
    --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/backup/01234567890123-abc123
```

### Point-in-Time Recovery (PITR)
```bash
# Enable PITR
aws dynamodb update-continuous-backups \
    --table-name Users \
    --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Check PITR status
aws dynamodb describe-continuous-backups --table-name Users

# Restore to point in time
aws dynamodb restore-table-to-point-in-time \
    --source-table-name Users \
    --target-table-name Users-Restored \
    --restore-date-time 2024-01-15T12:00:00Z
```

## Export to S3

```bash
# Export table to S3 (for analytics)
aws dynamodb export-table-to-point-in-time \
    --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users \
    --s3-bucket my-exports-bucket \
    --s3-prefix dynamodb-exports/ \
    --export-format DYNAMODB_JSON

# List exports
aws dynamodb list-exports --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users

# Describe export
aws dynamodb describe-export \
    --export-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/export/01234567890123-abc123
```

## Import from S3

```bash
# Import from S3
aws dynamodb import-table \
    --s3-bucket-source S3Bucket=my-import-bucket,S3KeyPrefix=imports/ \
    --input-format DYNAMODB_JSON \
    --table-creation-parameters '{
        "TableName": "ImportedUsers",
        "AttributeDefinitions": [{"AttributeName": "UserId", "AttributeType": "S"}],
        "KeySchema": [{"AttributeName": "UserId", "KeyType": "HASH"}],
        "BillingMode": "PAY_PER_REQUEST"
    }'

# List imports
aws dynamodb list-imports

# Describe import
aws dynamodb describe-import --import-arn <import-arn>
```

## Auto Scaling (Provisioned Mode)

```bash
# Register scalable target (read capacity)
aws application-autoscaling register-scalable-target \
    --service-namespace dynamodb \
    --resource-id "table/Users" \
    --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
    --min-capacity 5 \
    --max-capacity 1000

# Create scaling policy
aws application-autoscaling put-scaling-policy \
    --service-namespace dynamodb \
    --resource-id "table/Users" \
    --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
    --policy-name "UsersReadScaling" \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
        },
        "ScaleOutCooldown": 60,
        "ScaleInCooldown": 60
    }'
```

## Useful Queries

```bash
# Get table item count
aws dynamodb describe-table \
    --table-name Users \
    --query 'Table.ItemCount'

# Get table size in bytes
aws dynamodb describe-table \
    --table-name Users \
    --query 'Table.TableSizeBytes'

# List all GSIs
aws dynamodb describe-table \
    --table-name Users \
    --query 'Table.GlobalSecondaryIndexes[*].IndexName'

# Check table status
aws dynamodb describe-table \
    --table-name Users \
    --query 'Table.TableStatus'

# Get stream ARN
aws dynamodb describe-table \
    --table-name Users \
    --query 'Table.LatestStreamArn' \
    --output text
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **On-demand billing** | Start with PAY_PER_REQUEST for new workloads |
| **Partition key design** | Use high-cardinality keys to avoid hot partitions |
| **GSI projection** | Only project needed attributes to reduce costs |
| **Conditional writes** | Use conditions to prevent race conditions |
| **TTL** | Enable for session data, caches, temporary records |
| **Streams** | Use for change data capture, event-driven architectures |
| **Global Tables** | Enable for multi-region active-active |
| **PITR** | Enable for critical tables (35-day recovery window) |
| **Batch operations** | Use batch-write-item for bulk inserts (max 25) |
| **Transactions** | Use for ACID operations across multiple items |
