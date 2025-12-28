# Bastion & Tunneling

## SSM Session Manager

The modern, secure replacement for SSH bastion hosts. No open inbound ports required.

### Prerequisites
```bash
# Install Session Manager plugin
# macOS
brew install --cask session-manager-plugin

# Verify installation
session-manager-plugin

# Required VPC endpoints (for private subnets without NAT)
# - ssm.region.amazonaws.com
# - ssmmessages.region.amazonaws.com
# - ec2messages.region.amazonaws.com
```

### Interactive Shell
```bash
# Start session to EC2 instance
aws ssm start-session --target i-0123456789abcdef0

# Start session with specific region
aws ssm start-session \
    --target i-0123456789abcdef0 \
    --region us-west-2

# Start session to on-premises managed instance
aws ssm start-session --target mi-0123456789abcdef0
```

### Port Forwarding (Local)
Forward a local port to a port on the remote instance.

```bash
# Forward local port to remote port
aws ssm start-session \
    --target i-0123456789abcdef0 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'

# RDP access (Windows)
aws ssm start-session \
    --target i-windows-instance \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["3389"],"localPortNumber":["33389"]}'
# Then: mstsc /v:localhost:33389

# VNC access
aws ssm start-session \
    --target i-linux-instance \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["5901"],"localPortNumber":["5901"]}'
```

### Remote Host Port Forwarding
Access resources *through* a bastion instance (jump host pattern).

```bash
# Access RDS through bastion
aws ssm start-session \
    --target i-bastion-instance \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["mydb.cluster-xyz.us-east-1.rds.amazonaws.com"],
        "portNumber":["5432"],
        "localPortNumber":["5432"]
    }'
# Now connect: psql -h localhost -p 5432 -U admin mydb

# Access Aurora MySQL
aws ssm start-session \
    --target i-bastion-instance \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["aurora-cluster.cluster-xyz.us-east-1.rds.amazonaws.com"],
        "portNumber":["3306"],
        "localPortNumber":["3306"]
    }'
# Now connect: mysql -h 127.0.0.1 -P 3306 -u admin -p

# Access ElastiCache Redis
aws ssm start-session \
    --target i-bastion-instance \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["redis-cluster.xyz.cache.amazonaws.com"],
        "portNumber":["6379"],
        "localPortNumber":["6379"]
    }'
# Now connect: redis-cli -h localhost -p 6379

# Access OpenSearch
aws ssm start-session \
    --target i-bastion-instance \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["search-domain.us-east-1.es.amazonaws.com"],
        "portNumber":["443"],
        "localPortNumber":["9200"]
    }'
# Now: curl https://localhost:9200

# Access internal ALB
aws ssm start-session \
    --target i-bastion-instance \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["internal-alb-123.us-east-1.elb.amazonaws.com"],
        "portNumber":["443"],
        "localPortNumber":["8443"]
    }'
```

## EKS Private Cluster Access

Access private EKS clusters through SSM without exposing the API server publicly.

### Update Kubeconfig for Private Cluster
```bash
# Get cluster info
aws eks describe-cluster --name my-cluster \
    --query 'cluster.{endpoint:endpoint,ca:certificateAuthority.data}'

# Update kubeconfig (will fail if cluster is private)
aws eks update-kubeconfig --name my-cluster --region us-east-1
```

### Kubectl via SSM Port Forward
```bash
# 1. Get private endpoint
ENDPOINT=$(aws eks describe-cluster --name my-cluster \
    --query 'cluster.endpoint' --output text)
# Example: https://ABC123.gr7.us-east-1.eks.amazonaws.com

# 2. Extract hostname
EKS_HOST=$(echo $ENDPOINT | sed 's|https://||')

# 3. Start port forwarding to EKS API (port 443)
aws ssm start-session \
    --target i-bastion-in-eks-vpc \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{
        \"host\":[\"$EKS_HOST\"],
        \"portNumber\":[\"443\"],
        \"localPortNumber\":[\"6443\"]
    }"

# 4. In another terminal, modify kubeconfig
# Add to ~/.kube/config:
# clusters:
# - cluster:
#     server: https://127.0.0.1:6443
#     certificate-authority-data: <base64-ca-from-cluster>
#   name: my-cluster

# 5. Run kubectl
kubectl get nodes
kubectl get pods -A
```

### Alternative: SSM Document for EKS
```bash
# Create custom SSM document for kubectl
aws ssm create-document \
    --name "EKS-Kubectl" \
    --document-type "Session" \
    --content '{
        "schemaVersion": "1.0",
        "description": "Run kubectl commands via SSM",
        "sessionType": "InteractiveCommands",
        "inputs": {
            "runAsEnabled": true,
            "runAsDefaultUser": "ec2-user"
        }
    }'

# Run kubectl commands directly on bastion
aws ssm start-session \
    --target i-bastion-with-kubectl \
    --document-name EKS-Kubectl
```

### Using SSM as SSH Proxy for EKS
```bash
# ~/.ssh/config addition for SSM proxy
Host eks-bastion
    HostName i-bastion-instance-id
    User ec2-user
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"

# SSH to bastion, then kubectl
ssh eks-bastion
kubectl get nodes
```

## Aurora/RDS Database Access

### Direct Port Forwarding
```bash
# PostgreSQL (Aurora/RDS)
aws ssm start-session \
    --target i-bastion \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["prod-db.cluster-xyz.us-east-1.rds.amazonaws.com"],
        "portNumber":["5432"],
        "localPortNumber":["5432"]
    }'

# Connect with psql
psql "host=localhost port=5432 dbname=mydb user=admin sslmode=require"

# Connect with DBeaver/pgAdmin
# Host: localhost, Port: 5432, SSL: Required

# MySQL/Aurora MySQL
aws ssm start-session \
    --target i-bastion \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["prod-db.cluster-xyz.us-east-1.rds.amazonaws.com"],
        "portNumber":["3306"],
        "localPortNumber":["3306"]
    }'

# Connect with mysql client
mysql -h 127.0.0.1 -P 3306 -u admin -p --ssl-mode=REQUIRED
```

### JDBC Connection Through Tunnel
```bash
# Start tunnel in background
aws ssm start-session \
    --target i-bastion \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{
        "host":["aurora.cluster-xyz.us-east-1.rds.amazonaws.com"],
        "portNumber":["5432"],
        "localPortNumber":["5432"]
    }' &

# JDBC URL (use in application/IDE)
# PostgreSQL: jdbc:postgresql://localhost:5432/mydb?ssl=true
# MySQL: jdbc:mysql://localhost:3306/mydb?useSSL=true
```

### Shell Script for DB Tunnel
```bash
#!/bin/bash
# db-tunnel.sh - Start database tunnel

DB_HOST="${1:-prod-db.cluster-xyz.us-east-1.rds.amazonaws.com}"
DB_PORT="${2:-5432}"
LOCAL_PORT="${3:-5432}"
BASTION_ID="${BASTION_INSTANCE_ID:-i-0123456789abcdef0}"

echo "Starting tunnel to $DB_HOST:$DB_PORT on localhost:$LOCAL_PORT"
aws ssm start-session \
    --target "$BASTION_ID" \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{
        \"host\":[\"$DB_HOST\"],
        \"portNumber\":[\"$DB_PORT\"],
        \"localPortNumber\":[\"$LOCAL_PORT\"]
    }"
```

## SSH Access Methods

### EC2 Instance Connect
Push a temporary SSH key (valid 60 seconds).

```bash
# Push public key
aws ec2-instance-connect send-ssh-public-key \
    --instance-id i-0123456789abcdef0 \
    --instance-os-user ec2-user \
    --ssh-public-key file://~/.ssh/id_rsa.pub

# SSH within 60 seconds
ssh ec2-user@<public-ip>

# One-liner with AWS CLI
aws ec2-instance-connect send-ssh-public-key \
    --instance-id i-123 \
    --instance-os-user ec2-user \
    --ssh-public-key file://~/.ssh/id_rsa.pub && \
    ssh ec2-user@$(aws ec2 describe-instances --instance-ids i-123 \
        --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
```

### SSM as SSH Proxy
Configure SSH to tunnel through SSM (no public IP needed).

```bash
# ~/.ssh/config
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem

Host bastion-prod
    HostName i-0123456789abcdef0
    User ec2-user
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"

# Usage
ssh i-0123456789abcdef0
ssh bastion-prod

# SCP through SSM
scp -o ProxyCommand="aws ssm start-session --target i-123 --document-name AWS-StartSSHSession --parameters portNumber=22" \
    myfile.txt ec2-user@i-123:/home/ec2-user/
```

### Multi-Hop SSH Through Bastion
```bash
# ~/.ssh/config for jump host pattern
Host bastion
    HostName i-bastion-id
    User ec2-user
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"

Host private-server
    HostName 10.0.1.100
    User ec2-user
    ProxyJump bastion
    IdentityFile ~/.ssh/private-key.pem

# Usage
ssh private-server
```

## Run Command (Remote Execution)

### Execute Commands
```bash
# Run command on single instance
aws ssm send-command \
    --instance-ids i-0123456789abcdef0 \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["df -h","free -m"]'

# Run on multiple instances
aws ssm send-command \
    --instance-ids i-123 i-456 i-789 \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["yum update -y"]'

# Run on instances by tag
aws ssm send-command \
    --targets Key=tag:Environment,Values=Production \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["systemctl restart nginx"]'

# Run with timeout
aws ssm send-command \
    --instance-ids i-123 \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["./long-running-script.sh"]' \
    --timeout-seconds 3600
```

### Get Command Output
```bash
# Get command invocation results
COMMAND_ID=$(aws ssm send-command \
    --instance-ids i-123 \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["hostname"]' \
    --query 'Command.CommandId' \
    --output text)

# Wait and get output
aws ssm get-command-invocation \
    --command-id $COMMAND_ID \
    --instance-id i-123 \
    --query '{Status:Status,Output:StandardOutputContent}'

# List all command invocations
aws ssm list-command-invocations \
    --command-id $COMMAND_ID \
    --details
```

### Windows Commands
```bash
# PowerShell command
aws ssm send-command \
    --instance-ids i-windows-123 \
    --document-name "AWS-RunPowerShellScript" \
    --parameters 'commands=["Get-Process | Sort-Object CPU -Descending | Select-Object -First 10"]'

# Install Windows feature
aws ssm send-command \
    --instance-ids i-windows-123 \
    --document-name "AWS-RunPowerShellScript" \
    --parameters 'commands=["Install-WindowsFeature -Name Web-Server -IncludeManagementTools"]'
```

## Session Manager Preferences

### Configure Logging
```bash
# Create preferences document
aws ssm update-document \
    --name "SSM-SessionManagerRunShell" \
    --document-version "\$LATEST" \
    --content '{
        "schemaVersion": "1.0",
        "description": "Session Manager Preferences",
        "sessionType": "Standard_Stream",
        "inputs": {
            "s3BucketName": "my-session-logs-bucket",
            "s3KeyPrefix": "session-logs/",
            "s3EncryptionEnabled": true,
            "cloudWatchLogGroupName": "/aws/ssm/session-logs",
            "cloudWatchEncryptionEnabled": true,
            "kmsKeyId": "alias/session-manager-key",
            "runAsEnabled": true,
            "runAsDefaultUser": "ssm-user",
            "idleSessionTimeout": "20",
            "shellProfile": {
                "linux": "cd ~ && bash",
                "windows": ""
            }
        }
    }'
```

### Custom Session Documents
```bash
# Create interactive command document
aws ssm create-document \
    --name "Custom-InteractiveSession" \
    --document-type "Session" \
    --content '{
        "schemaVersion": "1.0",
        "description": "Custom interactive session",
        "sessionType": "InteractiveCommands",
        "inputs": {
            "runAsEnabled": true,
            "runAsDefaultUser": "admin"
        }
    }'
```

## Automation Patterns

### Dynamic Bastion Discovery
Find bastion instances by tag instead of hardcoding instance IDs.

```bash
# Find bastion by tag
BASTION_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Role,Values=bastion" \
              "Name=instance-state-name,Values=running" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)

# Verify bastion found
if [ "$BASTION_ID" = "None" ] || [ -z "$BASTION_ID" ]; then
    echo "Error: No running bastion instance found"
    exit 1
fi

# Use in SSM session
aws ssm start-session --target "$BASTION_ID"

# Find by multiple tags (environment + role)
BASTION_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Environment,Values=production" \
              "Name=tag:Role,Values=bastion" \
              "Name=instance-state-name,Values=running" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)
```

### Clean Shell Environment
Ensure SSO credentials aren't overridden by static environment variables.

```bash
# Clear any static credentials before SSO login
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN

# Now SSO login works correctly
export AWS_PROFILE=my-sso-profile
aws sso login

# Verify using SSO credentials (not static)
aws configure list
# Should show: profile my-sso-profile, not environment variables

# Full clean environment script
clean_aws_env() {
    unset AWS_ACCESS_KEY_ID
    unset AWS_SECRET_ACCESS_KEY
    unset AWS_SESSION_TOKEN
    unset AWS_SECURITY_TOKEN
    echo "Cleared static AWS credentials from environment"
}
```

### Credential Validation Patterns
Check if credentials are valid before proceeding.

```bash
# Validate credentials (returns 0 if valid, non-zero if expired/invalid)
validate_aws_creds() {
    if aws sts get-caller-identity &>/dev/null; then
        echo "✓ AWS credentials valid"
        aws sts get-caller-identity --query 'Arn' --output text
        return 0
    else
        echo "✗ AWS credentials invalid or expired"
        return 1
    fi
}

# Auto-login if credentials expired (SSO profiles)
ensure_aws_login() {
    if ! aws sts get-caller-identity &>/dev/null; then
        echo "Credentials expired, logging in..."
        aws sso login --profile "${AWS_PROFILE:-default}"
    fi
}

# Use in scripts
ensure_aws_login
aws s3 ls

# Check profile before operations
check_profile() {
    local current=$(aws configure list --query 'profile' --output text 2>/dev/null)
    echo "Current profile: ${AWS_PROFILE:-default}"
    echo "Account: $(aws sts get-caller-identity --query 'Account' --output text)"
    echo "ARN: $(aws sts get-caller-identity --query 'Arn' --output text)"
}
```

### Port Conflict Resolution
Handle "Address already in use" errors.

```bash
# Check what's using a port
lsof -i :5432

# Find and show process on port
check_port() {
    local port=$1
    local pid=$(lsof -ti :$port 2>/dev/null)
    if [ -n "$pid" ]; then
        echo "Port $port in use by PID $pid:"
        ps -p $pid -o pid,user,command
        return 1
    else
        echo "Port $port is available"
        return 0
    fi
}

# Kill process on port (use with caution)
free_port() {
    local port=$1
    local pid=$(lsof -ti :$port 2>/dev/null)
    if [ -n "$pid" ]; then
        echo "Killing process $pid on port $port"
        kill -9 $pid
        sleep 1
    fi
}

# Smart port selection (find available port)
find_available_port() {
    local start_port=${1:-5432}
    local port=$start_port
    while lsof -ti :$port &>/dev/null; do
        ((port++))
    done
    echo $port
}

# Example: auto-select port for DB tunnel
LOCAL_PORT=$(find_available_port 5432)
echo "Using port $LOCAL_PORT"
```

### Multiple Cluster Access
Work with multiple EKS clusters simultaneously.

```bash
# Each terminal: different cluster
# Terminal 1 (Dev)
export AWS_PROFILE=dev-admin
export KUBECONFIG=~/.kube/config-dev
./connect-cluster.sh dev-cluster

# Terminal 2 (Staging)
export AWS_PROFILE=staging-admin
export KUBECONFIG=~/.kube/config-staging
./connect-cluster.sh staging-cluster

# Terminal 3 (Prod - read only)
export AWS_PROFILE=prod-readonly
export KUBECONFIG=~/.kube/config-prod
./connect-cluster.sh prod-cluster

# Quick cluster context switch script
use_cluster() {
    local env=$1
    case $env in
        dev)
            export AWS_PROFILE=dev-admin
            export KUBECONFIG=~/.kube/config-dev
            ;;
        staging)
            export AWS_PROFILE=staging-admin
            export KUBECONFIG=~/.kube/config-staging
            ;;
        prod)
            export AWS_PROFILE=prod-readonly
            export KUBECONFIG=~/.kube/config-prod
            ;;
        *)
            echo "Unknown environment: $env"
            return 1
            ;;
    esac
    echo "Switched to $env environment"
    kubectl config current-context
}
```

### Session Cleanup on Exit
Ensure SSM sessions are cleaned up when shell exits.

```bash
#!/bin/bash
# ssm-connect.sh - Connect with automatic cleanup

SESSION_PID=""
SSM_LOG="/tmp/ssm-session-$$.log"

cleanup() {
    echo "Cleaning up SSM session..."
    [ -n "$SESSION_PID" ] && kill $SESSION_PID 2>/dev/null
    rm -f "$SSM_LOG"
    echo "Session terminated"
}

# Register cleanup on exit
trap cleanup EXIT INT TERM

# Start SSM port forwarding in background
aws ssm start-session \
    --target "$BASTION_ID" \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{
        \"host\":[\"$REMOTE_HOST\"],
        \"portNumber\":[\"$REMOTE_PORT\"],
        \"localPortNumber\":[\"$LOCAL_PORT\"]
    }" > "$SSM_LOG" 2>&1 &

SESSION_PID=$!

# Wait for tunnel to establish
sleep 3

# Check if session is running
if ! kill -0 $SESSION_PID 2>/dev/null; then
    echo "Failed to start SSM session"
    cat "$SSM_LOG"
    exit 1
fi

echo "Tunnel established on localhost:$LOCAL_PORT"
echo "Press Ctrl+C or type 'exit' to disconnect"

# Start interactive shell
$SHELL

# Cleanup happens automatically via trap
```

### EKS Connection Script
Complete script for connecting to private EKS clusters.

```bash
#!/bin/bash
# eks-connect.sh <cluster-name> [local-port]

CLUSTER_NAME=${1:?Usage: eks-connect.sh <cluster-name> [local-port]}
LOCAL_PORT=${2:-6443}

# Ensure clean environment
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

# Validate credentials
if ! aws sts get-caller-identity &>/dev/null; then
    echo "Please login first: aws sso login"
    exit 1
fi

# Get cluster endpoint
ENDPOINT=$(aws eks describe-cluster --name "$CLUSTER_NAME" \
    --query 'cluster.endpoint' --output text)
EKS_HOST=$(echo "$ENDPOINT" | sed 's|https://||')

# Find bastion
BASTION_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Role,Values=bastion" \
              "Name=instance-state-name,Values=running" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)

if [ "$BASTION_ID" = "None" ]; then
    echo "Error: No bastion found"
    exit 1
fi

# Check port availability
if lsof -ti :$LOCAL_PORT &>/dev/null; then
    echo "Port $LOCAL_PORT in use, finding alternative..."
    LOCAL_PORT=$(find_available_port $LOCAL_PORT)
fi

echo "Connecting to $CLUSTER_NAME via $BASTION_ID on port $LOCAL_PORT"

# Update kubeconfig
aws eks update-kubeconfig --name "$CLUSTER_NAME" 2>/dev/null
# Override server to use local port
kubectl config set-cluster "$CLUSTER_NAME" --server="https://127.0.0.1:$LOCAL_PORT"

# Start session (cleanup via trap)
trap 'kill $SSM_PID 2>/dev/null; exit' EXIT INT TERM

aws ssm start-session \
    --target "$BASTION_ID" \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{
        \"host\":[\"$EKS_HOST\"],
        \"portNumber\":[\"443\"],
        \"localPortNumber\":[\"$LOCAL_PORT\"]
    }" &
SSM_PID=$!

sleep 3

echo "Connected! Run kubectl commands in this shell."
echo "Type 'exit' to disconnect."

# Interactive shell with custom KUBECONFIG
KUBECONFIG=~/.kube/config $SHELL
```

---

## Useful Queries

```bash
# List SSM-managed instances
aws ssm describe-instance-information \
    --query 'InstanceInformationList[*].{ID:InstanceId,IP:IPAddress,Platform:PlatformType,Status:PingStatus}'

# Find instances by tag
aws ssm describe-instance-information \
    --filters Key=tag:Environment,Values=Production \
    --query 'InstanceInformationList[*].InstanceId'

# List active sessions
aws ssm describe-sessions \
    --state Active \
    --query 'Sessions[*].{SessionId:SessionId,Target:Target,Owner:Owner}'

# Get session history
aws ssm describe-sessions \
    --state History \
    --filters key=Owner,value=$(aws sts get-caller-identity --query 'Arn' --output text) \
    --query 'Sessions[*].{SessionId:SessionId,Target:Target,StartDate:StartDate}'

# Terminate session
aws ssm terminate-session --session-id session-id-here
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **SSM over SSH** | Use Session Manager for audit, logging, IAM control |
| **No public IPs** | Keep instances in private subnets, use SSM |
| **VPC endpoints** | Deploy SSM endpoints for private subnet access |
| **Logging** | Enable S3/CloudWatch logging for compliance |
| **IAM policies** | Restrict ssm:StartSession by instance tags |
| **Idle timeout** | Configure automatic session termination |
| **Port forwarding** | Use for database access instead of VPNs |
| **Run Command** | Prefer over SSH for automated tasks |
| **Session preferences** | Standardize shell profiles across team |
| **Multi-account** | Use cross-account IAM roles with SSM |
