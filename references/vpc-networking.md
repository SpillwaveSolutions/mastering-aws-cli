# VPC & Networking

## VPC Management

### Create VPC
```bash
# Create VPC with DNS support
aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=production-vpc}]'

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
    --vpc-id vpc-123 \
    --enable-dns-hostnames

# Enable DNS support
aws ec2 modify-vpc-attribute \
    --vpc-id vpc-123 \
    --enable-dns-support
```

### Subnets
```bash
# Create public subnet
aws ec2 create-subnet \
    --vpc-id vpc-123 \
    --cidr-block 10.0.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'

# Create private subnet
aws ec2 create-subnet \
    --vpc-id vpc-123 \
    --cidr-block 10.0.10.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'

# Enable auto-assign public IP (public subnets)
aws ec2 modify-subnet-attribute \
    --subnet-id subnet-public \
    --map-public-ip-on-launch

# List subnets
aws ec2 describe-subnets \
    --filters Name=vpc-id,Values=vpc-123 \
    --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Name:Tags[?Key==`Name`].Value|[0]}'
```

### Internet Gateway
```bash
# Create IGW
aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=production-igw}]'

# Attach to VPC
aws ec2 attach-internet-gateway \
    --vpc-id vpc-123 \
    --internet-gateway-id igw-123

# Detach IGW
aws ec2 detach-internet-gateway \
    --vpc-id vpc-123 \
    --internet-gateway-id igw-123
```

### NAT Gateway
```bash
# Allocate Elastic IP
EIP_ALLOC=$(aws ec2 allocate-address \
    --domain vpc \
    --query 'AllocationId' \
    --output text)

# Create NAT Gateway (public subnet)
aws ec2 create-nat-gateway \
    --subnet-id subnet-public-1a \
    --allocation-id $EIP_ALLOC \
    --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-1a}]'

# Wait for NAT Gateway
aws ec2 wait nat-gateway-available \
    --nat-gateway-ids nat-123

# Create private NAT Gateway (VPC-to-VPC traffic)
aws ec2 create-nat-gateway \
    --subnet-id subnet-private-1a \
    --connectivity-type private \
    --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=private-nat}]'

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id nat-123
```

### Route Tables
```bash
# Create route table
aws ec2 create-route-table \
    --vpc-id vpc-123 \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# Add route to IGW (public)
aws ec2 create-route \
    --route-table-id rtb-public \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id igw-123

# Add route to NAT Gateway (private)
aws ec2 create-route \
    --route-table-id rtb-private \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id nat-123

# Associate route table with subnet
aws ec2 associate-route-table \
    --route-table-id rtb-public \
    --subnet-id subnet-public-1a

# Replace route
aws ec2 replace-route \
    --route-table-id rtb-private \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id nat-456

# Delete route
aws ec2 delete-route \
    --route-table-id rtb-123 \
    --destination-cidr-block 10.1.0.0/16
```

## Security Groups

### Create and Configure
```bash
# Create security group
aws ec2 create-security-group \
    --group-name web-servers \
    --description "Allow HTTP/HTTPS traffic" \
    --vpc-id vpc-123

# Allow inbound HTTPS
aws ec2 authorize-security-group-ingress \
    --group-id sg-web \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# Allow inbound from CIDR range
aws ec2 authorize-security-group-ingress \
    --group-id sg-web \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/8

# Allow from another security group
aws ec2 authorize-security-group-ingress \
    --group-id sg-db \
    --protocol tcp \
    --port 5432 \
    --source-group sg-web

# Allow all traffic from security group
aws ec2 authorize-security-group-ingress \
    --group-id sg-internal \
    --protocol -1 \
    --source-group sg-internal

# Allow port range
aws ec2 authorize-security-group-ingress \
    --group-id sg-app \
    --protocol tcp \
    --port 8080-8090 \
    --cidr 10.0.0.0/16
```

### Using IP Permissions JSON
```bash
# Complex ingress rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-web \
    --ip-permissions '[
        {
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "HTTPS from anywhere"}]
        },
        {
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "HTTP redirect"}]
        }
    ]'

# Egress rules
aws ec2 authorize-security-group-egress \
    --group-id sg-app \
    --ip-permissions '[
        {
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "IpRanges": [{"CidrIp": "0.0.0.0/0"}]
        },
        {
            "IpProtocol": "tcp",
            "FromPort": 5432,
            "ToPort": 5432,
            "UserIdGroupPairs": [{"GroupId": "sg-db"}]
        }
    ]'
```

### Revoke Rules
```bash
# Revoke ingress
aws ec2 revoke-security-group-ingress \
    --group-id sg-web \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

# Revoke from security group
aws ec2 revoke-security-group-ingress \
    --group-id sg-db \
    --protocol tcp \
    --port 5432 \
    --source-group sg-old-app
```

## Network ACLs

### Create and Configure
```bash
# Create NACL
aws ec2 create-network-acl \
    --vpc-id vpc-123 \
    --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=public-nacl}]'

# Add inbound rule (allow HTTPS)
aws ec2 create-network-acl-entry \
    --network-acl-id acl-123 \
    --rule-number 100 \
    --protocol 6 \
    --port-range From=443,To=443 \
    --cidr-block 0.0.0.0/0 \
    --ingress \
    --rule-action allow

# Add outbound rule (ephemeral ports)
aws ec2 create-network-acl-entry \
    --network-acl-id acl-123 \
    --rule-number 100 \
    --protocol 6 \
    --port-range From=1024,To=65535 \
    --cidr-block 0.0.0.0/0 \
    --egress \
    --rule-action allow

# Associate with subnet
aws ec2 replace-network-acl-association \
    --association-id aclassoc-123 \
    --network-acl-id acl-new
```

## VPC Endpoints

### Gateway Endpoints (S3, DynamoDB)
```bash
# Create S3 gateway endpoint
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-123 \
    --service-name com.amazonaws.us-east-1.s3 \
    --route-table-ids rtb-private-1a rtb-private-1b \
    --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=s3-endpoint}]'

# Create DynamoDB gateway endpoint
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-123 \
    --service-name com.amazonaws.us-east-1.dynamodb \
    --route-table-ids rtb-private-1a rtb-private-1b

# Add endpoint policy
aws ec2 modify-vpc-endpoint \
    --vpc-endpoint-id vpce-123 \
    --policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ]
        }]
    }'
```

### Interface Endpoints (PrivateLink)
```bash
# Create interface endpoint (ECR)
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-123 \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.ecr.api \
    --subnet-ids subnet-private-1a subnet-private-1b \
    --security-group-ids sg-endpoints \
    --private-dns-enabled

# Create SSM endpoints (all 3 required)
for svc in ssm ssmmessages ec2messages; do
    aws ec2 create-vpc-endpoint \
        --vpc-id vpc-123 \
        --vpc-endpoint-type Interface \
        --service-name com.amazonaws.us-east-1.$svc \
        --subnet-ids subnet-private-1a \
        --security-group-ids sg-endpoints \
        --private-dns-enabled
done

# Create Secrets Manager endpoint
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-123 \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.secretsmanager \
    --subnet-ids subnet-private-1a subnet-private-1b \
    --security-group-ids sg-endpoints \
    --private-dns-enabled

# List endpoint services
aws ec2 describe-vpc-endpoint-services \
    --query 'ServiceNames[?contains(@, `secretsmanager`)]'
```

### Manage Endpoints
```bash
# List endpoints
aws ec2 describe-vpc-endpoints \
    --filters Name=vpc-id,Values=vpc-123 \
    --query 'VpcEndpoints[*].{ID:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType}'

# Modify endpoint (add subnets)
aws ec2 modify-vpc-endpoint \
    --vpc-endpoint-id vpce-123 \
    --add-subnet-ids subnet-private-1c

# Delete endpoint
aws ec2 delete-vpc-endpoints \
    --vpc-endpoint-ids vpce-123
```

## VPC Peering

### Create Peering
```bash
# Request peering (same account)
aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-requester \
    --peer-vpc-id vpc-accepter \
    --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=vpc1-to-vpc2}]'

# Request peering (cross-account)
aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-123 \
    --peer-vpc-id vpc-456 \
    --peer-owner-id 987654321098 \
    --peer-region us-west-2

# Accept peering
aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id pcx-123

# Add routes for peering
aws ec2 create-route \
    --route-table-id rtb-vpc1 \
    --destination-cidr-block 10.1.0.0/16 \
    --vpc-peering-connection-id pcx-123

aws ec2 create-route \
    --route-table-id rtb-vpc2 \
    --destination-cidr-block 10.0.0.0/16 \
    --vpc-peering-connection-id pcx-123

# Describe peering
aws ec2 describe-vpc-peering-connections \
    --vpc-peering-connection-ids pcx-123

# Delete peering
aws ec2 delete-vpc-peering-connection \
    --vpc-peering-connection-id pcx-123
```

## Transit Gateway

### Create Transit Gateway
```bash
# Create Transit Gateway
aws ec2 create-transit-gateway \
    --description "Central hub for VPC connectivity" \
    --options '{
        "AmazonSideAsn": 64512,
        "AutoAcceptSharedAttachments": "enable",
        "DefaultRouteTableAssociation": "enable",
        "DefaultRouteTablePropagation": "enable",
        "VpnEcmpSupport": "enable",
        "DnsSupport": "enable"
    }' \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=central-tgw}]'

# Wait for TGW to be available
aws ec2 describe-transit-gateways \
    --transit-gateway-ids tgw-123 \
    --query 'TransitGateways[0].State'
```

### Attach VPCs
```bash
# Create VPC attachment
aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id tgw-123 \
    --vpc-id vpc-prod \
    --subnet-ids subnet-private-1a subnet-private-1b \
    --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=prod-vpc}]'

# Create attachment with appliance mode (for firewalls)
aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id tgw-123 \
    --vpc-id vpc-security \
    --subnet-ids subnet-fw-1a subnet-fw-1b \
    --options ApplianceModeSupport=enable

# List attachments
aws ec2 describe-transit-gateway-vpc-attachments \
    --filters Name=transit-gateway-id,Values=tgw-123
```

### Route Tables
```bash
# Create route table
aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id tgw-123 \
    --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=prod-rt}]'

# Associate attachment with route table
aws ec2 associate-transit-gateway-route-table \
    --transit-gateway-route-table-id tgw-rtb-123 \
    --transit-gateway-attachment-id tgw-attach-vpc1

# Enable propagation
aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-route-table-id tgw-rtb-123 \
    --transit-gateway-attachment-id tgw-attach-vpc1

# Create static route
aws ec2 create-transit-gateway-route \
    --transit-gateway-route-table-id tgw-rtb-123 \
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-attachment-id tgw-attach-vpc1

# Create blackhole route
aws ec2 create-transit-gateway-route \
    --transit-gateway-route-table-id tgw-rtb-123 \
    --destination-cidr-block 192.168.0.0/16 \
    --blackhole

# Get routes
aws ec2 search-transit-gateway-routes \
    --transit-gateway-route-table-id tgw-rtb-123 \
    --filters Name=type,Values=static,propagated
```

### Cross-Account Sharing
```bash
# Share via RAM (Resource Access Manager)
aws ram create-resource-share \
    --name "TGW-Share" \
    --resource-arns arn:aws:ec2:us-east-1:123456789012:transit-gateway/tgw-123 \
    --principals 987654321098

# Accept in target account
aws ram accept-resource-share-invitation \
    --resource-share-invitation-arn arn:aws:ram:us-east-1:123456789012:resource-share-invitation/abc123
```

## VPC Flow Logs

### Enable Flow Logs
```bash
# Flow logs to CloudWatch
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-123 \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/flow-logs \
    --deliver-logs-permission-arn arn:aws:iam::123456789012:role/FlowLogsRole \
    --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=vpc-flow-logs}]'

# Flow logs to S3
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-123 \
    --traffic-type ALL \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::my-flow-logs-bucket/vpc-logs/ \
    --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status}'

# Subnet-level flow logs
aws ec2 create-flow-logs \
    --resource-type Subnet \
    --resource-ids subnet-123 \
    --traffic-type REJECT \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::my-flow-logs-bucket/rejected/

# ENI-level flow logs
aws ec2 create-flow-logs \
    --resource-type NetworkInterface \
    --resource-ids eni-123 \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/eni-logs
```

### Manage Flow Logs
```bash
# List flow logs
aws ec2 describe-flow-logs \
    --filter Name=resource-id,Values=vpc-123

# Delete flow logs
aws ec2 delete-flow-logs \
    --flow-log-ids fl-123
```

## Useful Queries

```bash
# List all VPCs
aws ec2 describe-vpcs \
    --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}'

# Find subnets by type
aws ec2 describe-subnets \
    --filters Name=tag:Name,Values=*private* \
    --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}'

# List security group rules
aws ec2 describe-security-groups \
    --group-ids sg-123 \
    --query 'SecurityGroups[0].IpPermissions'

# Find unused security groups
aws ec2 describe-network-interfaces \
    --query 'NetworkInterfaces[*].Groups[*].GroupId' \
    --output text | tr '\t' '\n' | sort -u > used-sgs.txt
aws ec2 describe-security-groups \
    --query 'SecurityGroups[*].GroupId' \
    --output text | tr '\t' '\n' | sort -u > all-sgs.txt
comm -23 all-sgs.txt used-sgs.txt

# List NAT Gateway status
aws ec2 describe-nat-gateways \
    --query 'NatGateways[*].{ID:NatGatewayId,State:State,Subnet:SubnetId}'

# Get Transit Gateway routes
aws ec2 search-transit-gateway-routes \
    --transit-gateway-route-table-id tgw-rtb-123 \
    --filters Name=state,Values=active \
    --query 'Routes[*].{CIDR:DestinationCidrBlock,Attachment:TransitGatewayAttachments[0].TransitGatewayAttachmentId}'
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **CIDR planning** | Use /16 for VPCs, leave room for growth |
| **Multi-AZ** | Deploy subnets across 3 AZs for HA |
| **Security groups** | Reference other SGs instead of CIDRs for internal traffic |
| **NAT Gateway** | Deploy one per AZ for HA |
| **VPC endpoints** | Use for S3/DynamoDB/ECR to reduce NAT costs |
| **Transit Gateway** | Use for hub-and-spoke with 3+ VPCs |
| **Flow logs** | Enable for security and troubleshooting |
| **Private subnets** | Keep databases and internal services private |
| **NACLs** | Use sparingly, prefer security groups |
| **DNS** | Enable DNS hostnames for service discovery |
