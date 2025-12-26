# AWS Glue

## Data Catalog

### Databases
```bash
# Create database
aws glue create-database \
    --database-input '{
        "Name": "analytics",
        "Description": "Data Lake Catalog",
        "LocationUri": "s3://my-datalake/"
    }'

# List databases
aws glue get-databases

# Get database
aws glue get-database --name analytics

# Delete database
aws glue delete-database --name analytics
```

### Tables
```bash
# Create table
aws glue create-table \
    --database-name analytics \
    --table-input file://table-definition.json

# Get table
aws glue get-table --database-name analytics --name raw_logs

# List tables
aws glue get-tables --database-name analytics

# Update table
aws glue update-table \
    --database-name analytics \
    --table-input file://updated-table.json

# Delete table
aws glue delete-table --database-name analytics --name raw_logs
```

**table-definition.json:**
```json
{
    "Name": "raw_logs",
    "StorageDescriptor": {
        "Columns": [
            {"Name": "timestamp", "Type": "timestamp"},
            {"Name": "user_id", "Type": "string"},
            {"Name": "event", "Type": "string"},
            {"Name": "properties", "Type": "map<string,string>"}
        ],
        "Location": "s3://my-datalake/raw/logs/",
        "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
        "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "SerdeInfo": {
            "SerializationLibrary": "org.openx.data.jsonserde.JsonSerDe"
        }
    },
    "PartitionKeys": [
        {"Name": "year", "Type": "string"},
        {"Name": "month", "Type": "string"},
        {"Name": "day", "Type": "string"}
    ],
    "TableType": "EXTERNAL_TABLE"
}
```

### Partitions
```bash
# Get partitions
aws glue get-partitions \
    --database-name analytics \
    --table-name raw_logs

# Create partition
aws glue create-partition \
    --database-name analytics \
    --table-name raw_logs \
    --partition-input '{
        "Values": ["2024", "01", "15"],
        "StorageDescriptor": {
            "Location": "s3://my-datalake/raw/logs/year=2024/month=01/day=15/"
        }
    }'

# Batch create partitions
aws glue batch-create-partition \
    --database-name analytics \
    --table-name raw_logs \
    --partition-input-list file://partitions.json
```

## Crawlers

### Create Crawler
```bash
# S3 crawler
aws glue create-crawler \
    --name s3-data-crawler \
    --role arn:aws:iam::123456789012:role/GlueServiceRole \
    --database-name analytics \
    --targets '{
        "S3Targets": [
            {"Path": "s3://my-datalake/raw/"},
            {"Path": "s3://my-datalake/processed/"}
        ]
    }' \
    --schedule "cron(0 12 * * ? *)" \
    --schema-change-policy '{
        "UpdateBehavior": "UPDATE_IN_DATABASE",
        "DeleteBehavior": "LOG"
    }' \
    --configuration '{
        "Version": 1.0,
        "CrawlerOutput": {
            "Partitions": {"AddOrUpdateBehavior": "InheritFromTable"}
        },
        "Grouping": {"TableGroupingPolicy": "CombineCompatibleSchemas"}
    }'

# JDBC crawler (database)
aws glue create-crawler \
    --name rds-crawler \
    --role arn:aws:iam::123456789012:role/GlueServiceRole \
    --database-name analytics \
    --targets '{
        "JdbcTargets": [{
            "ConnectionName": "rds-connection",
            "Path": "mydb/%"
        }]
    }'
```

### Manage Crawlers
```bash
# Start crawler
aws glue start-crawler --name s3-data-crawler

# Stop crawler
aws glue stop-crawler --name s3-data-crawler

# Get crawler status
aws glue get-crawler --name s3-data-crawler

# List crawlers
aws glue list-crawlers

# Update crawler
aws glue update-crawler \
    --name s3-data-crawler \
    --schedule "cron(0 */6 * * ? *)"

# Delete crawler
aws glue delete-crawler --name s3-data-crawler
```

## ETL Jobs

### Create Job
```bash
# Spark ETL job
aws glue create-job \
    --name parquet-transformer \
    --role arn:aws:iam::123456789012:role/GlueServiceRole \
    --command '{
        "Name": "glueetl",
        "ScriptLocation": "s3://my-scripts/transform.py",
        "PythonVersion": "3"
    }' \
    --glue-version "4.0" \
    --worker-type G.1X \
    --number-of-workers 10 \
    --default-arguments '{
        "--job-bookmark-option": "job-bookmark-enable",
        "--enable-metrics": "true",
        "--enable-continuous-cloudwatch-log": "true",
        "--TempDir": "s3://my-temp-bucket/glue/"
    }'

# Python Shell job (simpler, cheaper)
aws glue create-job \
    --name simple-etl \
    --role arn:aws:iam::123456789012:role/GlueServiceRole \
    --command '{
        "Name": "pythonshell",
        "ScriptLocation": "s3://my-scripts/simple.py",
        "PythonVersion": "3.9"
    }' \
    --max-capacity 0.0625
```

### Run Jobs
```bash
# Start job run
aws glue start-job-run \
    --job-name parquet-transformer \
    --arguments '{
        "--input_path": "s3://source-bucket/data/",
        "--output_path": "s3://dest-bucket/processed/"
    }'

# Start with override settings
aws glue start-job-run \
    --job-name parquet-transformer \
    --worker-type G.2X \
    --number-of-workers 20

# Get job run status
aws glue get-job-run \
    --job-name parquet-transformer \
    --run-id jr_abc123

# List job runs
aws glue get-job-runs --job-name parquet-transformer

# Stop job run
aws glue batch-stop-job-run \
    --job-name parquet-transformer \
    --job-run-ids jr_abc123
```

### Manage Jobs
```bash
# List jobs
aws glue list-jobs

# Get job details
aws glue get-job --job-name parquet-transformer

# Update job
aws glue update-job \
    --job-name parquet-transformer \
    --job-update '{
        "NumberOfWorkers": 15,
        "WorkerType": "G.2X"
    }'

# Delete job
aws glue delete-job --job-name parquet-transformer

# Reset job bookmark
aws glue reset-job-bookmark --job-name parquet-transformer
```

## Triggers

### Create Triggers
```bash
# Scheduled trigger
aws glue create-trigger \
    --name daily-etl-trigger \
    --type SCHEDULED \
    --schedule "cron(0 8 * * ? *)" \
    --actions '[{"JobName": "parquet-transformer"}]' \
    --start-on-creation

# On-demand trigger
aws glue create-trigger \
    --name manual-trigger \
    --type ON_DEMAND \
    --actions '[{"JobName": "parquet-transformer"}]'

# Conditional trigger (run when another job succeeds)
aws glue create-trigger \
    --name conditional-trigger \
    --type CONDITIONAL \
    --predicate '{
        "Logical": "AND",
        "Conditions": [{
            "LogicalOperator": "EQUALS",
            "JobName": "extract-job",
            "State": "SUCCEEDED"
        }]
    }' \
    --actions '[{"JobName": "transform-job"}]' \
    --start-on-creation

# Event trigger (EventBridge)
aws glue create-trigger \
    --name event-trigger \
    --type EVENT \
    --actions '[{"JobName": "parquet-transformer"}]'
```

### Manage Triggers
```bash
# Start trigger
aws glue start-trigger --name daily-etl-trigger

# Stop trigger
aws glue stop-trigger --name daily-etl-trigger

# Activate on-demand trigger
aws glue start-trigger --name manual-trigger

# Get trigger
aws glue get-trigger --name daily-etl-trigger

# List triggers
aws glue list-triggers

# Delete trigger
aws glue delete-trigger --name daily-etl-trigger
```

## Workflows

### Create Workflow
```bash
# Create workflow
aws glue create-workflow \
    --name data-pipeline \
    --description "Daily data processing pipeline"

# Add triggers to workflow
aws glue create-trigger \
    --name workflow-start \
    --workflow-name data-pipeline \
    --type SCHEDULED \
    --schedule "cron(0 8 * * ? *)" \
    --actions '[{"JobName": "extract-job"}]'

aws glue create-trigger \
    --name extract-complete \
    --workflow-name data-pipeline \
    --type CONDITIONAL \
    --predicate '{"Conditions": [{"JobName": "extract-job", "State": "SUCCEEDED"}]}' \
    --actions '[{"JobName": "transform-job"}]'

aws glue create-trigger \
    --name transform-complete \
    --workflow-name data-pipeline \
    --type CONDITIONAL \
    --predicate '{"Conditions": [{"JobName": "transform-job", "State": "SUCCEEDED"}]}' \
    --actions '[{"JobName": "load-job"}]'
```

### Manage Workflows
```bash
# Start workflow run
aws glue start-workflow-run --name data-pipeline

# Get workflow
aws glue get-workflow --name data-pipeline

# Get workflow run
aws glue get-workflow-run \
    --name data-pipeline \
    --run-id wr_abc123

# List workflow runs
aws glue get-workflow-runs --name data-pipeline

# Stop workflow run
aws glue stop-workflow-run \
    --name data-pipeline \
    --run-id wr_abc123

# Delete workflow
aws glue delete-workflow --name data-pipeline
```

## Connections

### Create Connection
```bash
# JDBC connection (RDS/Aurora)
aws glue create-connection \
    --connection-input '{
        "Name": "rds-connection",
        "ConnectionType": "JDBC",
        "ConnectionProperties": {
            "JDBC_CONNECTION_URL": "jdbc:postgresql://mydb.abc123.us-east-1.rds.amazonaws.com:5432/mydb",
            "USERNAME": "admin",
            "PASSWORD": "password"
        },
        "PhysicalConnectionRequirements": {
            "SubnetId": "subnet-12345",
            "SecurityGroupIdList": ["sg-12345"],
            "AvailabilityZone": "us-east-1a"
        }
    }'

# Network connection (for VPC resources)
aws glue create-connection \
    --connection-input '{
        "Name": "vpc-connection",
        "ConnectionType": "NETWORK",
        "PhysicalConnectionRequirements": {
            "SubnetId": "subnet-12345",
            "SecurityGroupIdList": ["sg-12345"]
        }
    }'
```

### Manage Connections
```bash
# List connections
aws glue get-connections

# Get connection
aws glue get-connection --name rds-connection

# Test connection (via job)
aws glue start-job-run \
    --job-name connection-test \
    --arguments '{"--connection_name": "rds-connection"}'

# Delete connection
aws glue delete-connection --name rds-connection
```

## Schema Registry

### Create Registry
```bash
aws glue create-registry \
    --registry-name streaming-schemas \
    --description "Schemas for streaming data"
```

### Create Schema
```bash
# Avro schema
aws glue create-schema \
    --registry-id RegistryName=streaming-schemas \
    --schema-name user-events \
    --data-format AVRO \
    --compatibility BACKWARD \
    --schema-definition file://schema.avsc

# JSON schema
aws glue create-schema \
    --registry-id RegistryName=streaming-schemas \
    --schema-name api-logs \
    --data-format JSON \
    --compatibility FULL \
    --schema-definition file://schema.json
```

### Manage Schemas
```bash
# List schemas
aws glue list-schemas --registry-id RegistryName=streaming-schemas

# Get schema
aws glue get-schema --schema-id SchemaName=user-events,RegistryName=streaming-schemas

# Register new version
aws glue register-schema-version \
    --schema-id SchemaName=user-events,RegistryName=streaming-schemas \
    --schema-definition file://schema-v2.avsc

# List schema versions
aws glue list-schema-versions \
    --schema-id SchemaName=user-events,RegistryName=streaming-schemas
```

## Useful Queries

```bash
# Get job run status
aws glue get-job-runs \
    --job-name my-job \
    --query 'JobRuns[0].{Status:JobRunState,Started:StartedOn,Duration:ExecutionTime}'

# Find failed job runs
aws glue get-job-runs \
    --job-name my-job \
    --query 'JobRuns[?JobRunState==`FAILED`].{RunId:Id,Error:ErrorMessage}'

# List all tables in database
aws glue get-tables \
    --database-name analytics \
    --query 'TableList[*].Name'

# Get crawler metrics
aws glue get-crawler-metrics --crawler-name-list s3-data-crawler
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Worker types** | Use G.1X for most jobs; G.2X for memory-intensive |
| **Job bookmarks** | Enable for incremental processing |
| **Crawlers** | Use specific prefixes, schedule during low-traffic |
| **Partitions** | Partition by date for time-series data |
| **Workflows** | Use for complex multi-job pipelines |
| **Connections** | Use VPC connections for private resources |
| **Triggers** | Prefer conditional triggers over polling |
| **Schema Registry** | Use for Kafka/Kinesis data validation |
| **Monitoring** | Enable CloudWatch logs and metrics |
| **Cost** | Use Python Shell for simple scripts |
