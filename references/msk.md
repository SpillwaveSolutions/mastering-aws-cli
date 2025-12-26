# Amazon MSK (Managed Kafka)

## Cluster Types

### Serverless Cluster
```bash
# Create serverless cluster (auto-scaling)
aws kafka create-cluster-v2 \
    --cluster-name app-events \
    --serverless '{
        "VpcConfigs": [{
            "SubnetIds": ["subnet-1", "subnet-2", "subnet-3"],
            "SecurityGroupIds": ["sg-12345"]
        }],
        "ClientAuthentication": {
            "Sasl": {"Iam": {"Enabled": true}}
        }
    }'
```

### Provisioned Cluster
```bash
# Create provisioned cluster
aws kafka create-cluster \
    --cluster-name production-cluster \
    --kafka-version "3.5.1" \
    --number-of-broker-nodes 3 \
    --broker-node-group-info '{
        "InstanceType": "kafka.m5.large",
        "ClientSubnets": ["subnet-1", "subnet-2", "subnet-3"],
        "SecurityGroups": ["sg-12345"],
        "StorageInfo": {
            "EbsStorageInfo": {
                "VolumeSize": 100,
                "ProvisionedThroughput": {
                    "Enabled": true,
                    "VolumeThroughput": 250
                }
            }
        }
    }' \
    --encryption-info '{
        "EncryptionInTransit": {
            "ClientBroker": "TLS",
            "InCluster": true
        },
        "EncryptionAtRest": {
            "DataVolumeKMSKeyId": "alias/aws/kafka"
        }
    }' \
    --client-authentication '{
        "Sasl": {"Iam": {"Enabled": true}},
        "Tls": {"CertificateAuthorityArnList": []}
    }' \
    --enhanced-monitoring PER_TOPIC_PER_BROKER \
    --logging-info '{
        "BrokerLogs": {
            "CloudWatchLogs": {
                "Enabled": true,
                "LogGroup": "/aws/msk/production-cluster"
            }
        }
    }'
```

## Cluster Management

### List and Describe
```bash
# List clusters
aws kafka list-clusters-v2

# Describe cluster
aws kafka describe-cluster-v2 --cluster-arn $CLUSTER_ARN

# Get bootstrap brokers
aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN

# Get cluster operation (during updates)
aws kafka describe-cluster-operation --cluster-operation-arn $OP_ARN
```

### Update Cluster
```bash
# Update broker count (scale out)
aws kafka update-broker-count \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --target-number-of-broker-nodes 6

# Update broker storage
aws kafka update-broker-storage \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --target-broker-ebs-volume-info '[{"KafkaBrokerNodeId": "1", "VolumeSizeGB": 200}]'

# Update broker type
aws kafka update-broker-type \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --target-instance-type kafka.m5.xlarge

# Update Kafka version
aws kafka update-cluster-kafka-version \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --target-kafka-version "3.6.0"

# Update monitoring
aws kafka update-monitoring \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --enhanced-monitoring PER_TOPIC_PER_PARTITION \
    --open-monitoring '{
        "Prometheus": {
            "JmxExporter": {"EnabledInBroker": true},
            "NodeExporter": {"EnabledInBroker": true}
        }
    }'
```

### Delete Cluster
```bash
aws kafka delete-cluster --cluster-arn $CLUSTER_ARN
```

## Configuration

### Create Configuration
```bash
# Create custom configuration
aws kafka create-configuration \
    --name production-config \
    --kafka-versions "3.5.1" "3.6.0" \
    --server-properties "$(cat <<EOF
auto.create.topics.enable=false
default.replication.factor=3
min.insync.replicas=2
num.partitions=6
log.retention.hours=168
log.segment.bytes=1073741824
compression.type=lz4
EOF
)"
```

### Apply Configuration
```bash
# Update cluster to use configuration
aws kafka update-cluster-configuration \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --configuration-info Arn=$CONFIG_ARN,Revision=1
```

### Manage Configurations
```bash
# List configurations
aws kafka list-configurations

# Describe configuration
aws kafka describe-configuration --arn $CONFIG_ARN

# List configuration revisions
aws kafka list-configuration-revisions --arn $CONFIG_ARN

# Update configuration (creates new revision)
aws kafka update-configuration \
    --arn $CONFIG_ARN \
    --server-properties "auto.create.topics.enable=true"
```

## MSK Connect

### Create Connector
```bash
# Create MSK Connect connector
aws kafkaconnect create-connector \
    --connector-name s3-sink-connector \
    --kafka-cluster '{
        "ApacheKafkaCluster": {
            "BootstrapServers": "b-1.cluster.abc123.kafka.us-east-1.amazonaws.com:9098",
            "Vpc": {
                "SecurityGroups": ["sg-12345"],
                "Subnets": ["subnet-1", "subnet-2"]
            }
        }
    }' \
    --kafka-cluster-client-authentication '{
        "AuthenticationType": "IAM"
    }' \
    --kafka-cluster-encryption-in-transit '{
        "EncryptionType": "TLS"
    }' \
    --capacity '{
        "AutoScaling": {
            "MaxWorkerCount": 4,
            "MinWorkerCount": 1,
            "ScaleInPolicy": {"CpuUtilizationPercentage": 20},
            "ScaleOutPolicy": {"CpuUtilizationPercentage": 80},
            "McuCount": 1
        }
    }' \
    --connector-configuration '{
        "connector.class": "io.confluent.connect.s3.S3SinkConnector",
        "tasks.max": "2",
        "topics": "my-topic",
        "s3.bucket.name": "my-bucket",
        "s3.region": "us-east-1",
        "flush.size": "1000",
        "storage.class": "io.confluent.connect.s3.storage.S3Storage",
        "format.class": "io.confluent.connect.s3.format.json.JsonFormat"
    }' \
    --plugins '[{
        "CustomPlugin": {
            "CustomPluginArn": "arn:aws:kafkaconnect:us-east-1:123456789012:custom-plugin/s3-plugin/abc123",
            "Revision": 1
        }
    }]' \
    --service-execution-role-arn arn:aws:iam::123456789012:role/MSKConnectRole
```

### Manage Connectors
```bash
# List connectors
aws kafkaconnect list-connectors

# Describe connector
aws kafkaconnect describe-connector --connector-arn $CONNECTOR_ARN

# Update connector
aws kafkaconnect update-connector \
    --connector-arn $CONNECTOR_ARN \
    --current-version $VERSION \
    --capacity '{
        "AutoScaling": {
            "MaxWorkerCount": 8,
            "MinWorkerCount": 2,
            "McuCount": 2
        }
    }'

# Delete connector
aws kafkaconnect delete-connector --connector-arn $CONNECTOR_ARN
```

### Custom Plugins
```bash
# Create custom plugin
aws kafkaconnect create-custom-plugin \
    --name s3-sink-plugin \
    --content-type ZIP \
    --location '{
        "S3Location": {
            "BucketArn": "arn:aws:s3:::my-plugins-bucket",
            "FileKey": "plugins/s3-sink-connector.zip"
        }
    }'

# List plugins
aws kafkaconnect list-custom-plugins
```

## Replicator

### Create Replicator
```bash
# Create cross-region replicator
aws kafka create-replicator \
    --replicator-name cross-region-replicator \
    --kafka-clusters '[
        {
            "AmazonMskCluster": {"MskClusterArn": "arn:aws:kafka:us-east-1:123456789012:cluster/source"},
            "VpcConfig": {"SecurityGroupIds": ["sg-source"], "SubnetIds": ["subnet-source"]}
        },
        {
            "AmazonMskCluster": {"MskClusterArn": "arn:aws:kafka:eu-west-1:123456789012:cluster/target"},
            "VpcConfig": {"SecurityGroupIds": ["sg-target"], "SubnetIds": ["subnet-target"]}
        }
    ]' \
    --replication-info-list '[{
        "SourceKafkaClusterArn": "arn:aws:kafka:us-east-1:123456789012:cluster/source",
        "TargetKafkaClusterArn": "arn:aws:kafka:eu-west-1:123456789012:cluster/target",
        "TopicReplication": {
            "TopicsToReplicate": ["orders.*", "events.*"],
            "CopyTopicConfigurations": true,
            "DetectAndCopyNewTopics": true
        },
        "ConsumerGroupReplication": {
            "ConsumerGroupsToReplicate": [".*"]
        },
        "TargetCompressionType": "LZ4"
    }]' \
    --service-execution-role-arn arn:aws:iam::123456789012:role/MSKReplicatorRole
```

## Monitoring

### CloudWatch Metrics
```bash
# Get broker CPU metrics
aws cloudwatch get-metric-statistics \
    --namespace "AWS/Kafka" \
    --metric-name "CpuUser" \
    --dimensions Name=ClusterName,Value=production-cluster Name=BrokerId,Value=1 \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Average

# Get consumer lag
aws cloudwatch get-metric-statistics \
    --namespace "AWS/Kafka" \
    --metric-name "EstimatedMaxTimeLag" \
    --dimensions Name=ClusterName,Value=production-cluster Name=Topic,Value=my-topic Name=ConsumerGroup,Value=my-group \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 60 \
    --statistics Maximum
```

### Enable Prometheus Monitoring
```bash
aws kafka update-monitoring \
    --cluster-arn $CLUSTER_ARN \
    --current-version $CLUSTER_VERSION \
    --open-monitoring '{
        "Prometheus": {
            "JmxExporter": {"EnabledInBroker": true},
            "NodeExporter": {"EnabledInBroker": true}
        }
    }'
```

## Useful Queries

```bash
# Get bootstrap brokers for IAM auth
aws kafka get-bootstrap-brokers \
    --cluster-arn $CLUSTER_ARN \
    --query 'BootstrapBrokerStringSaslIam' \
    --output text

# Get cluster status
aws kafka describe-cluster-v2 \
    --cluster-arn $CLUSTER_ARN \
    --query 'ClusterInfo.State'

# List cluster operations
aws kafka list-cluster-operations --cluster-arn $CLUSTER_ARN

# Get Zookeeper connection string
aws kafka describe-cluster \
    --cluster-arn $CLUSTER_ARN \
    --query 'ClusterInfo.ZookeeperConnectString'
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Serverless** | Use for variable or unpredictable workloads |
| **IAM auth** | Use IAM for AWS service integration |
| **Encryption** | Enable TLS in-transit and KMS at-rest |
| **Monitoring** | Enable PER_TOPIC_PER_BROKER for debugging |
| **Replication** | Set min.insync.replicas=2 for durability |
| **Partitions** | Plan partition count based on throughput needs |
| **MSK Connect** | Use for managed Kafka Connect deployments |
| **Tiered storage** | Enable for cost-effective long retention |
| **Multi-AZ** | Use 3 AZs for high availability |
| **Prometheus** | Enable for detailed JMX metrics |
