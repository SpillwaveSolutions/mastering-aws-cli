# Amazon Kinesis

## Data Streams

### Create Stream
```bash
# Provisioned mode (fixed shards)
aws kinesis create-stream \
    --stream-name app-clickstream \
    --shard-count 4

# On-demand mode (auto-scaling)
aws kinesis create-stream \
    --stream-name app-events \
    --stream-mode-details StreamMode=ON_DEMAND

# Wait for stream to be active
aws kinesis wait stream-exists --stream-name app-clickstream
```

### Manage Streams
```bash
# List streams
aws kinesis list-streams

# Describe stream
aws kinesis describe-stream --stream-name app-clickstream

# Describe stream summary (faster)
aws kinesis describe-stream-summary --stream-name app-clickstream

# Update shard count (provisioned mode)
aws kinesis update-shard-count \
    --stream-name app-clickstream \
    --target-shard-count 8 \
    --scaling-type UNIFORM_SCALING

# Update stream mode
aws kinesis update-stream-mode \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --stream-mode-details StreamMode=ON_DEMAND

# Delete stream
aws kinesis delete-stream --stream-name app-clickstream

# Enable encryption
aws kinesis start-stream-encryption \
    --stream-name app-clickstream \
    --encryption-type KMS \
    --key-id alias/aws/kinesis
```

## Producing Data

### Put Single Record
```bash
# Put record with data
aws kinesis put-record \
    --stream-name app-clickstream \
    --partition-key "user-123" \
    --data "$(echo -n '{"event":"click","page":"/home"}' | base64)"

# Put record from file
aws kinesis put-record \
    --stream-name app-clickstream \
    --partition-key "user-456" \
    --data fileb://event.json

# With explicit hash key (direct shard targeting)
aws kinesis put-record \
    --stream-name app-clickstream \
    --partition-key "user-123" \
    --explicit-hash-key "170141183460469231731687303715884105727" \
    --data "..."
```

### Put Multiple Records (Batch)
```bash
# Put records batch
aws kinesis put-records \
    --stream-name app-clickstream \
    --records '[
        {"Data": "eyJldmVudCI6ImNsaWNrIn0=", "PartitionKey": "user-1"},
        {"Data": "eyJldmVudCI6InZpZXcifQ==", "PartitionKey": "user-2"},
        {"Data": "eyJldmVudCI6InB1cmNoYXNlIn0=", "PartitionKey": "user-3"}
    ]'

# From file
aws kinesis put-records \
    --stream-name app-clickstream \
    --records file://records.json
```

**records.json:**
```json
[
    {"Data": "base64data1", "PartitionKey": "key1"},
    {"Data": "base64data2", "PartitionKey": "key2"}
]
```

## Consuming Data

### Get Shard Iterator
```bash
# Latest records (new data only)
ITERATOR=$(aws kinesis get-shard-iterator \
    --stream-name app-clickstream \
    --shard-id shardId-000000000000 \
    --shard-iterator-type LATEST \
    --query 'ShardIterator' \
    --output text)

# From beginning (all data)
ITERATOR=$(aws kinesis get-shard-iterator \
    --stream-name app-clickstream \
    --shard-id shardId-000000000000 \
    --shard-iterator-type TRIM_HORIZON \
    --query 'ShardIterator' \
    --output text)

# From specific sequence number
ITERATOR=$(aws kinesis get-shard-iterator \
    --stream-name app-clickstream \
    --shard-id shardId-000000000000 \
    --shard-iterator-type AT_SEQUENCE_NUMBER \
    --starting-sequence-number "12345678901234567890" \
    --query 'ShardIterator' \
    --output text)

# From timestamp
ITERATOR=$(aws kinesis get-shard-iterator \
    --stream-name app-clickstream \
    --shard-id shardId-000000000000 \
    --shard-iterator-type AT_TIMESTAMP \
    --timestamp "2024-01-15T12:00:00Z" \
    --query 'ShardIterator' \
    --output text)
```

### Get Records
```bash
# Get records
aws kinesis get-records --shard-iterator $ITERATOR

# Get records with limit
aws kinesis get-records \
    --shard-iterator $ITERATOR \
    --limit 100

# Decode records (using jq)
aws kinesis get-records --shard-iterator $ITERATOR | \
    jq -r '.Records[].Data' | \
    while read data; do echo $data | base64 -d; echo; done
```

### List Shards
```bash
# List all shards
aws kinesis list-shards --stream-name app-clickstream

# List with filter
aws kinesis list-shards \
    --stream-name app-clickstream \
    --shard-filter Type=AT_LATEST
```

## Enhanced Fan-Out

### Register Consumer
```bash
# Register enhanced fan-out consumer
aws kinesis register-stream-consumer \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --consumer-name my-app-consumer

# Wait for consumer to be active
aws kinesis describe-stream-consumer \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --consumer-name my-app-consumer
```

### Manage Consumers
```bash
# List consumers
aws kinesis list-stream-consumers \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream

# Describe consumer
aws kinesis describe-stream-consumer \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --consumer-name my-app-consumer

# Deregister consumer
aws kinesis deregister-stream-consumer \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --consumer-name my-app-consumer
```

### Subscribe to Shard (Enhanced Fan-Out)
```bash
# Subscribe (returns subscription ARN for SubscribeToShard API)
aws kinesis subscribe-to-shard \
    --consumer-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream/consumer/my-app-consumer \
    --shard-id shardId-000000000000 \
    --starting-position Type=LATEST
```

## Kinesis Data Firehose

### Create Delivery Stream
```bash
# Direct to S3
aws firehose create-delivery-stream \
    --delivery-stream-name raw-events-to-s3 \
    --delivery-stream-type DirectPut \
    --s3-destination-configuration '{
        "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole",
        "BucketARN": "arn:aws:s3:::my-data-bucket",
        "Prefix": "raw/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
        "ErrorOutputPrefix": "errors/",
        "BufferingHints": {
            "SizeInMBs": 128,
            "IntervalInSeconds": 300
        },
        "CompressionFormat": "GZIP"
    }'

# From Kinesis Data Stream to S3
aws firehose create-delivery-stream \
    --delivery-stream-name stream-to-s3 \
    --delivery-stream-type KinesisStreamAsSource \
    --kinesis-stream-source-configuration '{
        "KinesisStreamARN": "arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream",
        "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole"
    }' \
    --s3-destination-configuration '{
        "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole",
        "BucketARN": "arn:aws:s3:::my-data-bucket",
        "Prefix": "processed/"
    }'

# To OpenSearch
aws firehose create-delivery-stream \
    --delivery-stream-name logs-to-opensearch \
    --delivery-stream-type DirectPut \
    --elasticsearch-destination-configuration '{
        "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole",
        "DomainARN": "arn:aws:es:us-east-1:123456789012:domain/my-domain",
        "IndexName": "logs",
        "TypeName": "_doc",
        "IndexRotationPeriod": "OneDay",
        "BufferingHints": {
            "IntervalInSeconds": 60,
            "SizeInMBs": 5
        },
        "S3BackupMode": "FailedDocumentsOnly",
        "S3Configuration": {
            "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole",
            "BucketARN": "arn:aws:s3:::my-backup-bucket"
        }
    }'
```

### Manage Firehose
```bash
# List delivery streams
aws firehose list-delivery-streams

# Describe delivery stream
aws firehose describe-delivery-stream --delivery-stream-name raw-events-to-s3

# Put record to Firehose
aws firehose put-record \
    --delivery-stream-name raw-events-to-s3 \
    --record '{"Data": "eyJldmVudCI6InRlc3QifQ=="}'

# Put batch
aws firehose put-record-batch \
    --delivery-stream-name raw-events-to-s3 \
    --records '[{"Data": "..."}, {"Data": "..."}]'

# Update destination
aws firehose update-destination \
    --delivery-stream-name raw-events-to-s3 \
    --current-delivery-stream-version-id 1 \
    --destination-id destinationId-000000000001 \
    --s3-destination-update '{
        "BufferingHints": {"SizeInMBs": 256}
    }'

# Delete delivery stream
aws firehose delete-delivery-stream --delivery-stream-name raw-events-to-s3
```

## Lambda Integration

### Create Event Source Mapping
```bash
# Kinesis to Lambda
aws lambda create-event-source-mapping \
    --function-name process-stream \
    --event-source-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --starting-position LATEST \
    --batch-size 100 \
    --maximum-batching-window-in-seconds 5 \
    --parallelization-factor 2 \
    --maximum-retry-attempts 3 \
    --bisect-batch-on-function-error \
    --destination-config '{
        "OnFailure": {
            "Destination": "arn:aws:sqs:us-east-1:123456789012:dlq"
        }
    }'

# With tumbling window
aws lambda create-event-source-mapping \
    --function-name aggregate-stream \
    --event-source-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --starting-position LATEST \
    --batch-size 1000 \
    --tumbling-window-in-seconds 60

# With event filtering
aws lambda create-event-source-mapping \
    --function-name process-orders \
    --event-source-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-events \
    --starting-position LATEST \
    --filter-criteria '{
        "Filters": [{
            "Pattern": "{\"data\": {\"eventType\": [\"ORDER_PLACED\"]}}"
        }]
    }'
```

### Manage Event Source Mapping
```bash
# List mappings
aws lambda list-event-source-mappings \
    --function-name process-stream

# Update mapping
aws lambda update-event-source-mapping \
    --uuid abc-123-uuid \
    --batch-size 200 \
    --parallelization-factor 4

# Disable mapping
aws lambda update-event-source-mapping \
    --uuid abc-123-uuid \
    --no-enabled
```

## Kinesis Data Analytics

### Create Application
```bash
# Create SQL application
aws kinesisanalytics create-application \
    --application-name real-time-analytics \
    --application-description "Real-time stream processing" \
    --runtime-environment SQL-1_0 \
    --service-execution-role arn:aws:iam::123456789012:role/KinesisAnalyticsRole

# Create Flink application
aws kinesisanalyticsv2 create-application \
    --application-name flink-processor \
    --runtime-environment FLINK-1_15 \
    --service-execution-role arn:aws:iam::123456789012:role/KinesisAnalyticsRole \
    --application-configuration '{
        "FlinkApplicationConfiguration": {
            "ParallelismConfiguration": {
                "ConfigurationType": "CUSTOM",
                "Parallelism": 4,
                "ParallelismPerKPU": 1,
                "AutoScalingEnabled": true
            }
        },
        "ApplicationCodeConfiguration": {
            "CodeContent": {
                "S3ContentLocation": {
                    "BucketARN": "arn:aws:s3:::my-apps-bucket",
                    "FileKey": "flink-app.jar"
                }
            },
            "CodeContentType": "ZIPFILE"
        }
    }'
```

## Useful Queries

```bash
# Get stream status
aws kinesis describe-stream-summary \
    --stream-name app-clickstream \
    --query 'StreamDescriptionSummary.StreamStatus'

# Get shard count
aws kinesis describe-stream-summary \
    --stream-name app-clickstream \
    --query 'StreamDescriptionSummary.OpenShardCount'

# Get consumer count
aws kinesis list-stream-consumers \
    --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/app-clickstream \
    --query 'Consumers | length(@)'

# Get Firehose status
aws firehose describe-delivery-stream \
    --delivery-stream-name raw-events-to-s3 \
    --query 'DeliveryStreamDescription.DeliveryStreamStatus'
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **On-demand mode** | Use for variable or unpredictable traffic |
| **Partition keys** | Randomize to avoid hot shards |
| **Enhanced fan-out** | Use for multiple low-latency consumers |
| **Batching** | Use PutRecords for higher throughput |
| **Firehose** | Use for simple S3/OpenSearch delivery |
| **Lambda parallelization** | Increase for high-volume streams |
| **Filtering** | Use event filters to reduce Lambda invocations |
| **DLQ** | Configure for failed record handling |
| **Encryption** | Enable KMS encryption for sensitive data |
| **Retention** | Extend beyond 24h for replay capability |
