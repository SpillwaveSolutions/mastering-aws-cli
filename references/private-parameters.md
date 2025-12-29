# Secrets & Parameters Management

## When to Use Which

| Need | Use | Why |
|------|-----|-----|
| App config, feature flags | Parameter Store | Free, simple hierarchies |
| API keys, passwords | Secrets Manager | Auto-rotation, audit |
| Database credentials | Secrets Manager | RDS integration |
| Simple key-value | Parameter Store | Lower cost |
| Cross-region secrets | Secrets Manager | Built-in replication |

## Parameter Store

### Create/Update Parameters

```bash
# String parameter
aws ssm put-parameter --name "/app/prod/db/host" --value "db.example.com" --type String

# SecureString with default KMS
aws ssm put-parameter --name "/app/prod/db/password" --value "secret123" --type SecureString

# SecureString with custom KMS key
aws ssm put-parameter --name "/app/prod/api/key" --value "sk-xxx" --type SecureString --key-id alias/my-key

# Update existing (requires --overwrite)
aws ssm put-parameter --name "/app/prod/db/host" --value "new-db.example.com" --type String --overwrite

# StringList
aws ssm put-parameter --name "/app/prod/allowed-ips" --value "10.0.0.1,10.0.0.2,10.0.0.3" --type StringList
```

### Retrieve Parameters

```bash
# Single parameter
aws ssm get-parameter --name "/app/prod/db/host"

# Decrypt SecureString
aws ssm get-parameter --name "/app/prod/db/password" --with-decryption

# Value only
aws ssm get-parameter --name "/app/prod/db/host" --query "Parameter.Value" --output text

# Multiple parameters
aws ssm get-parameters --names "/app/prod/db/host" "/app/prod/db/port" --with-decryption

# By path (hierarchy)
aws ssm get-parameters-by-path --path "/app/prod" --recursive --with-decryption

# By path with filter
aws ssm get-parameters-by-path --path "/app/prod" --parameter-filters "Key=Type,Values=SecureString"
```

### Parameter Hierarchies

```
/app
  /prod
    /db
      /host
      /password
    /api
      /key
  /dev
    /db
      /host
```

Benefits:
- Organize by environment, service, component
- Retrieve entire subtrees with `get-parameters-by-path`
- Apply IAM policies to paths

### Delete Parameters

```bash
# Single delete
aws ssm delete-parameter --name "/app/prod/old-param"

# Bulk delete
aws ssm delete-parameters --names "/app/prod/old1" "/app/prod/old2"
```

### Parameter History

```bash
# View version history
aws ssm get-parameter-history --name "/app/prod/db/host"

# Get specific version
aws ssm get-parameter --name "/app/prod/db/host:2"

# Get labels
aws ssm get-parameter-history --name "/app/prod/db/host" --with-decryption
```

## Secrets Manager

### Create Secrets

```bash
# Simple string secret
aws secretsmanager create-secret --name "prod/api/stripe-key" --secret-string "sk_live_xxx"

# JSON secret (common for credentials)
aws secretsmanager create-secret --name "prod/db/credentials" --secret-string '{"username":"admin","password":"secret123","host":"db.example.com"}'

# With description and tags
aws secretsmanager create-secret \
  --name "prod/api/key" \
  --secret-string "xxx" \
  --description "Production API key" \
  --tags Key=Environment,Value=prod Key=Team,Value=backend

# Binary secret
aws secretsmanager create-secret --name "prod/certs/tls" --secret-binary fileb://cert.der
```

### Retrieve Secrets

```bash
# Get current version
aws secretsmanager get-secret-value --secret-id "prod/api/key"

# Value only
aws secretsmanager get-secret-value --secret-id "prod/api/key" --query SecretString --output text

# Parse JSON secret
aws secretsmanager get-secret-value --secret-id "prod/db/credentials" --query SecretString --output text | jq -r .password

# Get specific version
aws secretsmanager get-secret-value --secret-id "prod/api/key" --version-id "abc123"

# Get by stage
aws secretsmanager get-secret-value --secret-id "prod/api/key" --version-stage AWSPREVIOUS
```

### Update Secrets

```bash
# Update value
aws secretsmanager put-secret-value --secret-id "prod/api/key" --secret-string "new-value"

# Update metadata only
aws secretsmanager update-secret --secret-id "prod/api/key" --description "Updated description"

# Rotate with custom KMS key
aws secretsmanager update-secret --secret-id "prod/api/key" --kms-key-id alias/new-key
```

### Rotation

```bash
# Enable rotation with Lambda
aws secretsmanager rotate-secret \
  --secret-id "prod/db/credentials" \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789:function:rotate-db-creds \
  --rotation-rules AutomaticallyAfterDays=30

# Trigger immediate rotation
aws secretsmanager rotate-secret --secret-id "prod/db/credentials"

# Check rotation status
aws secretsmanager describe-secret --secret-id "prod/db/credentials" --query "RotationEnabled"
```

### Delete and Restore

```bash
# Schedule deletion (7-30 days)
aws secretsmanager delete-secret --secret-id "prod/old-key" --recovery-window-in-days 7

# Force immediate deletion (cannot recover!)
aws secretsmanager delete-secret --secret-id "prod/old-key" --force-delete-without-recovery

# Cancel deletion/restore
aws secretsmanager restore-secret --secret-id "prod/old-key"
```

### Cross-Region Replication

```bash
# Add replica regions
aws secretsmanager replicate-secret-to-regions \
  --secret-id "prod/api/key" \
  --add-replica-regions Region=eu-west-1 Region=ap-southeast-1

# Remove replica
aws secretsmanager remove-regions-from-replication \
  --secret-id "prod/api/key" \
  --remove-replica-regions eu-west-1
```

## Cross-Service Patterns

### Lambda Environment from Secrets Manager

```bash
# Store Lambda config as secret
aws secretsmanager create-secret --name "prod/lambda/my-func/config" \
  --secret-string '{"API_KEY":"xxx","DB_HOST":"db.example.com"}'

# Lambda reads at runtime:
# boto3.client("secretsmanager").get_secret_value(SecretId="prod/lambda/my-func/config")
```

### ECS Task from Parameter Store

```bash
# Store task secrets
aws ssm put-parameter --name "/ecs/prod/my-service/db-password" --value "xxx" --type SecureString

# Reference in task definition:
# "secrets": [{"name": "DB_PASSWORD", "valueFrom": "/ecs/prod/my-service/db-password"}]
```

## Useful Queries

```bash
# List all parameters by path
aws ssm describe-parameters --parameter-filters "Key=Path,Values=/app/prod"

# Find parameters by tag
aws ssm describe-parameters --parameter-filters "Key=tag:Environment,Values=production"

# List all secrets
aws secretsmanager list-secrets

# Find secrets by name pattern
aws secretsmanager list-secrets --filters Key=name,Values=prod

# Find secrets by tag
aws secretsmanager list-secrets --filters Key=tag-key,Values=Environment Key=tag-value,Values=prod
```

## Best Practices

1. **Use hierarchies**: `/app/env/service/key` for organization
2. **Tag everything**: Environment, Team, Application for filtering
3. **Prefer SecureString**: Always encrypt sensitive values
4. **Rotate regularly**: Enable auto-rotation for Secrets Manager
5. **Limit IAM scope**: Grant access to specific paths, not `*`
6. **Audit access**: Enable CloudTrail for both services
7. **Use resource policies**: Control cross-account access
