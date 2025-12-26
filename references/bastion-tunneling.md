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
