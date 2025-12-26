# Elastic Container Registry (ECR)

## Repository Management

### Create Repository
```bash
# Basic repository with scanning and immutability
aws ecr create-repository \
    --repository-name my-app/backend \
    --image-scanning-configuration scanOnPush=true \
    --image-tag-mutability IMMUTABLE

# With KMS encryption
aws ecr create-repository \
    --repository-name my-app/backend \
    --image-scanning-configuration scanOnPush=true \
    --image-tag-mutability IMMUTABLE \
    --encryption-configuration encryptionType=KMS,kmsKey=alias/ecr-key

# With tags
aws ecr create-repository \
    --repository-name my-app/backend \
    --tags Key=Environment,Value=Production Key=Team,Value=Backend
```

### List and Describe Repositories
```bash
# List all repositories
aws ecr describe-repositories

# Describe specific repository
aws ecr describe-repositories --repository-names my-app/backend

# Get repository URI
aws ecr describe-repositories \
    --repository-names my-app/backend \
    --query 'repositories[0].repositoryUri' \
    --output text
```

### Delete Repository
```bash
# Delete empty repository
aws ecr delete-repository --repository-name my-app/backend

# Force delete (removes all images)
aws ecr delete-repository --repository-name my-app/backend --force
```

## Authentication

### Docker Login
```bash
# Get login password and authenticate Docker
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# For a specific profile
aws ecr get-login-password --region us-east-1 --profile production | \
    docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

### Get Authorization Token Details
```bash
# Get base64-encoded token (valid 12 hours)
aws ecr get-authorization-token \
    --query 'authorizationData[0].authorizationToken' \
    --output text | base64 -d

# Get expiration time
aws ecr get-authorization-token \
    --query 'authorizationData[0].expiresAt'
```

## Push/Pull Workflows

### Build and Push Image
```bash
# Set variables
ACCOUNT_ID=123456789012
REGION=us-east-1
REPO_NAME=my-app/backend
TAG=v1.2.0

# Authenticate
aws ecr get-login-password --region $REGION | \
    docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

# Build image
docker build -t $REPO_NAME:$TAG .

# Tag for ECR
docker tag $REPO_NAME:$TAG $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO_NAME:$TAG

# Push
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO_NAME:$TAG

# Also tag as latest
docker tag $REPO_NAME:$TAG $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO_NAME:latest
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO_NAME:latest
```

### Pull Image
```bash
# Authenticate first
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull image
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app/backend:v1.2.0
```

## Image Management

### List Images
```bash
# List all images in repository
aws ecr list-images --repository-name my-app/backend

# List with details
aws ecr describe-images --repository-name my-app/backend

# List images sorted by push date
aws ecr describe-images \
    --repository-name my-app/backend \
    --query 'sort_by(imageDetails, &imagePushedAt)[*].{Tag:imageTags[0],Pushed:imagePushedAt,Size:imageSizeInBytes}'

# Find untagged images
aws ecr describe-images \
    --repository-name my-app/backend \
    --filter tagStatus=UNTAGGED
```

### Delete Images
```bash
# Delete by tag
aws ecr batch-delete-image \
    --repository-name my-app/backend \
    --image-ids imageTag=v1.0.0

# Delete by digest
aws ecr batch-delete-image \
    --repository-name my-app/backend \
    --image-ids imageDigest=sha256:abc123...

# Delete multiple images
aws ecr batch-delete-image \
    --repository-name my-app/backend \
    --image-ids imageTag=v1.0.0 imageTag=v1.0.1 imageTag=v1.0.2

# Delete all untagged images
aws ecr describe-images \
    --repository-name my-app/backend \
    --filter tagStatus=UNTAGGED \
    --query 'imageDetails[*].imageDigest' \
    --output text | \
    xargs -I {} aws ecr batch-delete-image \
        --repository-name my-app/backend \
        --image-ids imageDigest={}
```

### Get Image Details
```bash
# Get specific image by tag
aws ecr describe-images \
    --repository-name my-app/backend \
    --image-ids imageTag=v1.2.0

# Get image manifest
aws ecr batch-get-image \
    --repository-name my-app/backend \
    --image-ids imageTag=v1.2.0 \
    --query 'images[0].imageManifest' \
    --output text
```

## Image Scanning

### Basic Scanning (On-Push)
```bash
# Enable scan on push for repository
aws ecr put-image-scanning-configuration \
    --repository-name my-app/backend \
    --image-scanning-configuration scanOnPush=true
```

### On-Demand Scanning
```bash
# Start manual scan
aws ecr start-image-scan \
    --repository-name my-app/backend \
    --image-id imageTag=v1.2.0

# Wait for scan to complete
aws ecr wait image-scan-complete \
    --repository-name my-app/backend \
    --image-id imageTag=v1.2.0
```

### Get Scan Findings
```bash
# Get scan results
aws ecr describe-image-scan-findings \
    --repository-name my-app/backend \
    --image-id imageTag=v1.2.0

# Get only critical and high vulnerabilities
aws ecr describe-image-scan-findings \
    --repository-name my-app/backend \
    --image-id imageTag=v1.2.0 \
    --query 'imageScanFindings.findings[?severity==`CRITICAL` || severity==`HIGH`]'

# Get vulnerability counts by severity
aws ecr describe-image-scan-findings \
    --repository-name my-app/backend \
    --image-id imageTag=v1.2.0 \
    --query 'imageScanFindings.findingSeverityCounts'
```

### Enhanced Scanning (Amazon Inspector)
```bash
# Enable enhanced scanning at registry level
aws ecr put-registry-scanning-configuration \
    --scan-type ENHANCED \
    --rules '[{"repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}],"scanFrequency":"SCAN_ON_PUSH"}]'

# Enable continuous scanning for specific repositories
aws ecr put-registry-scanning-configuration \
    --scan-type ENHANCED \
    --rules '[{"repositoryFilters":[{"filter":"prod/*","filterType":"WILDCARD"}],"scanFrequency":"CONTINUOUS_SCAN"}]'

# Get registry scanning configuration
aws ecr get-registry-scanning-configuration
```

## Lifecycle Policies

### Create Lifecycle Policy
```bash
aws ecr put-lifecycle-policy \
    --repository-name my-app/backend \
    --lifecycle-policy-text file://lifecycle-policy.json
```

**lifecycle-policy.json (Keep tagged, expire old untagged):**
```json
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Expire untagged images older than 1 day",
            "selection": {
                "tagStatus": "untagged",
                "countType": "sinceImagePushed",
                "countUnit": "days",
                "countNumber": 1
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 2,
            "description": "Keep only 10 dev images",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["dev-", "feature-"],
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 3,
            "description": "Keep last 50 production images",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["v", "release-"],
                "countType": "imageCountMoreThan",
                "countNumber": 50
            },
            "action": {
                "type": "expire"
            }
        }
    ]
}
```

### Manage Lifecycle Policies
```bash
# Get lifecycle policy
aws ecr get-lifecycle-policy --repository-name my-app/backend

# Preview lifecycle policy (dry run)
aws ecr get-lifecycle-policy-preview \
    --repository-name my-app/backend \
    --lifecycle-policy-text file://lifecycle-policy.json

# Delete lifecycle policy
aws ecr delete-lifecycle-policy --repository-name my-app/backend
```

## Repository Policies (Cross-Account Access)

### Set Repository Policy
```bash
aws ecr set-repository-policy \
    --repository-name my-app/backend \
    --policy-text file://repo-policy.json
```

**repo-policy.json (Cross-account pull access):**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCrossAccountPull",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::111111111111:root",
                    "arn:aws:iam::222222222222:root"
                ]
            },
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ]
        }
    ]
}
```

**repo-policy.json (Allow specific role):**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowECSTaskRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111111111111:role/ECSTaskRole"
            },
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ]
        }
    ]
}
```

### Manage Repository Policies
```bash
# Get current policy
aws ecr get-repository-policy --repository-name my-app/backend

# Delete policy
aws ecr delete-repository-policy --repository-name my-app/backend
```

## Cross-Region Replication

### Configure Registry Replication
```bash
aws ecr put-replication-configuration \
    --replication-configuration file://replication-config.json
```

**replication-config.json (Single region):**
```json
{
    "rules": [
        {
            "destinations": [
                {
                    "region": "eu-west-1",
                    "registryId": "123456789012"
                }
            ],
            "repositoryFilters": [
                {
                    "filter": "prod/",
                    "filterType": "PREFIX_MATCH"
                }
            ]
        }
    ]
}
```

**replication-config.json (Multi-region):**
```json
{
    "rules": [
        {
            "destinations": [
                {"region": "eu-west-1", "registryId": "123456789012"},
                {"region": "ap-southeast-1", "registryId": "123456789012"},
                {"region": "us-west-2", "registryId": "123456789012"}
            ],
            "repositoryFilters": [
                {
                    "filter": "prod/",
                    "filterType": "PREFIX_MATCH"
                }
            ]
        }
    ]
}
```

### Cross-Account Replication
```json
{
    "rules": [
        {
            "destinations": [
                {
                    "region": "us-east-1",
                    "registryId": "999999999999"
                }
            ]
        }
    ]
}
```

### Get Replication Configuration
```bash
aws ecr describe-registry
```

## Pull Through Cache

Cache images from external registries (Docker Hub, GitHub, Quay, etc.).

### Create Pull Through Cache Rule
```bash
# Docker Hub (public)
aws ecr create-pull-through-cache-rule \
    --ecr-repository-prefix docker-hub \
    --upstream-registry-url registry-1.docker.io

# GitHub Container Registry
aws ecr create-pull-through-cache-rule \
    --ecr-repository-prefix ghcr \
    --upstream-registry-url ghcr.io

# Quay.io
aws ecr create-pull-through-cache-rule \
    --ecr-repository-prefix quay \
    --upstream-registry-url quay.io

# With credentials (for private registries)
aws ecr create-pull-through-cache-rule \
    --ecr-repository-prefix docker-hub \
    --upstream-registry-url registry-1.docker.io \
    --credential-arn arn:aws:secretsmanager:us-east-1:123456789012:secret:dockerhub-creds
```

### Use Cached Images
```bash
# Instead of: docker pull nginx:latest
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-hub/library/nginx:latest

# Instead of: docker pull ghcr.io/owner/image:tag
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/ghcr/owner/image:tag
```

### Manage Pull Through Cache
```bash
# List rules
aws ecr describe-pull-through-cache-rules

# Delete rule
aws ecr delete-pull-through-cache-rule --ecr-repository-prefix docker-hub
```

## Registry Settings

### Registry Policy
```bash
# Set registry policy (for replication permissions)
aws ecr put-registry-policy --policy-text file://registry-policy.json
```

**registry-policy.json:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowReplicationFromAccount",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111111111111:root"
            },
            "Action": [
                "ecr:CreateRepository",
                "ecr:ReplicateImage"
            ],
            "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/*"
        }
    ]
}
```

### Get Registry Settings
```bash
# Describe registry
aws ecr describe-registry

# Get registry policy
aws ecr get-registry-policy
```

## Useful Queries

```bash
# Get all repository URIs
aws ecr describe-repositories \
    --query 'repositories[*].repositoryUri' \
    --output table

# Find largest images
aws ecr describe-images \
    --repository-name my-app/backend \
    --query 'sort_by(imageDetails, &imageSizeInBytes)[-5:].{Tag:imageTags[0],SizeMB:imageSizeInBytes}' \
    --output table

# Count images per repository
for repo in $(aws ecr describe-repositories --query 'repositories[*].repositoryName' --output text); do
    count=$(aws ecr list-images --repository-name $repo --query 'length(imageIds)' --output text)
    echo "$repo: $count images"
done

# Find images with vulnerabilities
aws ecr describe-images \
    --repository-name my-app/backend \
    --query 'imageDetails[?imageScanFindingsSummary.findingSeverityCounts.CRITICAL > `0`].imageTags'
```

## Image Tagging Best Practices

| Strategy | Example | Use Case |
|:---------|:--------|:---------|
| **Semantic versioning** | `v1.2.3` | Production releases |
| **Git SHA** | `abc1234` | Traceability to commits |
| **Build number** | `build-456` | CI/CD pipelines |
| **Combined** | `v1.2.3-abc1234` | Best of both worlds |
| **Environment** | `prod-v1.2.3` | Multi-environment |
| **Date-based** | `2024-01-15-abc1234` | Rolling deployments |

```bash
# Multi-tag strategy in CI/CD
TAG_VERSION="v1.2.3"
TAG_SHA=$(git rev-parse --short HEAD)
TAG_DATE=$(date +%Y%m%d)

docker tag app:latest $ECR_URI:$TAG_VERSION
docker tag app:latest $ECR_URI:$TAG_SHA
docker tag app:latest $ECR_URI:$TAG_VERSION-$TAG_SHA
docker tag app:latest $ECR_URI:latest

docker push $ECR_URI --all-tags
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Immutable tags** | Enable for release tags to prevent overwrites |
| **Scan on push** | Enable basic scanning for all repositories |
| **Enhanced scanning** | Use for production (continuous vulnerability monitoring) |
| **Lifecycle policies** | Always set policies to clean up old/untagged images |
| **Cross-region replication** | Replicate production images to DR regions |
| **Pull through cache** | Cache external images to reduce Docker Hub rate limits |
| **Repository naming** | Use namespaces like `team/app` or `env/app` |
| **Tag strategy** | Combine semantic versions with Git SHAs |
| **Cross-account access** | Use repository policies, not IAM policies |
| **KMS encryption** | Use customer-managed keys for sensitive images |
