# Secrets Manager & Parameter Store

## Secrets Manager

### Create Secrets
```bash
# Create secret with key/value
aws secretsmanager create-secret \
    --name prod/myapp/db-credentials \
    --description "Production database credentials" \
    --secret-string '{"username":"admin","password":"MyP@ssw0rd!","host":"db.example.com","port":"5432"}'

# Create secret from file
aws secretsmanager create-secret \
    --name prod/myapp/api-key \
    --secret-string file://api-key.json

# Create binary secret (certificates, keys)
aws secretsmanager create-secret \
    --name prod/myapp/tls-cert \
    --secret-binary fileb://certificate.pem

# Create with KMS key
aws secretsmanager create-secret \
    --name prod/myapp/sensitive \
    --secret-string '{"key":"value"}' \
    --kms-key-id alias/secrets-key

# Create with tags
aws secretsmanager create-secret \
    --name prod/myapp/config \
    --secret-string '{"api_url":"https://api.example.com"}' \
    --tags Key=Environment,Value=Production Key=Application,Value=MyApp
```

### Retrieve Secrets
```bash
# Get secret value (latest version)
aws secretsmanager get-secret-value \
    --secret-id prod/myapp/db-credentials \
    --query SecretString \
    --output text

# Get and parse JSON
aws secretsmanager get-secret-value \
    --secret-id prod/myapp/db-credentials \
    --query SecretString \
    --output text | jq -r '.password'

# Get specific version
aws secretsmanager get-secret-value \
    --secret-id prod/myapp/db-credentials \
    --version-id "abc123-version-id"

# Get by version stage
aws secretsmanager get-secret-value \
    --secret-id prod/myapp/db-credentials \
    --version-stage AWSPREVIOUS

# Get binary secret
aws secretsmanager get-secret-value \
    --secret-id prod/myapp/tls-cert \
    --query SecretBinary \
    --output text | base64 -d > certificate.pem

# Shell: Export as environment variable
export DB_PASSWORD=\$(aws secretsmanager get-secret-value \
    --secret-id prod/myapp/db-credentials \
    --query SecretString \
    --output text | jq -r '.password')
```

### Update Secrets
```bash
# Update entire secret
aws secretsmanager update-secret \
    --secret-id prod/myapp/db-credentials \
    --secret-string '{"username":"admin","password":"NewP@ssw0rd!","host":"db.example.com","port":"5432"}'

# Put new version (creates new version)
aws secretsmanager put-secret-value \
    --secret-id prod/myapp/db-credentials \
    --secret-string '{"username":"admin","password":"RotatedP@ss!"}' \
    --version-stages AWSCURRENT

# Update description/KMS key
aws secretsmanager update-secret \
    --secret-id prod/myapp/db-credentials \
    --description "Updated description" \
    --kms-key-id alias/new-key
```

### Rotation Configuration
```bash
# Enable rotation with Lambda
aws secretsmanager rotate-secret \
    --secret-id prod/myapp/db-credentials \
    --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsRotator \
    --rotation-rules AutomaticallyAfterDays=30

# Trigger immediate rotation
aws secretsmanager rotate-secret \
    --secret-id prod/myapp/db-credentials

# Get rotation status
aws secretsmanager describe-secret \
    --secret-id prod/myapp/db-credentials \
    --query '{RotationEnabled:RotationEnabled,RotationLambda:RotationLambdaARN,RotationRules:RotationRules}'
```

### Manage Secrets
```bash
# List secrets
aws secretsmanager list-secrets

# List with filter
aws secretsmanager list-secrets \
    --filters Key=name,Values=prod/ \
    --query 'SecretList[*].{Name:Name,ARN:ARN}'

# Describe secret (metadata)
aws secretsmanager describe-secret \
    --secret-id prod/myapp/db-credentials

# Delete secret (soft delete with recovery window)
aws secretsmanager delete-secret \
    --secret-id prod/myapp/db-credentials \
    --recovery-window-in-days 7

# Delete immediately (no recovery)
aws secretsmanager delete-secret \
    --secret-id prod/myapp/db-credentials \
    --force-delete-without-recovery

# Restore deleted secret
aws secretsmanager restore-secret \
    --secret-id prod/myapp/db-credentials
```

### Resource Policy
```bash
# Get resource policy
aws secretsmanager get-resource-policy \
    --secret-id prod/myapp/db-credentials

# Put resource policy (cross-account access)
aws secretsmanager put-resource-policy \
    --secret-id prod/myapp/db-credentials \
    --resource-policy file://secret-policy.json
```

### Replication
```bash
# Replicate secret to other regions
aws secretsmanager replicate-secret-to-regions \
    --secret-id prod/myapp/db-credentials \
    --add-replica-regions Region=eu-west-1 Region=ap-southeast-1

# Remove replica
aws secretsmanager remove-regions-from-replication \
    --secret-id prod/myapp/db-credentials \
    --remove-replica-regions eu-west-1
```

## SSM Parameter Store

### Create Parameters
```bash
# String parameter
aws ssm put-parameter \
    --name "/myapp/config/api_url" \
    --value "https://api.example.com" \
    --type String \
    --description "API endpoint URL"

# Secure string (encrypted with KMS)
aws ssm put-parameter \
    --name "/myapp/secrets/api_key" \
    --value "sk-1234567890abcdef" \
    --type SecureString \
    --description "API key"

# Secure string with custom KMS key
aws ssm put-parameter \
    --name "/myapp/secrets/db_password" \
    --value "MySecretPassword" \
    --type SecureString \
    --key-id alias/parameter-store-key

# String list
aws ssm put-parameter \
    --name "/myapp/config/allowed_origins" \
    --value "https://app.example.com,https://www.example.com" \
    --type StringList

# Overwrite existing
aws ssm put-parameter \
    --name "/myapp/config/api_url" \
    --value "https://api-v2.example.com" \
    --type String \
    --overwrite
```

### Get Parameters
```bash
# Get single parameter
aws ssm get-parameter \
    --name "/myapp/config/api_url" \
    --query 'Parameter.Value' \
    --output text

# Get SecureString (decrypted)
aws ssm get-parameter \
    --name "/myapp/secrets/api_key" \
    --with-decryption \
    --query 'Parameter.Value' \
    --output text

# Get multiple parameters
aws ssm get-parameters \
    --names "/myapp/config/api_url" "/myapp/config/timeout" \
    --query 'Parameters[*].{Name:Name,Value:Value}'

# Get with decryption
aws ssm get-parameters \
    --names "/myapp/secrets/api_key" "/myapp/secrets/db_password" \
    --with-decryption
```

### Parameter Hierarchies
```bash
# Get by path (hierarchy)
aws ssm get-parameters-by-path \
    --path "/myapp/config/" \
    --query 'Parameters[*].{Name:Name,Value:Value}'

# Recursive (all nested paths)
aws ssm get-parameters-by-path \
    --path "/myapp/" \
    --recursive \
    --with-decryption
```

### Parameter History & Labels
```bash
# Get parameter history
aws ssm get-parameter-history \
    --name "/myapp/config/api_url"

# Get specific version
aws ssm get-parameter \
    --name "/myapp/config/api_url:1"

# Add label to version
aws ssm label-parameter-version \
    --name "/myapp/config/api_url" \
    --parameter-version 3 \
    --labels production stable

# Get by label
aws ssm get-parameter \
    --name "/myapp/config/api_url:production"
```

### Manage Parameters
```bash
# List all parameters
aws ssm describe-parameters

# List with filter
aws ssm describe-parameters \
    --parameter-filters Key=Name,Option=Contains,Values=myapp

# Filter by type
aws ssm describe-parameters \
    --parameter-filters Key=Type,Values=SecureString

# Delete parameter
aws ssm delete-parameter \
    --name "/myapp/config/old_setting"

# Delete multiple
aws ssm delete-parameters \
    --names "/myapp/temp/param1" "/myapp/temp/param2"
```

## Comparison: Secrets Manager vs Parameter Store

| Feature | Secrets Manager | Parameter Store |
|:--------|:----------------|:----------------|
| **Purpose** | Secrets with rotation | Configuration & secrets |
| **Rotation** | Built-in Lambda rotation | Manual or custom |
| **Cost** | \$0.40/secret/month + API | Free (standard) / \$0.05 (advanced) |
| **Size Limit** | 64 KB | 4 KB (standard) / 8 KB (advanced) |
| **Versioning** | Automatic | Automatic |
| **Cross-Region** | Built-in replication | Manual |
| **Best For** | DB passwords, API keys | App config, feature flags |

## Integration Patterns

### ECS Task Definition
```json
{
    "containerDefinitions": [{
        "name": "app",
        "secrets": [
            {
                "name": "DB_PASSWORD",
                "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-creds:password::"
            },
            {
                "name": "API_URL",
                "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/config/api_url"
            }
        ]
    }]
}
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Naming convention** | Use hierarchical paths: /env/app/component/secret |
| **Secrets Manager** | Use for credentials requiring rotation |
| **Parameter Store** | Use for config values and less sensitive data |
| **SecureString** | Always use for sensitive SSM parameters |
| **Rotation** | Enable automatic rotation for database credentials |
| **KMS keys** | Use customer-managed keys for audit control |
| **Least privilege** | Grant specific secret/parameter access, not wildcard |
