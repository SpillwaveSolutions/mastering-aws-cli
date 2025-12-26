# Aurora & RDS

## Aurora Serverless v2

### Create Cluster
```bash
# Aurora PostgreSQL Serverless v2
aws rds create-db-cluster \
    --db-cluster-identifier my-cluster \
    --engine aurora-postgresql \
    --engine-version 15.4 \
    --master-username admin \
    --master-user-password SecurePassword123! \
    --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
    --db-subnet-group-name my-subnet-group \
    --vpc-security-group-ids sg-12345678 \
    --storage-encrypted \
    --enable-cloudwatch-logs-exports postgresql

# Aurora MySQL Serverless v2
aws rds create-db-cluster \
    --db-cluster-identifier mysql-cluster \
    --engine aurora-mysql \
    --engine-version 3.04.0 \
    --master-username admin \
    --master-user-password SecurePassword123! \
    --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=32 \
    --db-subnet-group-name my-subnet-group \
    --vpc-security-group-ids sg-12345678
```

### Create Instance
```bash
# Primary writer instance
aws rds create-db-instance \
    --db-instance-identifier my-primary \
    --db-cluster-identifier my-cluster \
    --engine aurora-postgresql \
    --db-instance-class db.serverless

# Add reader instance
aws rds create-db-instance \
    --db-instance-identifier my-reader-1 \
    --db-cluster-identifier my-cluster \
    --engine aurora-postgresql \
    --db-instance-class db.serverless

# Wait for instance
aws rds wait db-instance-available --db-instance-identifier my-primary
```

## Aurora Provisioned

### Create Cluster
```bash
# Aurora PostgreSQL with provisioned capacity
aws rds create-db-cluster \
    --db-cluster-identifier prod-cluster \
    --engine aurora-postgresql \
    --engine-version 15.4 \
    --master-username admin \
    --master-user-password SecurePassword123! \
    --db-subnet-group-name my-subnet-group \
    --vpc-security-group-ids sg-12345678 \
    --storage-encrypted \
    --kms-key-id alias/aws/rds \
    --backup-retention-period 7 \
    --preferred-backup-window "03:00-04:00" \
    --preferred-maintenance-window "sun:05:00-sun:06:00"

# Create writer instance
aws rds create-db-instance \
    --db-instance-identifier prod-primary \
    --db-cluster-identifier prod-cluster \
    --engine aurora-postgresql \
    --db-instance-class db.r6g.large \
    --publicly-accessible false

# Create reader instance
aws rds create-db-instance \
    --db-instance-identifier prod-reader-1 \
    --db-cluster-identifier prod-cluster \
    --engine aurora-postgresql \
    --db-instance-class db.r6g.large \
    --publicly-accessible false
```

## Standard RDS

### Create Instance
```bash
# RDS PostgreSQL
aws rds create-db-instance \
    --db-instance-identifier prod-postgres \
    --db-instance-class db.t3.medium \
    --engine postgres \
    --engine-version 15.4 \
    --allocated-storage 100 \
    --max-allocated-storage 500 \
    --master-username admin \
    --master-user-password SecurePassword123! \
    --db-subnet-group-name my-subnet-group \
    --vpc-security-group-ids sg-12345678 \
    --storage-encrypted \
    --multi-az \
    --publicly-accessible false \
    --backup-retention-period 7

# RDS MySQL
aws rds create-db-instance \
    --db-instance-identifier prod-mysql \
    --db-instance-class db.r6g.large \
    --engine mysql \
    --engine-version 8.0.35 \
    --allocated-storage 100 \
    --max-allocated-storage 1000 \
    --master-username admin \
    --master-user-password SecurePassword123! \
    --storage-type gp3 \
    --iops 3000 \
    --storage-throughput 125
```

## Cluster and Instance Management

### Describe and List
```bash
# List clusters
aws rds describe-db-clusters

# Describe specific cluster
aws rds describe-db-clusters --db-cluster-identifier my-cluster

# Get cluster endpoint
aws rds describe-db-clusters \
    --db-cluster-identifier my-cluster \
    --query 'DBClusters[0].Endpoint' \
    --output text

# Get reader endpoint
aws rds describe-db-clusters \
    --db-cluster-identifier my-cluster \
    --query 'DBClusters[0].ReaderEndpoint' \
    --output text

# List instances
aws rds describe-db-instances

# Describe specific instance
aws rds describe-db-instances --db-instance-identifier my-primary
```

### Modify Cluster
```bash
# Modify scaling configuration
aws rds modify-db-cluster \
    --db-cluster-identifier my-cluster \
    --serverless-v2-scaling-configuration MinCapacity=1,MaxCapacity=32 \
    --apply-immediately

# Enable deletion protection
aws rds modify-db-cluster \
    --db-cluster-identifier my-cluster \
    --deletion-protection

# Change maintenance window
aws rds modify-db-cluster \
    --db-cluster-identifier my-cluster \
    --preferred-maintenance-window "sun:05:00-sun:06:00"
```

### Modify Instance
```bash
# Change instance class
aws rds modify-db-instance \
    --db-instance-identifier my-primary \
    --db-instance-class db.r6g.xlarge \
    --apply-immediately

# Scale storage (RDS only)
aws rds modify-db-instance \
    --db-instance-identifier prod-postgres \
    --allocated-storage 200 \
    --apply-immediately
```

### Delete
```bash
# Delete instance (skip final snapshot)
aws rds delete-db-instance \
    --db-instance-identifier my-reader-1 \
    --skip-final-snapshot

# Delete cluster (with final snapshot)
aws rds delete-db-cluster \
    --db-cluster-identifier my-cluster \
    --final-db-snapshot-identifier my-cluster-final-snapshot

# Delete cluster (skip final snapshot)
aws rds delete-db-cluster \
    --db-cluster-identifier my-cluster \
    --skip-final-snapshot
```

## Aurora Global Database

### Create Global Database
```bash
# Create global database from existing cluster
aws rds create-global-cluster \
    --global-cluster-identifier my-global-db \
    --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:my-cluster

# Add secondary region
aws rds create-db-cluster \
    --db-cluster-identifier my-cluster-eu \
    --engine aurora-postgresql \
    --engine-version 15.4 \
    --global-cluster-identifier my-global-db \
    --db-subnet-group-name eu-subnet-group \
    --vpc-security-group-ids sg-eu-12345 \
    --region eu-west-1

# Add instance in secondary region
aws rds create-db-instance \
    --db-instance-identifier my-cluster-eu-1 \
    --db-cluster-identifier my-cluster-eu \
    --engine aurora-postgresql \
    --db-instance-class db.r6g.large \
    --region eu-west-1
```

### Manage Global Database
```bash
# Describe global database
aws rds describe-global-clusters --global-cluster-identifier my-global-db

# Failover to secondary (planned)
aws rds failover-global-cluster \
    --global-cluster-identifier my-global-db \
    --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:my-cluster-eu

# Remove cluster from global database
aws rds remove-from-global-cluster \
    --global-cluster-identifier my-global-db \
    --db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:my-cluster-eu

# Delete global database
aws rds delete-global-cluster --global-cluster-identifier my-global-db
```

## RDS Proxy

### Create Proxy
```bash
# Create RDS Proxy
aws rds create-db-proxy \
    --db-proxy-name my-proxy \
    --engine-family POSTGRESQL \
    --auth '[{
        "AuthScheme": "SECRETS",
        "SecretArn": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-creds",
        "IAMAuth": "DISABLED"
    }]' \
    --role-arn arn:aws:iam::123456789012:role/RDSProxyRole \
    --vpc-subnet-ids subnet-1 subnet-2 \
    --vpc-security-group-ids sg-12345678 \
    --require-tls

# Register target (Aurora cluster)
aws rds register-db-proxy-targets \
    --db-proxy-name my-proxy \
    --db-cluster-identifiers my-cluster

# Register target (RDS instance)
aws rds register-db-proxy-targets \
    --db-proxy-name my-proxy \
    --db-instance-identifiers prod-postgres
```

### Manage Proxy
```bash
# Describe proxy
aws rds describe-db-proxies --db-proxy-name my-proxy

# Get proxy endpoint
aws rds describe-db-proxies \
    --db-proxy-name my-proxy \
    --query 'DBProxies[0].Endpoint' \
    --output text

# Describe target groups
aws rds describe-db-proxy-target-groups --db-proxy-name my-proxy

# Delete proxy
aws rds delete-db-proxy --db-proxy-name my-proxy
```

## Snapshots and Cloning

### Manual Snapshots
```bash
# Create cluster snapshot
aws rds create-db-cluster-snapshot \
    --db-cluster-snapshot-identifier my-cluster-snap-01 \
    --db-cluster-identifier my-cluster

# Create instance snapshot (RDS only)
aws rds create-db-snapshot \
    --db-snapshot-identifier prod-postgres-snap-01 \
    --db-instance-identifier prod-postgres

# List snapshots
aws rds describe-db-cluster-snapshots --db-cluster-identifier my-cluster

# Copy snapshot cross-region
aws rds copy-db-cluster-snapshot \
    --source-db-cluster-snapshot-identifier arn:aws:rds:us-east-1:123456789012:cluster-snapshot:my-cluster-snap-01 \
    --target-db-cluster-snapshot-identifier my-cluster-snap-01-copy \
    --region eu-west-1

# Share snapshot with another account
aws rds modify-db-cluster-snapshot-attribute \
    --db-cluster-snapshot-identifier my-cluster-snap-01 \
    --attribute-name restore \
    --values-to-add 987654321098
```

### Restore from Snapshot
```bash
# Restore Aurora cluster
aws rds restore-db-cluster-from-snapshot \
    --db-cluster-identifier restored-cluster \
    --snapshot-identifier my-cluster-snap-01 \
    --engine aurora-postgresql \
    --db-subnet-group-name my-subnet-group \
    --vpc-security-group-ids sg-12345678

# Then create instance
aws rds create-db-instance \
    --db-instance-identifier restored-primary \
    --db-cluster-identifier restored-cluster \
    --engine aurora-postgresql \
    --db-instance-class db.serverless

# Restore RDS instance
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier restored-postgres \
    --db-snapshot-identifier prod-postgres-snap-01 \
    --db-instance-class db.t3.medium
```

### Aurora Cloning
```bash
# Fast clone (copy-on-write)
aws rds restore-db-cluster-to-point-in-time \
    --source-db-cluster-identifier my-cluster \
    --db-cluster-identifier test-clone \
    --restore-type copy-on-write \
    --use-latest-restorable-time

# Clone to specific point in time
aws rds restore-db-cluster-to-point-in-time \
    --source-db-cluster-identifier my-cluster \
    --db-cluster-identifier test-clone \
    --restore-type copy-on-write \
    --restore-to-time "2024-01-15T12:00:00Z"
```

## Blue/Green Deployments

### Create Blue/Green Deployment
```bash
# Create deployment for version upgrade
aws rds create-blue-green-deployment \
    --blue-green-deployment-name pg-upgrade-16 \
    --source arn:aws:rds:us-east-1:123456789012:cluster:my-cluster \
    --target-engine-version 16.1

# Create deployment with instance class change
aws rds create-blue-green-deployment \
    --blue-green-deployment-name scale-up \
    --source arn:aws:rds:us-east-1:123456789012:db:prod-postgres \
    --target-db-instance-class db.r6g.xlarge
```

### Manage Blue/Green Deployment
```bash
# Describe deployment
aws rds describe-blue-green-deployments \
    --blue-green-deployment-identifier bgd-12345678

# Switchover (after verification)
aws rds switchover-blue-green-deployment \
    --blue-green-deployment-identifier bgd-12345678 \
    --switchover-timeout 300

# Delete deployment (cleanup old environment)
aws rds delete-blue-green-deployment \
    --blue-green-deployment-identifier bgd-12345678 \
    --delete-target
```

## Parameter Groups

### Create Parameter Group
```bash
# Create cluster parameter group
aws rds create-db-cluster-parameter-group \
    --db-cluster-parameter-group-name my-aurora-pg-params \
    --db-parameter-group-family aurora-postgresql15 \
    --description "Custom Aurora PostgreSQL parameters"

# Create instance parameter group
aws rds create-db-parameter-group \
    --db-parameter-group-name my-postgres-params \
    --db-parameter-group-family postgres15 \
    --description "Custom PostgreSQL parameters"
```

### Modify Parameters
```bash
# Modify cluster parameters
aws rds modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name my-aurora-pg-params \
    --parameters "ParameterName=log_statement,ParameterValue=all,ApplyMethod=immediate" \
                 "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate"

# Apply to cluster
aws rds modify-db-cluster \
    --db-cluster-identifier my-cluster \
    --db-cluster-parameter-group-name my-aurora-pg-params \
    --apply-immediately
```

## IAM Database Authentication

### Enable IAM Auth
```bash
# Enable on cluster
aws rds modify-db-cluster \
    --db-cluster-identifier my-cluster \
    --enable-iam-database-authentication \
    --apply-immediately

# Enable on RDS instance
aws rds modify-db-instance \
    --db-instance-identifier prod-postgres \
    --enable-iam-database-authentication \
    --apply-immediately
```

### Generate Auth Token
```bash
# Generate authentication token
aws rds generate-db-auth-token \
    --hostname my-cluster.cluster-abc123.us-east-1.rds.amazonaws.com \
    --port 5432 \
    --username iam_user \
    --region us-east-1
```

## Bastion Access to Aurora

### Port Forwarding via SSM
```bash
# Get cluster endpoint
ENDPOINT=$(aws rds describe-db-clusters \
    --db-cluster-identifier my-cluster \
    --query 'DBClusters[0].Endpoint' \
    --output text)

# Start port forwarding through bastion
aws ssm start-session \
    --target i-bastion-instance-id \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{\"host\":[\"$ENDPOINT\"],\"portNumber\":[\"5432\"],\"localPortNumber\":[\"5432\"]}"

# In another terminal, connect via psql
psql -h localhost -p 5432 -U admin -d mydb
```

### SSH Tunnel via SSM
```bash
# Configure SSH (~/.ssh/config)
# Host i-* mi-*
#     ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"

# Create SSH tunnel
ssh -L 5432:my-cluster.cluster-abc123.us-east-1.rds.amazonaws.com:5432 \
    ec2-user@i-bastion-instance-id -N &

# Connect via tunnel
psql -h localhost -p 5432 -U admin -d mydb
```

## Monitoring

### Enable Enhanced Monitoring
```bash
aws rds modify-db-instance \
    --db-instance-identifier my-primary \
    --monitoring-interval 60 \
    --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role
```

### Enable Performance Insights
```bash
aws rds modify-db-instance \
    --db-instance-identifier my-primary \
    --enable-performance-insights \
    --performance-insights-retention-period 7
```

### CloudWatch Logs
```bash
# Enable log exports
aws rds modify-db-cluster \
    --db-cluster-identifier my-cluster \
    --cloudwatch-logs-export-configuration \
        EnableLogTypes=postgresql,upgrade
```

## Useful Queries

```bash
# Get cluster status
aws rds describe-db-clusters \
    --db-cluster-identifier my-cluster \
    --query 'DBClusters[0].Status'

# List all endpoints
aws rds describe-db-clusters \
    --db-cluster-identifier my-cluster \
    --query 'DBClusters[0].{Writer:Endpoint,Reader:ReaderEndpoint}'

# Get current capacity (Serverless v2)
aws rds describe-db-clusters \
    --db-cluster-identifier my-cluster \
    --query 'DBClusters[0].ServerlessV2ScalingConfiguration'

# Find instances in cluster
aws rds describe-db-instances \
    --filters Name=db-cluster-id,Values=my-cluster \
    --query 'DBInstances[*].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Status:DBInstanceStatus}'
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Serverless v2** | Use MinCapacity=0.5 for dev, higher for production |
| **Secrets Manager** | Store credentials in Secrets Manager, not scripts |
| **Private access** | Keep PubliclyAccessible=false, use bastion/proxy |
| **Encryption** | Enable storage encryption with KMS |
| **Multi-AZ** | Use for production (automatic with Aurora) |
| **RDS Proxy** | Use for Lambda or connection pooling |
| **Global Database** | Use for cross-region disaster recovery |
| **Blue/Green** | Use for major version upgrades |
| **Cloning** | Use fast clones for testing |
| **IAM Auth** | Use for temporary, rotatable credentials |
