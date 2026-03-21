# Day 34 — AWS CloudFormation Nested Stacks

Deploy a production-style multi-tier Nginx application on AWS using CloudFormation nested stacks. A single parent stack orchestrates four child stacks covering networking, compute, load balancing, and static file storage. Traffic is served over plain HTTP — no certificate or DNS configuration required.

---

## CloudFormation Nested Stacks — Concepts

### What is a Nested Stack?

A nested stack is a CloudFormation stack that is created as a resource inside another stack using the `AWS::CloudFormation::Stack` resource type. The stack that contains this resource is called the **parent stack**. The stack being referenced is called the **child stack** or **nested stack**.

```yaml
# This is all it takes to embed a child stack inside a parent
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/network-stack.yaml
      Parameters:
        VPCCidr: 10.0.0.0/16
```

The child template lives in S3. The parent references it by URL. CloudFormation downloads, parses, and deploys it as if it were a standalone stack — except it is owned and managed by the parent.

---

### Why Use Nested Stacks?

**1. CloudFormation resource limit**
A single CloudFormation template has a hard limit of **500 resources**. A complex infrastructure can easily hit this. Nested stacks let you split resources across multiple templates, each with its own 500 resource budget.

**2. Template size limit**
A template body has a maximum size of **51,200 bytes** when passed directly and **460,800 bytes** when uploaded to S3. Splitting into nested stacks keeps each template small and manageable.

**3. Separation of concerns**
Each stack owns one layer of the infrastructure. The network team can update the network stack without touching the compute stack. The app team can redeploy the compute stack without affecting networking. Each stack has its own lifecycle, drift detection, and rollback scope.

**4. Reusability**
A child template can be referenced by multiple parent stacks. For example a standard VPC template can be reused across dev, staging, and production environments by passing different parameters.

**5. Easier to read and maintain**
A 2000-line monolithic template is hard to navigate. Five 400-line templates with clear names (`network-stack`, `alb-stack`, `compute-stack`) are much easier to understand and review.

---

### Nested Stack Resource Type

```yaml
Type: AWS::CloudFormation::Stack
Properties:
  TemplateURL: String           # required — S3 URL of the child template
  Parameters:                   # optional — key-value pairs passed to the child
    Key: Value
  TimeoutInMinutes: Number      # optional — how long before CloudFormation gives up
  NotificationARNs:             # optional — SNS topics for stack events
    - arn:aws:sns:...
  Tags:                         # optional — tags applied to the nested stack
    - Key: Name
      Value: MyStack
```

> **Important:** `TemplateURL` must be an S3 HTTPS URL. Local file paths do not work. The S3 bucket must be in the same region as the stack being deployed.

---

### Passing Values Between Stacks

There are two patterns for sharing values between stacks. In nested stacks the **Parameters pattern** is strongly preferred.

#### Pattern 1 — Parameters (recommended for nested stacks)

The parent reads a child stack output using `!GetAtt` and passes it directly as a parameter to the next child stack. The child stack receives it as a plain parameter — no cross-stack reference needed.

```yaml
# Parent reads Network stack output and passes to ALB stack
ALBStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: !Sub '${TemplateBaseURL}/alb-stack.yaml'
    Parameters:
      VPCId:           !GetAtt NetworkStack.Outputs.VPCId
      PublicSubnet1Id: !GetAtt NetworkStack.Outputs.PublicSubnet1Id
```

```yaml
# ALB stack receives it as a plain parameter — no ImportValue needed
Parameters:
  VPCId:
    Type: AWS::EC2::VPC::Id
```

This is clean, explicit, and never breaks due to name mismatches.

#### Pattern 2 — Fn::ImportValue (avoid in nested stacks)

`Fn::ImportValue` reads a value exported by another stack using its export name. This works well for **independent standalone stacks** but is fragile in nested stacks because the export name must be constructed from the nested stack name, which contains an auto-generated suffix.

```yaml
# Fragile — if NetworkStackName param has any mismatch this silently fails
VpcId:
  Fn::ImportValue: !Sub '${NetworkStackName}-VPCId'
```

When CloudFormation creates a nested stack named `NetworkStack` inside a parent named `my-app`, the actual stack name becomes something like `my-app-NetworkStack-1A2B3C4D5E6F`. If `NetworkStackName` parameter is set to just `my-app-NetworkStack`, the export lookup silently fails and the resource creation fails with a cryptic error.

**Rule:** In nested stacks always use `!GetAtt StackResource.Outputs.OutputKey`. Reserve `Fn::ImportValue` for cross-account or cross-region sharing between independent standalone stacks.

---

### Reading Child Stack Outputs

Once a child stack is deployed, the parent can read any value from its `Outputs` section using `!GetAtt`:

```yaml
# Syntax
!GetAtt LogicalResourceName.Outputs.OutputKey

# Real examples from this project
!GetAtt NetworkStack.Outputs.VPCId
!GetAtt ALBStack.Outputs.TargetGroupArn
!GetAtt DataStack.Outputs.BucketArn
```

The `LogicalResourceName` is the key you used in the parent's `Resources` section (e.g. `NetworkStack`). The `OutputKey` is the key from the child template's `Outputs` section (e.g. `VPCId`).

---

### Controlling Deployment Order — DependsOn

By default CloudFormation deploys resources in parallel where possible. If a child stack needs outputs from another child stack, you must tell CloudFormation to wait using `DependsOn`.

```yaml
# ALB stack must wait for Network stack to finish
ALBStack:
  Type: AWS::CloudFormation::Stack
  DependsOn: NetworkStack      # single dependency
  Properties:
    ...

# Compute stack must wait for all three stacks
ComputeStack:
  Type: AWS::CloudFormation::Stack
  DependsOn:
    - NetworkStack             # multiple dependencies
    - ALBStack
    - DataStack
  Properties:
    ...
```

> **Note:** When you use `!GetAtt StackA.Outputs.SomeKey` inside another stack's Parameters, CloudFormation automatically infers the dependency and waits. `DependsOn` is technically redundant in that case but is still good practice to write explicitly so the dependency is clear to anyone reading the template.

---

### Stack Lifecycle and Ownership

A nested stack's lifecycle is completely tied to the parent stack.

| Parent action | What happens to child stacks |
|---|---|
| Create | All child stacks are created in dependency order |
| Update | Only child stacks with changed parameters or template URLs are updated |
| Delete | All child stacks are deleted in reverse dependency order |
| Rollback | All child stacks roll back together |

You **cannot delete a nested stack directly** from the console or CLI while the parent exists. Attempting this fails with an error. You must delete the parent, which then deletes all children in the correct order.

---

### Rollback Behaviour and Timeouts

**Rollback scope:** If any resource in any nested stack fails to create or update, CloudFormation rolls back the entire parent stack and all its children. The failure of one child rolls back everything.

**TimeoutInMinutes:** Always set this on nested stacks that involve resources with unpredictable creation times — NAT gateways, ACM certificates waiting for DNS validation, RDS instances. Without a timeout a stuck resource can hold up the deployment indefinitely.

```yaml
NetworkStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: !Sub '${TemplateBaseURL}/network-stack.yaml'
    TimeoutInMinutes: 15   # NAT Gateway creation can take a few minutes
```

---

### CAPABILITY_NAMED_IAM

Any stack that creates IAM resources (roles, policies, instance profiles) requires explicit acknowledgement using the `--capabilities` flag. Without it the deployment fails immediately.

```bash
aws cloudformation deploy   --template-file parent-stack.yaml   --stack-name my-app   --capabilities CAPABILITY_NAMED_IAM
```

| Capability | When required |
|---|---|
| `CAPABILITY_IAM` | Stack creates IAM resources with auto-generated names |
| `CAPABILITY_NAMED_IAM` | Stack creates IAM resources with explicit `RoleName` or `PolicyName` |
| `CAPABILITY_AUTO_EXPAND` | Stack uses macros or transforms such as SAM |

Since the compute stack creates an IAM role with an explicit `RoleName: !Sub '${AWS::StackName}-EC2InstanceRole'`, `CAPABILITY_NAMED_IAM` is required.

---

### Template URL Rules

Child templates must be in S3. The URL format matters:

```
# Correct formats
https://s3.amazonaws.com/bucket-name/path/template.yaml
https://s3.us-east-1.amazonaws.com/bucket-name/path/template.yaml
https://bucket-name.s3.amazonaws.com/path/template.yaml

# Wrong — local paths do not work
./network-stack.yaml               ❌
file:///home/user/template.yaml    ❌
```

The S3 bucket must be in the **same region** as the CloudFormation stack. Cross-region template URLs fail at deployment time.

---

### Nested Stack Naming

When CloudFormation creates a nested stack it generates a name by combining the parent stack name, the logical resource name, and a random suffix:

```
Parent stack name:    my-app-stack
Logical resource:     NetworkStack
Generated name:       my-app-stack-NetworkStack-1A2B3C4D5E6F
```

This generated name is what appears in the CloudFormation console. It is also what `${AWS::StackName}` resolves to inside the child template. This is why using `${AWS::StackName}` as a prefix for resource names that have character limits (like ALB names — 32 chars max) fails. The generated name is already much longer than 32 characters.

**Solution:** Omit the `Name` property from length-limited resources and use `Tags` for console identification instead.

---

### Outputs Best Practices

Child stacks should always define `Outputs` for every value the parent or sibling stacks might need. Export names are optional for nested stacks (since the parent uses `!GetAtt` not `Fn::ImportValue`) but are useful if you ever want to query the child stack independently.

```yaml
# Child stack outputs
Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'   # optional but useful

# Parent reads it
Parameters:
  VPCId: !GetAtt NetworkStack.Outputs.VPCId
```

---

### Nested Stacks vs StackSets

| Feature | Nested Stacks | StackSets |
|---|---|---|
| Purpose | Split one deployment into multiple templates | Deploy the same stack to multiple accounts/regions |
| Parent-child relationship | Yes — parent owns children | No — each stack is independent |
| Cross-account | No | Yes |
| Cross-region | No | Yes |
| Use case | Breaking up a large single-account deployment | Deploying identical infra across org units |

---

### Drift Detection on Nested Stacks

Drift detection checks whether the actual state of your resources matches what CloudFormation expects. On nested stacks you run drift detection on the **parent stack** — CloudFormation automatically runs it on all nested stacks and surfaces the results in a single report.

```bash
# Detect drift on parent (includes all nested stacks)
aws cloudformation detect-stack-drift --stack-name my-app-stack

# Check result
aws cloudformation describe-stack-drift-detection-status   --stack-drift-detection-id <id-from-above>
```

---

### Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `TemplateURL must be a valid S3 URL` | Using a local file path | Upload template to S3 first |
| `Resource failed to create: [ALB, TG]` | `Fn::ImportValue` export name mismatch | Switch to `!GetAtt` + Parameters pattern |
| `Name exceeds maximum length` | `${AWS::StackName}` prefix too long | Remove the `Name` property, use Tags |
| `CAPABILITY_NAMED_IAM required` | Stack creates named IAM resources | Add `--capabilities CAPABILITY_NAMED_IAM` |
| `Cannot delete nested stack` | Trying to delete child without parent | Delete the parent stack instead |
| `S3 bucket must be in the same region` | Template URL bucket in wrong region | Re-upload templates to bucket in same region |

---

## Architecture Overview

```
Internet
   │
   ▼
Application Load Balancer  (public subnets — us-east-1a, us-east-1b)
   │   port 80 → forwards to Nginx Target Group
   ▼
Auto Scaling Group         (private subnets — us-east-1a, us-east-1b)
   │   min: 1  desired: 2  max: 4
   ▼
Nginx EC2 Instances        (Amazon Linux 2023 / t3.micro)
   │   no SSH key — access via SSM Session Manager
   ▼
S3 Bucket                  (static website hosting)
   │   EC2 uploads templates on boot via IAM role
```

### VPC CIDR Layout

| Subnet | CIDR | AZ | Type |
|---|---|---|---|
| PrivateSubnet1 | 192.168.0.0/24 | us-east-1a | Private |
| PrivateSubnet2 | 192.168.1.0/24 | us-east-1b | Private |
| PublicSubnet1 | 192.168.2.0/24 | us-east-1a | Public |
| PublicSubnet2 | 192.168.3.0/24 | us-east-1b | Public |

---

## Template Structure

```
.
├── parent-stack.yaml       ← deploy this one
├── network-stack.yaml      ← child: VPC, subnets, NAT, SGs
├── alb-stack.yaml          ← child: ALB, target group, listener
├── compute-stack.yaml      ← child: IAM, launch template, ASG
└── data-stack.yaml         ← child: S3 static website bucket
```

All child templates must be uploaded to an S3 bucket before deploying the parent stack.

---

## Deployment Order

The parent stack deploys child stacks in this order based on dependencies:

```
Network ──┐
          ├──► ALB ──► Compute
Data    ──┘
```

Network and Data have no dependencies — they deploy in parallel first.
ALB depends on Network outputs (VPC, subnets, security groups).
Compute depends on Network + ALB (target group ARN) + Data (S3 bucket name and ARN).

### How values flow between stacks

The pattern used throughout is: **parent reads child outputs via `!GetAtt` and passes them as explicit parameters to the next child stack**. No child stack uses `Fn::ImportValue` — this avoids export name mismatch errors that occur when nested stack names contain auto-generated suffixes.

```
NetworkStack outputs
       │
       │  !GetAtt NetworkStack.Outputs.VPCId
       ▼
Parent passes as Parameters
       │
       ▼
ALBStack receives as Parameters (VPCId, SubnetIds, SGIds)
       │
       │  !GetAtt ALBStack.Outputs.TargetGroupArn
       ▼
Parent passes as Parameters
       │
       ▼
ComputeStack receives as Parameters (TargetGroupArn, SubnetIds, S3BucketArn...)
```

---

## Stack 1 — Network Stack (`network-stack.yaml`)

**Deploys first. No dependencies.**

### Resources created

| Resource | Type | Details |
|---|---|---|
| MyVPC | VPC | CIDR: 192.168.0.0/20, DNS enabled |
| PrivateSubnet1 | Subnet | 192.168.0.0/24 — us-east-1a |
| PrivateSubnet2 | Subnet | 192.168.1.0/24 — us-east-1b |
| PublicSubnet1 | Subnet | 192.168.2.0/24 — us-east-1a, MapPublicIpOnLaunch: true |
| PublicSubnet2 | Subnet | 192.168.3.0/24 — us-east-1b, MapPublicIpOnLaunch: true |
| InternetGateway | IGW | Attached to VPC |
| PublicRouteTable | Route Table | 0.0.0.0/0 → IGW |
| NatGatewayEIP | Elastic IP | Allocated for NAT gateway |
| NatGateway | NAT GW | Placed in PublicSubnet1 |
| PrivateRouteTable | Route Table | 0.0.0.0/0 → NAT Gateway |
| ALBSecurityGroup | SG | Inbound: 80, 443 from 0.0.0.0/0 |
| EC2SecurityGroup | SG | Inbound: 80 from ALBSecurityGroup only |

### Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Network Stack - VPC with public/private subnets across 2 AZs,
  Internet Gateway, NAT Gateway, route tables, and Security Groups.

Resources:

  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/20
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: MyVPC

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 192.168.0.0/24
      AvailabilityZone: us-east-1a
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: PrivateSubnet1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 192.168.1.0/24
      AvailabilityZone: us-east-1b
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: PrivateSubnet2

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 192.168.2.0/24
      AvailabilityZone: us-east-1a
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 192.168.3.0/24
      AvailabilityZone: us-east-1b
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet2

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: MyInternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: PublicRouteTable

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: MyNatGateway

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: PrivateRouteTable

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for the Application Load Balancer
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: Allow HTTP from internet
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: Allow HTTPS from internet
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: ALBSecurityGroup

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for EC2 instances - allows traffic from ALB only
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup
          Description: Allow HTTP from ALB only
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: EC2SecurityGroup

Outputs:
  NetworkStackName:
    Value: !Ref AWS::StackName
    Export:
      Name: !Sub '${AWS::StackName}-StackName'

  VPCId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'

  PrivateSubnet1Id:
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub '${AWS::StackName}-PrivateSubnet1Id'

  PrivateSubnet2Id:
    Value: !Ref PrivateSubnet2
    Export:
      Name: !Sub '${AWS::StackName}-PrivateSubnet2Id'

  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnet1Id'

  PublicSubnet2Id:
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnet2Id'

  ALBSecurityGroupId:
    Value: !Ref ALBSecurityGroup
    Export:
      Name: !Sub '${AWS::StackName}-ALBSecurityGroupId'

  EC2SecurityGroupId:
    Value: !Ref EC2SecurityGroup
    Export:
      Name: !Sub '${AWS::StackName}-EC2SecurityGroupId'
```

---

## Stack 2 — ALB Stack (`alb-stack.yaml`)

**Depends on: Network stack.**

### Resources created

| Resource | Type | Details |
|---|---|---|
| ApplicationLoadBalancer | ALB | Internet-facing, public subnets |
| NginxTargetGroup | Target Group | HTTP port 80, health check on / |
| HTTPListener | Listener | Port 80 → forward to target group |

> **Note on naming:** The `Name` property is intentionally omitted from the ALB and Target Group. AWS enforces a 32 character limit on these names. When deployed as a nested stack, `${AWS::StackName}` expands to something like `test-stack-ALBStack-UGMC4ND0JSJ6` which immediately exceeds the limit. Omitting the name lets AWS auto-generate a valid unique name. The `Tags` block still carries a `Name` tag for console visibility.

### Traffic routing

```
User → http://alb-dns-name:80
              │
       HTTPListener (port 80)
              │  Action: forward
              ▼
       NginxTargetGroup (HTTP:80)
              │
              ▼
       EC2 Instances (Nginx)
```

### Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  ALB Stack - Application Load Balancer, Target Group, and HTTP listener.

Parameters:
  VPCId:
    Type: AWS::EC2::VPC::Id

  PublicSubnet1Id:
    Type: AWS::EC2::Subnet::Id

  PublicSubnet2Id:
    Type: AWS::EC2::Subnet::Id

  ALBSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id

Resources:

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Type: application
      IpAddressType: ipv4
      Subnets:
        - !Ref PublicSubnet1Id
        - !Ref PublicSubnet2Id
      SecurityGroups:
        - !Ref ALBSecurityGroupId
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: '60'
        - Key: deletion_protection.enabled
          Value: 'true'
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-ALB'

  NginxTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Protocol: HTTP
      Port: 80
      VpcId: !Ref VPCId
      TargetType: instance
      HealthCheckEnabled: true
      HealthCheckProtocol: HTTP
      HealthCheckPort: traffic-port
      HealthCheckPath: /
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      Matcher:
        HttpCode: '200'
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: '30'
        - Key: stickiness.enabled
          Value: 'true'
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-NginxTG'

  HTTPListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref NginxTargetGroup

Outputs:
  ALBArn:
    Value: !Ref ApplicationLoadBalancer
    Export:
      Name: !Sub '${AWS::StackName}-ALBArn'

  ALBDNSName:
    Value: !GetAtt ApplicationLoadBalancer.DNSName
    Export:
      Name: !Sub '${AWS::StackName}-ALBDNSName'

  TargetGroupArn:
    Value: !Ref NginxTargetGroup
    Export:
      Name: !Sub '${AWS::StackName}-TargetGroupArn'
```

---

## Stack 3 — Compute Stack (`compute-stack.yaml`)

**Depends on: Network stack + ALB stack + Data stack.**

### Resources created

| Resource | Type | Details |
|---|---|---|
| EC2InstanceRole | IAM Role | SSM managed policy + S3 upload inline policy |
| EC2InstanceProfile | Instance Profile | Wraps the IAM role for EC2 |
| NginxLaunchTemplate | Launch Template | AL2023, IMDSv2 enforced, no SSH key |
| NginxAutoScalingGroup | ASG | Private subnets, registers with ALB TG |
| ScaleOutPolicy | Scaling Policy | CPU target tracking at 60% |

### IAM Role permissions

The EC2 instances have two permission sets attached to the role:

**`AmazonSSMManagedInstanceCore`** (AWS managed policy) — allows Systems Manager Session Manager to connect to the instance without needing a bastion host or SSH key.

**`S3UploadPolicy`** (inline policy) — scoped tightly to the specific data bucket:

```
s3:PutObject      → upload files to bucket
s3:PutObjectAcl   → set ACL on uploaded objects
s3:GetObject      → read back uploaded files
s3:DeleteObject   → clean up if needed
s3:ListBucket     → list bucket contents
```

### UserData explained

When an EC2 instance boots the following script runs automatically:

```
1. dnf update + install nginx, amazon-ssm-agent, unzip, wget
2. systemctl enable/start amazon-ssm-agent
3. Fetch instance metadata via IMDSv2:
      - Get session token (PUT request to metadata service)
      - Use token to fetch INSTANCE_ID and AZ
4. Write /usr/share/nginx/html/index.html showing instance ID and AZ
5. systemctl enable/start nginx
6. wget 2155_modern_musician.zip from tooplate.com
7. wget 2152_event_invitation.zip from tooplate.com
8. unzip both archives
9. aws s3 cp modern_musician/ → s3://bucket/
10. aws s3 cp event_invitation/ → s3://bucket/
```

### Why IMDSv2 for metadata?

IMDSv2 requires a session token before any metadata request. This protects against SSRF (Server Side Request Forgery) attacks where a malicious payload tricks the server into fetching its own credentials from the metadata endpoint.

```bash
# Step 1 — get a token valid for 6 hours
TOKEN=$(curl -sf -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Step 2 — use token for all metadata requests
INSTANCE_ID=$(curl -sf -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
```

### Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Compute Stack - IAM Role, Launch Template, Auto Scaling Group for Nginx.

Parameters:
  PrivateSubnet1Id:
    Type: AWS::EC2::Subnet::Id
  PrivateSubnet2Id:
    Type: AWS::EC2::Subnet::Id
  EC2SecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
  TargetGroupArn:
    Type: String
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
  MinSize:
    Type: Number
    Default: 1
  MaxSize:
    Type: Number
    Default: 4
  DesiredCapacity:
    Type: Number
    Default: 2
  S3BucketName:
    Type: String
  S3BucketArn:
    Type: String

Resources:

  EC2InstanceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${AWS::StackName}-EC2InstanceRole'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Policies:
        - PolicyName: S3UploadPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowPutObjectToDataBucket
                Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:PutObjectAcl
                  - s3:GetObject
                  - s3:DeleteObject
                  - s3:ListBucket
                Resource:
                  - !Ref S3BucketArn
                  - !Sub '${S3BucketArn}/*'

  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: !Sub '${AWS::StackName}-EC2InstanceProfile'
      Roles:
        - !Ref EC2InstanceRole

  NginxLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub '${AWS::StackName}-NginxLaunchTemplate'
      LaunchTemplateData:
        ImageId: !Ref LatestAmiId
        InstanceType: !Ref InstanceType
        IamInstanceProfile:
          Arn: !GetAtt EC2InstanceProfile.Arn
        SecurityGroupIds:
          - !Ref EC2SecurityGroupId
        MetadataOptions:
          HttpTokens: required
          HttpEndpoint: enabled
        Monitoring:
          Enabled: true
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            set -euxo pipefail
            dnf update -y
            dnf install -y nginx amazon-ssm-agent unzip wget
            systemctl enable amazon-ssm-agent
            systemctl start amazon-ssm-agent
            systemctl enable nginx
            systemctl start nginx
            cd /tmp
            wget https://www.tooplate.com/zip-templates/2155_modern_musician.zip
            wget https://www.tooplate.com/zip-templates/2152_event_invitation.zip
            unzip 2155_modern_musician.zip -d /usr/share/nginx/html/
            unzip 2152_event_invitation.zip -d /tmp/
            aws s3 cp /usr/share/nginx/html/2155_modern_musician/ s3://${S3BucketName}/2155_modern_musician/ --recursive
            aws s3 cp /tmp/2152_event_invitation/ s3://${S3BucketName}/2152_event_invitation/ --recursive
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub '${AWS::StackName}-NginxInstance'

  NginxAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub '${AWS::StackName}-NginxASG'
      LaunchTemplate:
        LaunchTemplateId: !Ref NginxLaunchTemplate
        Version: !GetAtt NginxLaunchTemplate.LatestVersionNumber
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref DesiredCapacity
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1Id
        - !Ref PrivateSubnet2Id
      TargetGroupARNs:
        - !Ref TargetGroupArn
      HealthCheckType: ELB
      HealthCheckGracePeriod: 120
      MetricsCollection:
        - Granularity: '1Minute'
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-NginxASG'
          PropagateAtLaunch: true

  ScaleOutPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref NginxAutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 60.0

Outputs:
  AutoScalingGroupName:
    Value: !Ref NginxAutoScalingGroup
    Export:
      Name: !Sub '${AWS::StackName}-ASGName'

  LaunchTemplateId:
    Value: !Ref NginxLaunchTemplate
    Export:
      Name: !Sub '${AWS::StackName}-LaunchTemplateId'

  EC2InstanceRoleArn:
    Value: !GetAtt EC2InstanceRole.Arn
    Export:
      Name: !Sub '${AWS::StackName}-EC2RoleArn'
```

---

## Stack 4 — Data Stack (`data-stack.yaml`)

**Deploys in parallel with Network. No dependencies.**

### Resources created

| Resource | Type | Details |
|---|---|---|
| MyS3Bucket | S3 Bucket | Versioning enabled, static website hosting |
| BucketPolicy | Bucket Policy | Public `s3:GetObject` for website visitors |

EC2 instances write to this bucket via their IAM role — no write permission is needed in the bucket policy.

### Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Data Stack - S3 bucket for static website hosting.

Resources:

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: 'my-unique-bucket-name-19-03'
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
      CorsConfiguration:
        CorsRules:
          - AllowedHeaders: ['*']
            AllowedMethods: [GET, HEAD]
            AllowedOrigins: ['*']
            MaxAge: 3000
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-StaticWebsiteBucket'

  BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref MyS3Bucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: PublicReadGetObject
            Effect: Allow
            Principal: '*'
            Action: 's3:GetObject'
            Resource: !Sub 'arn:aws:s3:::${MyS3Bucket}/*'

Outputs:
  BucketName:
    Value: !Ref MyS3Bucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'

  BucketArn:
    Value: !GetAtt MyS3Bucket.Arn
    Export:
      Name: !Sub '${AWS::StackName}-BucketArn'

  WebsiteURL:
    Value: !GetAtt MyS3Bucket.WebsiteURL
    Export:
      Name: !Sub '${AWS::StackName}-WebsiteURL'
```

---

## Stack 5 — Parent Stack (`parent-stack.yaml`)

**The only stack you deploy directly.**

The parent stack wires all child stacks together. It reads outputs from each child using `!GetAtt` and passes them as parameters to the next dependent child.

### Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Parent Stack - Deploys all nested stacks in dependency order.

Parameters:
  TemplateBaseURL:
    Type: String
    Description: >
      S3 URL prefix where child templates are stored (no trailing slash).
      Example: https://s3.amazonaws.com/my-cfn-templates/nested

  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]

  MinSize:
    Type: Number
    Default: 1

  MaxSize:
    Type: Number
    Default: 4

  DesiredCapacity:
    Type: Number
    Default: 2

Resources:

  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub '${TemplateBaseURL}/network-stack.yaml'
      TimeoutInMinutes: 15
      Tags:
        - Key: ManagedBy
          Value: ParentStack

  DataStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub '${TemplateBaseURL}/data-stack.yaml'
      TimeoutInMinutes: 10
      Tags:
        - Key: ManagedBy
          Value: ParentStack

  ALBStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: !Sub '${TemplateBaseURL}/alb-stack.yaml'
      TimeoutInMinutes: 15
      Parameters:
        VPCId:              !GetAtt NetworkStack.Outputs.VPCId
        PublicSubnet1Id:    !GetAtt NetworkStack.Outputs.PublicSubnet1Id
        PublicSubnet2Id:    !GetAtt NetworkStack.Outputs.PublicSubnet2Id
        ALBSecurityGroupId: !GetAtt NetworkStack.Outputs.ALBSecurityGroupId
      Tags:
        - Key: ManagedBy
          Value: ParentStack

  ComputeStack:
    Type: AWS::CloudFormation::Stack
    DependsOn:
      - NetworkStack
      - ALBStack
      - DataStack
    Properties:
      TemplateURL: !Sub '${TemplateBaseURL}/compute-stack.yaml'
      TimeoutInMinutes: 20
      Parameters:
        PrivateSubnet1Id:   !GetAtt NetworkStack.Outputs.PrivateSubnet1Id
        PrivateSubnet2Id:   !GetAtt NetworkStack.Outputs.PrivateSubnet2Id
        EC2SecurityGroupId: !GetAtt NetworkStack.Outputs.EC2SecurityGroupId
        TargetGroupArn:     !GetAtt ALBStack.Outputs.TargetGroupArn
        S3BucketName:       !GetAtt DataStack.Outputs.BucketName
        S3BucketArn:        !GetAtt DataStack.Outputs.BucketArn
        InstanceType:       !Ref InstanceType
        MinSize:            !Ref MinSize
        MaxSize:            !Ref MaxSize
        DesiredCapacity:    !Ref DesiredCapacity
      Tags:
        - Key: ManagedBy
          Value: ParentStack

Outputs:
  ALBDNSName:
    Description: Access the Nginx app at http://<this value>
    Value: !GetAtt ALBStack.Outputs.ALBDNSName

  S3WebsiteURL:
    Value: !GetAtt DataStack.Outputs.WebsiteURL

  VPCId:
    Value: !GetAtt NetworkStack.Outputs.VPCId

  AutoScalingGroupName:
    Value: !GetAtt ComputeStack.Outputs.AutoScalingGroupName
```

---

## How to Deploy

### Step 1 — Upload templates to S3

```bash
aws s3 mb s3://my-cfn-templates

aws s3 cp network-stack.yaml  s3://my-cfn-templates/nested/
aws s3 cp alb-stack.yaml      s3://my-cfn-templates/nested/
aws s3 cp compute-stack.yaml  s3://my-cfn-templates/nested/
aws s3 cp data-stack.yaml     s3://my-cfn-templates/nested/
```

### Step 2 — Deploy the parent stack

```bash
aws cloudformation deploy \
  --template-file parent-stack.yaml \
  --stack-name my-app-stack \
  --parameter-overrides \
      TemplateBaseURL=https://s3.amazonaws.com/my-cfn-templates/nested \
      InstanceType=t3.micro \
      DesiredCapacity=2 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Step 3 — Get the ALB DNS name

```bash
aws cloudformation describe-stacks \
  --stack-name my-app-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`ALBDNSName`].OutputValue' \
  --output text
```

Open `http://<ALBDNSName>` in a browser to see the Nginx page.

### Step 4 — Connect to an instance via SSM (no SSH needed)

```bash
# List running instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=*NginxInstance*" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text

# Start a session
aws ssm start-session --target i-0abc1234def56789
```

---

## Bugs Fixed During Development

| Template | Bug | Fix |
|---|---|---|
| network-stack | `Vpcid` (wrong case) on public subnets | Changed to `VpcId` |
| network-stack | Public subnets not assigning public IPs | Added `MapPublicIpOnLaunch: true` |
| network-stack | No Outputs section | Added all 8 exports |
| alb-stack | `Name` exceeding 32 char ALB limit | Removed `Name` property from ALB and TG |
| alb-stack | `Fn::ImportValue` with dynamic key | Replaced with direct Parameters from parent |
| compute-stack | `yum install unzip` on AL2023 | Changed to `dnf install unzip` |
| compute-stack | `${!Ref S3BucketName}` wrong syntax in `!Sub` | Changed to `${S3BucketName}` |
| compute-stack | No S3 write permission on IAM role | Added `S3UploadPolicy` inline policy |

---

## Key Concepts Learned

**Nested stacks vs flat stacks** — a flat stack puts all resources in one template which becomes hard to manage at scale. Nested stacks split resources by concern (network, compute, data) and allow each stack to be developed, tested, and redeployed independently.

**`!GetAtt` over `Fn::ImportValue`** — `Fn::ImportValue` requires a static export name. When nested stacks have auto-generated suffixes in their names, building the export key dynamically from a parameter is fragile and breaks silently. `!GetAtt Stack.Outputs.Key` is resolved by the parent at deploy time and is always accurate.

**ALB name length limit** — AWS enforces a 32 character limit on ALB and Target Group names. `${AWS::StackName}` inside a nested stack expands to the full generated name which easily exceeds this. Omit the `Name` property and use `Tags` for console identification instead.


**IMDSv2** — enforcing `HttpTokens: required` in the launch template blocks unauthenticated metadata requests. All metadata calls must first obtain a session token via a `PUT` request. This prevents SSRF exploits from reading IAM credentials from the metadata service.

**SSM Session Manager** — by attaching `AmazonSSMManagedInstanceCore` to the IAM role, instances in private subnets can be accessed via the AWS console or CLI without a bastion host, VPN, or open SSH port. The NAT gateway provides outbound internet access so the SSM agent can reach the SSM endpoint.