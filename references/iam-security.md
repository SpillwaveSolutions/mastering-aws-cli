# IAM & Security

## Contents

- [Users & Credentials](#users--credentials)
- [Roles](#roles)
  - [Cross-Account Role Setup Checklist](#cross-account-role-setup-checklist)
  - [GitHub Actions OIDC Setup Checklist](#github-actions-oidc-setup-checklist)
- [Policies](#policies)
- [STS (Security Token Service)](#sts-security-token-service)
- [Instance Profiles](#instance-profiles)
- [Permission Boundaries](#permission-boundaries)
- [IAM Access Analyzer](#iam-access-analyzer)
- [Service-Linked Roles](#service-linked-roles)
- [OIDC Providers](#oidc-providers)
- [Useful Queries](#useful-queries)
- [Best Practices](#best-practices)

---

## Users & Credentials

### User Management
```bash
# Create user with path-based organization
aws iam create-user --user-name developer --path /engineering/

# Create user with tags
aws iam create-user \
    --user-name developer \
    --tags Key=Team,Value=Backend Key=Environment,Value=Production

# List users
aws iam list-users
aws iam list-users --path-prefix /engineering/

# Get user details
aws iam get-user --user-name developer

# Delete user (must remove from groups and delete keys first)
aws iam delete-user --user-name developer
```

### Access Keys
```bash
# Create access key (SAVE OUTPUT - shown only once!)
aws iam create-access-key --user-name developer

# List access keys
aws iam list-access-keys --user-name developer

# Rotate key (create new, then deactivate old)
aws iam update-access-key \
    --user-name developer \
    --access-key-id AKIAIOSFODNN7EXAMPLE \
    --status Inactive

# Delete old key
aws iam delete-access-key \
    --user-name developer \
    --access-key-id AKIAIOSFODNN7EXAMPLE

# Get last used info
aws iam get-access-key-last-used --access-key-id AKIAIOSFODNN7EXAMPLE
```

### MFA Management
```bash
# Create virtual MFA device
aws iam create-virtual-mfa-device \
    --virtual-mfa-device-name developer-mfa \
    --outfile QRCode.png \
    --bootstrap-method QRCodePNG

# Enable MFA for user (requires two consecutive codes)
aws iam enable-mfa-device \
    --user-name developer \
    --serial-number arn:aws:iam::123456789012:mfa/developer-mfa \
    --authentication-code1 123456 \
    --authentication-code2 789012

# List MFA devices
aws iam list-mfa-devices --user-name developer

# Deactivate MFA
aws iam deactivate-mfa-device \
    --user-name developer \
    --serial-number arn:aws:iam::123456789012:mfa/developer-mfa
```

### Groups
```bash
# Create group
aws iam create-group --group-name Developers

# Add user to group
aws iam add-user-to-group --user-name developer --group-name Developers

# List groups for user
aws iam list-groups-for-user --user-name developer

# Remove user from group
aws iam remove-user-from-group --user-name developer --group-name Developers
```

## Roles

### Create Role with Trust Policy

**Basic role creation:**
```bash
aws iam create-role \
    --role-name MyRole \
    --assume-role-policy-document file://trust-policy.json \
    --description "Role for service X" \
    --max-session-duration 3600
```

### Trust Policy Examples

**EC2 Service Role:**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
```

**Lambda Service Role:**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "lambda.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
```

**ECS Tasks Role:**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ecs-tasks.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
```

**Cross-Account Role (with external ID):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::OTHER_ACCOUNT_ID:root"},
        "Action": "sts:AssumeRole",
        "Condition": {
            "StringEquals": {"sts:ExternalId": "UniqueSecretId123"}
        }
    }]
}
```

**GitHub OIDC (for GitHub Actions):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
            "StringEquals": {
                "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
            },
            "StringLike": {
                "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
            }
        }
    }]
}
```

**EKS OIDC (IRSA - IAM Roles for Service Accounts):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
            "StringEquals": {
                "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:default:my-service-account",
                "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com"
            }
        }
    }]
}
```

### Cross-Account Role Setup Checklist

```
[ ] 1. Create trust-policy.json with target account ARN and optional external ID
[ ] 2. Create role: aws iam create-role --role-name CrossAccountRole --assume-role-policy-document file://trust-policy.json
[ ] 3. Attach policy: aws iam attach-role-policy --role-name CrossAccountRole --policy-arn <policy-arn>
[ ] 4. Verify role: aws iam get-role --role-name CrossAccountRole
[ ] 5. Test from target account: aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/CrossAccountRole --role-session-name test
[ ] 6. Verify assumed identity: aws sts get-caller-identity
```

### GitHub Actions OIDC Setup Checklist

```
[ ] 1. Create OIDC provider (one-time per account):
      aws iam create-open-id-connect-provider \
          --url "https://token.actions.githubusercontent.com" \
          --client-id-list "sts.amazonaws.com" \
          --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"
[ ] 2. Create trust-policy.json with GitHub OIDC principal (see template above)
[ ] 3. Create role: aws iam create-role --role-name GitHubActionsRole --assume-role-policy-document file://trust-policy.json
[ ] 4. Attach permissions: aws iam attach-role-policy --role-name GitHubActionsRole --policy-arn <policy-arn>
[ ] 5. Verify provider: aws iam list-open-id-connect-providers
[ ] 6. Test in GitHub Actions workflow with aws-actions/configure-aws-credentials
```

### Role Management
```bash
# List roles
aws iam list-roles
aws iam list-roles --path-prefix /service-roles/

# Get role details
aws iam get-role --role-name MyRole

# Update trust policy
aws iam update-assume-role-policy \
    --role-name MyRole \
    --policy-document file://new-trust-policy.json

# Delete role (must detach policies first)
aws iam delete-role --role-name MyRole
```

## Policies

### Create Managed Policy
```bash
aws iam create-policy \
    --policy-name S3ReadAccess \
    --policy-document file://policy.json \
    --description "Read-only access to S3"
```

**Policy Example (S3 Read-Only):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "s3:GetObject",
            "s3:ListBucket"
        ],
        "Resource": [
            "arn:aws:s3:::my-bucket",
            "arn:aws:s3:::my-bucket/*"
        ]
    }]
}
```

**Policy Example (DynamoDB CRUD):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "dynamodb:GetItem",
            "dynamodb:PutItem",
            "dynamodb:UpdateItem",
            "dynamodb:DeleteItem",
            "dynamodb:Query",
            "dynamodb:Scan"
        ],
        "Resource": "arn:aws:s3:us-east-1:123456789012:table/MyTable"
    }]
}
```

### Attach Policies
```bash
# Attach managed policy to role
aws iam attach-role-policy \
    --role-name MyRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Attach to user
aws iam attach-user-policy \
    --user-name developer \
    --policy-arn arn:aws:iam::123456789012:policy/S3ReadAccess

# Attach to group
aws iam attach-group-policy \
    --group-name Developers \
    --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# List attached policies
aws iam list-attached-role-policies --role-name MyRole
aws iam list-attached-user-policies --user-name developer
```

### Inline Policies
```bash
# Put inline policy on role
aws iam put-role-policy \
    --role-name MyRole \
    --policy-name InlineS3Policy \
    --policy-document file://inline-policy.json

# Get inline policy
aws iam get-role-policy --role-name MyRole --policy-name InlineS3Policy

# Delete inline policy
aws iam delete-role-policy --role-name MyRole --policy-name InlineS3Policy
```

### Policy Versioning
```bash
# Create new version
aws iam create-policy-version \
    --policy-arn arn:aws:iam::123456789012:policy/S3ReadAccess \
    --policy-document file://updated-policy.json \
    --set-as-default

# List versions
aws iam list-policy-versions \
    --policy-arn arn:aws:iam::123456789012:policy/S3ReadAccess

# Set default version
aws iam set-default-policy-version \
    --policy-arn arn:aws:iam::123456789012:policy/S3ReadAccess \
    --version-id v2
```

## STS (Security Token Service)

### Assume Role
```bash
# Basic assume role
aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/CrossAccountRole \
    --role-session-name MySession \
    --duration-seconds 3600

# With MFA
aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/MFAProtectedRole \
    --role-session-name MFASession \
    --serial-number arn:aws:iam::123456789012:mfa/myuser \
    --token-code 123456

# With external ID (for cross-account)
aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/VendorRole \
    --role-session-name VendorSession \
    --external-id UniqueSecretId123
```

### Use Assumed Credentials
```bash
# Capture and export credentials
CREDS=$(aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/AdminRole \
    --role-session-name AdminSession \
    --output json)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')

# Verify new identity
aws sts get-caller-identity
```

### Identity Verification
```bash
# Get current identity
aws sts get-caller-identity
# Returns: Account, UserId, Arn

# Get session token (for MFA)
aws sts get-session-token \
    --serial-number arn:aws:iam::123456789012:mfa/myuser \
    --token-code 123456 \
    --duration-seconds 3600
```

### Web Identity (OIDC)
```bash
# Assume role with web identity token
aws sts assume-role-with-web-identity \
    --role-arn arn:aws:iam::123456789012:role/FederatedRole \
    --role-session-name WebSession \
    --web-identity-token file://token.jwt
```

## Instance Profiles

```bash
# Create instance profile
aws iam create-instance-profile --instance-profile-name WebServerProfile

# Add role to instance profile
aws iam add-role-to-instance-profile \
    --instance-profile-name WebServerProfile \
    --role-name EC2-S3-Access

# Associate with EC2 instance
aws ec2 associate-iam-instance-profile \
    --instance-id i-1234567890abcdef0 \
    --iam-instance-profile Name=WebServerProfile

# Replace instance profile
aws ec2 replace-iam-instance-profile-association \
    --association-id iip-assoc-0e7736511a163c209 \
    --iam-instance-profile Name=NewProfile

# List instance profiles
aws iam list-instance-profiles

# Get instance profile for role
aws iam list-instance-profiles-for-role --role-name EC2-S3-Access
```

## Permission Boundaries

```bash
# Set permission boundary on role
aws iam put-role-permissions-boundary \
    --role-name DeveloperRole \
    --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary

# Set on user
aws iam put-user-permissions-boundary \
    --user-name developer \
    --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary

# Remove boundary
aws iam delete-role-permissions-boundary --role-name DeveloperRole
```

**Permission Boundary Policy Example:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "dynamodb:*",
                "lambda:*",
                "logs:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Deny",
            "Action": [
                "iam:*",
                "organizations:*",
                "account:*"
            ],
            "Resource": "*"
        }
    ]
}
```

## IAM Access Analyzer

```bash
# Create analyzer (account-level)
aws accessanalyzer create-analyzer \
    --analyzer-name ExternalAccessAnalyzer \
    --type ACCOUNT

# Create analyzer (organization-level)
aws accessanalyzer create-analyzer \
    --analyzer-name OrgAnalyzer \
    --type ORGANIZATION

# List analyzers
aws accessanalyzer list-analyzers

# List findings
aws accessanalyzer list-findings \
    --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/ExternalAccessAnalyzer

# Filter findings
aws accessanalyzer list-findings \
    --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/ExternalAccessAnalyzer \
    --filter '{"resourceType": {"eq": ["AWS::S3::Bucket"]}, "status": {"eq": ["ACTIVE"]}}'

# Get finding details
aws accessanalyzer get-finding \
    --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/ExternalAccessAnalyzer \
    --id finding-id

# Archive finding (mark as reviewed)
aws accessanalyzer update-findings \
    --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/ExternalAccessAnalyzer \
    --ids finding-id \
    --status ARCHIVED
```

## Service-Linked Roles

```bash
# Create service-linked role
aws iam create-service-linked-role \
    --aws-service-name elasticloadbalancing.amazonaws.com

# List service-linked roles
aws iam list-roles --path-prefix /aws-service-role/

# Get service-linked role
aws iam get-role --role-name AWSServiceRoleForElasticLoadBalancing

# Delete service-linked role (if not in use)
aws iam delete-service-linked-role \
    --role-name AWSServiceRoleForElasticLoadBalancing
```

## OIDC Providers

```bash
# Create OIDC provider (GitHub Actions)
aws iam create-open-id-connect-provider \
    --url "https://token.actions.githubusercontent.com" \
    --client-id-list "sts.amazonaws.com" \
    --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"

# Get OIDC provider details
aws iam get-open-id-connect-provider \
    --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com

# List OIDC providers
aws iam list-open-id-connect-providers

# Delete OIDC provider
aws iam delete-open-id-connect-provider \
    --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com
```

## Useful Queries

```bash
# Find unused credentials (access keys not used in 90 days)
aws iam generate-credential-report
aws iam get-credential-report --output text --query 'Content' | base64 -d

# List all policies attached to a role
aws iam list-attached-role-policies --role-name MyRole
aws iam list-role-policies --role-name MyRole  # inline policies

# Get effective permissions for identity
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:user/developer \
    --action-names s3:GetObject s3:PutObject \
    --resource-arns arn:aws:s3:::my-bucket/*

# Find roles that can be assumed by a service
aws iam list-roles \
    --query 'Roles[?contains(AssumeRolePolicyDocument.Statement[0].Principal.Service, `lambda.amazonaws.com`)].RoleName'
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Least privilege** | Start with minimal permissions, add as needed |
| **Use roles** | Prefer roles over long-lived access keys for compute |
| **MFA** | Require MFA for sensitive operations |
| **Permission boundaries** | Limit maximum permissions for delegated admins |
| **External IDs** | Use external IDs for cross-account roles with third parties |
| **Rotate keys** | Rotate access keys every 90 days maximum |
| **Access Analyzer** | Enable to detect unintended external access |
| **Service-linked roles** | Use AWS-managed roles for services when available |
| **Audit** | Use CloudTrail to monitor IAM actions |
