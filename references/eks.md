# Elastic Kubernetes Service (EKS)

## Cluster Management

### Create Cluster
```bash
# Create cluster with Kubernetes 1.32
aws eks create-cluster \
    --name production-cluster \
    --version 1.32 \
    --role-arn arn:aws:iam::123456789012:role/EKSClusterRole \
    --resources-vpc-config \
        subnetIds=subnet-1,subnet-2,subnet-3,\
securityGroupIds=sg-1,\
endpointPublicAccess=false,\
endpointPrivateAccess=true \
    --kubernetes-network-config serviceIpv4Cidr=10.100.0.0/16 \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Wait for cluster to be active
aws eks wait cluster-active --name production-cluster
```

### Describe and List Clusters
```bash
# List clusters
aws eks list-clusters

# Describe cluster
aws eks describe-cluster --name production-cluster

# Get cluster endpoint
aws eks describe-cluster \
    --name production-cluster \
    --query 'cluster.endpoint' \
    --output text

# Get cluster OIDC issuer
aws eks describe-cluster \
    --name production-cluster \
    --query 'cluster.identity.oidc.issuer' \
    --output text
```

### Update Cluster
```bash
# Update Kubernetes version
aws eks update-cluster-version \
    --name production-cluster \
    --kubernetes-version 1.33

# Update cluster config (logging)
aws eks update-cluster-config \
    --name production-cluster \
    --logging '{"clusterLogging":[{"types":["api","audit"],"enabled":true}]}'

# Update endpoint access
aws eks update-cluster-config \
    --name production-cluster \
    --resources-vpc-config \
        endpointPublicAccess=true,\
endpointPrivateAccess=true,\
publicAccessCidrs=["203.0.113.0/24"]
```

## Managed Node Groups

### Create Node Group
```bash
# On-Demand node group
aws eks create-nodegroup \
    --cluster-name production-cluster \
    --nodegroup-name workers-on-demand \
    --node-role arn:aws:iam::123456789012:role/EKSNodeRole \
    --subnets subnet-1 subnet-2 subnet-3 \
    --instance-types m6g.large m6g.xlarge \
    --capacity-type ON_DEMAND \
    --scaling-config minSize=2,maxSize=10,desiredSize=3 \
    --disk-size 50 \
    --labels environment=production,tier=workers

# Spot node group (cost optimization)
aws eks create-nodegroup \
    --cluster-name production-cluster \
    --nodegroup-name workers-spot \
    --node-role arn:aws:iam::123456789012:role/EKSNodeRole \
    --subnets subnet-1 subnet-2 subnet-3 \
    --instance-types m6g.large m6g.xlarge c6g.large c6g.xlarge \
    --capacity-type SPOT \
    --scaling-config minSize=0,maxSize=20,desiredSize=5

# With launch template
aws eks create-nodegroup \
    --cluster-name production-cluster \
    --nodegroup-name custom-workers \
    --node-role arn:aws:iam::123456789012:role/EKSNodeRole \
    --subnets subnet-1 subnet-2 \
    --launch-template name=eks-node-template,version=1 \
    --scaling-config minSize=1,maxSize=5,desiredSize=2
```

### Manage Node Groups
```bash
# List node groups
aws eks list-nodegroups --cluster-name production-cluster

# Describe node group
aws eks describe-nodegroup \
    --cluster-name production-cluster \
    --nodegroup-name workers-on-demand

# Update scaling
aws eks update-nodegroup-config \
    --cluster-name production-cluster \
    --nodegroup-name workers-on-demand \
    --scaling-config minSize=3,maxSize=15,desiredSize=5

# Update node group version (rolling update)
aws eks update-nodegroup-version \
    --cluster-name production-cluster \
    --nodegroup-name workers-on-demand \
    --kubernetes-version 1.32

# Delete node group
aws eks delete-nodegroup \
    --cluster-name production-cluster \
    --nodegroup-name workers-spot
```

## Fargate Profiles

### Create Fargate Profile
```bash
aws eks create-fargate-profile \
    --cluster-name production-cluster \
    --fargate-profile-name app-profile \
    --pod-execution-role-arn arn:aws:iam::123456789012:role/EKSFargatePodRole \
    --subnets subnet-private-1 subnet-private-2 \
    --selectors \
        namespace=production,labels={compute=fargate} \
        namespace=kube-system,labels={k8s-app=kube-dns}
```

### Manage Fargate Profiles
```bash
# List profiles
aws eks list-fargate-profiles --cluster-name production-cluster

# Describe profile
aws eks describe-fargate-profile \
    --cluster-name production-cluster \
    --fargate-profile-name app-profile

# Delete profile
aws eks delete-fargate-profile \
    --cluster-name production-cluster \
    --fargate-profile-name app-profile
```

## Add-ons

### Manage Add-ons
```bash
# List available add-ons
aws eks describe-addon-versions --kubernetes-version 1.32

# List installed add-ons
aws eks list-addons --cluster-name production-cluster

# Create add-on
aws eks create-addon \
    --cluster-name production-cluster \
    --addon-name vpc-cni \
    --addon-version v1.16.0-eksbuild.1 \
    --service-account-role-arn arn:aws:iam::123456789012:role/VPCCNIRole

aws eks create-addon \
    --cluster-name production-cluster \
    --addon-name coredns

aws eks create-addon \
    --cluster-name production-cluster \
    --addon-name kube-proxy

aws eks create-addon \
    --cluster-name production-cluster \
    --addon-name aws-ebs-csi-driver \
    --service-account-role-arn arn:aws:iam::123456789012:role/EBSCSIRole

# Update add-on
aws eks update-addon \
    --cluster-name production-cluster \
    --addon-name vpc-cni \
    --addon-version v1.17.0-eksbuild.1

# Delete add-on
aws eks delete-addon \
    --cluster-name production-cluster \
    --addon-name aws-ebs-csi-driver
```

## Kubeconfig Setup

### Basic Configuration
```bash
# Update kubeconfig (uses current AWS credentials)
aws eks update-kubeconfig \
    --name production-cluster \
    --region us-east-1

# With custom alias
aws eks update-kubeconfig \
    --name production-cluster \
    --alias prod-cluster

# With role assumption (cross-account or different role)
aws eks update-kubeconfig \
    --name production-cluster \
    --role-arn arn:aws:iam::123456789012:role/EKSAdminRole

# Dry run (show kubeconfig without writing)
aws eks update-kubeconfig \
    --name production-cluster \
    --dry-run
```

### Verify Access
```bash
# Test connection
kubectl get nodes
kubectl get pods -A

# Get cluster info
kubectl cluster-info
```

## IAM Roles for Service Accounts (IRSA)

IRSA allows Kubernetes pods to assume IAM roles without using node-level credentials.

### Step 1: Create OIDC Provider
```bash
# Get OIDC issuer URL
OIDC_URL=$(aws eks describe-cluster \
    --name production-cluster \
    --query 'cluster.identity.oidc.issuer' \
    --output text)

# Extract OIDC ID
OIDC_ID=$(echo $OIDC_URL | sed 's|https://||')

# Create OIDC provider (if not exists)
aws iam create-open-id-connect-provider \
    --url $OIDC_URL \
    --client-id-list sts.amazonaws.com \
    --thumbprint-list $(openssl s_client -servername oidc.eks.us-east-1.amazonaws.com \
        -connect oidc.eks.us-east-1.amazonaws.com:443 2>/dev/null \
        | openssl x509 -fingerprint -sha1 -noout \
        | sed 's/://g' | cut -d= -f2 | tr '[:upper:]' '[:lower:]')
```

### Step 2: Create IAM Role with Trust Policy
```bash
# Create trust policy file
cat > trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Federated": "arn:aws:iam::123456789012:oidc-provider/${OIDC_ID}"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
            "StringEquals": {
                "${OIDC_ID}:sub": "system:serviceaccount:default:my-service-account",
                "${OIDC_ID}:aud": "sts.amazonaws.com"
            }
        }
    }]
}
EOF

# Create role
aws iam create-role \
    --role-name MyPodRole \
    --assume-role-policy-document file://trust-policy.json

# Attach policy
aws iam attach-role-policy \
    --role-name MyPodRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 3: Annotate Service Account
```bash
# Create service account with annotation (kubectl)
kubectl create serviceaccount my-service-account
kubectl annotate serviceaccount my-service-account \
    eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/MyPodRole
```

## Accessing Private EKS Clusters

### Method 1: SSM Port Forwarding
```bash
# Get EKS API endpoint (without https://)
EKS_ENDPOINT=$(aws eks describe-cluster \
    --name production-cluster \
    --query 'cluster.endpoint' \
    --output text | sed 's|https://||')

# Start port forwarding through bastion
aws ssm start-session \
    --target i-bastion-instance-id \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{\"host\":[\"$EKS_ENDPOINT\"],\"portNumber\":[\"443\"],\"localPortNumber\":[\"9443\"]}"

# In another terminal, update kubeconfig
aws eks update-kubeconfig --name production-cluster

# Modify kubeconfig to use localhost:9443
kubectl config set-cluster arn:aws:eks:us-east-1:123456789012:cluster/production-cluster \
    --server=https://localhost:9443

# Disable certificate verification (for port forwarding)
kubectl config set-cluster arn:aws:eks:us-east-1:123456789012:cluster/production-cluster \
    --insecure-skip-tls-verify=true

# Test access
kubectl get nodes
```

### Method 2: SOCKS Proxy via SSH over SSM

**Configure SSH (~/.ssh/config):**
```
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
    User ec2-user
    StrictHostKeyChecking no
```

**Start SOCKS proxy:**
```bash
# Start SSH SOCKS proxy through SSM bastion
ssh -N -D 127.0.0.1:1080 i-bastion-instance-id &

# Configure kubectl to use proxy
export HTTPS_PROXY=socks5h://127.0.0.1:1080

# Update kubeconfig normally
aws eks update-kubeconfig --name production-cluster

# Use kubectl (traffic goes through SOCKS proxy)
kubectl get nodes
```

### Prerequisites for Private Cluster Access
```bash
# Required VPC endpoints
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxxxx \
    --service-name com.amazonaws.us-east-1.ssm \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-xxxxx \
    --security-group-ids sg-xxxxx

aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxxxx \
    --service-name com.amazonaws.us-east-1.ssmmessages \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-xxxxx \
    --security-group-ids sg-xxxxx

aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxxxx \
    --service-name com.amazonaws.us-east-1.ec2messages \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-xxxxx \
    --security-group-ids sg-xxxxx
```

## Access Entries (EKS Access Management)

### Create Access Entry
```bash
# Grant cluster access to IAM principal
aws eks create-access-entry \
    --cluster-name production-cluster \
    --principal-arn arn:aws:iam::123456789012:role/DeveloperRole \
    --type STANDARD

# Associate access policy
aws eks associate-access-policy \
    --cluster-name production-cluster \
    --principal-arn arn:aws:iam::123456789012:role/DeveloperRole \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=namespace,namespaces=default

# Admin access
aws eks associate-access-policy \
    --cluster-name production-cluster \
    --principal-arn arn:aws:iam::123456789012:role/AdminRole \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
    --access-scope type=cluster
```

### Manage Access Entries
```bash
# List access entries
aws eks list-access-entries --cluster-name production-cluster

# Describe access entry
aws eks describe-access-entry \
    --cluster-name production-cluster \
    --principal-arn arn:aws:iam::123456789012:role/DeveloperRole

# Delete access entry
aws eks delete-access-entry \
    --cluster-name production-cluster \
    --principal-arn arn:aws:iam::123456789012:role/DeveloperRole
```

## Useful Queries

```bash
# Get node group status
aws eks describe-nodegroup \
    --cluster-name production-cluster \
    --nodegroup-name workers-on-demand \
    --query 'nodegroup.status'

# List all node groups with status
aws eks list-nodegroups --cluster-name production-cluster \
    --query 'nodegroups[]' --output text | \
    xargs -I {} aws eks describe-nodegroup \
        --cluster-name production-cluster \
        --nodegroup-name {} \
        --query '{name:nodegroup.nodegroupName,status:nodegroup.status}'

# Get cluster CA certificate
aws eks describe-cluster \
    --name production-cluster \
    --query 'cluster.certificateAuthority.data' \
    --output text | base64 -d

# Check cluster health
aws eks describe-cluster \
    --name production-cluster \
    --query 'cluster.{status:status,version:version,endpoint:endpoint}'
```

## Waiters

```bash
# Wait for cluster creation
aws eks wait cluster-active --name production-cluster

# Wait for node group creation
aws eks wait nodegroup-active \
    --cluster-name production-cluster \
    --nodegroup-name workers-on-demand

# Wait for node group deletion
aws eks wait nodegroup-deleted \
    --cluster-name production-cluster \
    --nodegroup-name old-workers
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Private endpoints** | Use private cluster endpoints for production |
| **IRSA** | Use IAM Roles for Service Accounts instead of node role |
| **Spot instances** | Use Spot for stateless, fault-tolerant workloads |
| **Managed node groups** | Use managed node groups for easier operations |
| **Add-on management** | Use EKS add-ons for VPC CNI, CoreDNS, kube-proxy |
| **Version alignment** | Keep control plane and node group versions aligned |
| **Access entries** | Use EKS access entries for cluster access management |
| **Logging** | Enable control plane logging for audit and troubleshooting |
