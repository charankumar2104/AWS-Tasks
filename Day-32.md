#  Day-32 Lab

## Topic: AWS CloudFormation Templates (CFT), EC2 Provisioning, IAM Integration, and S3 Access

---

##  Objective

In this lab we explored **Infrastructure as Code (IaC)** using **AWS CloudFormation**.

We created two templates:

- **Task-1:** Deploy a complete infrastructure (VPC + EC2 + Networking + User configuration)
- **Task-2:** Deploy EC2 with **IAM Role and S3 access**

Additionally, we understood:

- CloudFormation concepts
- Stack creation workflow
- Template options and configurations

---

##  What is AWS CloudFormation?

AWS **CloudFormation** is a service that allows you to define and provision AWS infrastructure using **code (YAML or JSON)**. Instead of manually creating resources, you define everything in a **template file**.

### Why CloudFormation Exists

Before IaC, teams faced:

- Manual resource creation
- Human errors
- Inconsistent environments
- Difficult to replicate setups

CloudFormation solves these problems by following this flow:

```text
Define → Deploy → Reuse → Automate
```

### Benefits

| Benefit                | Description                   |
| ---------------------- | ----------------------------- |
| Infrastructure as Code | Define resources in templates |
| Automation             | No manual setup               |
| Consistency            | Same environment every time   |
| Version Control        | Track infrastructure changes  |
| Rollback               | Automatic rollback on failure |

---

##  CloudFormation Template Structure

A typical template contains:

| Section                  | Purpose                        |
| ------------------------ | ------------------------------ |
| AWSTemplateFormatVersion | Template version               |
| Description              | Info about template            |
| Parameters               | User inputs                    |
| Mappings                 | Region-specific values         |
| Conditions               | Conditional resource creation  |
| Resources                | AWS resources (required)       |
| Outputs                  | Output values                  |

### Intrinsic Functions

| Function  | Purpose                            |
| --------- | ---------------------------------- |
| `!Ref`    | Reference a resource or parameter  |
| `!GetAtt` | Get an attribute from a resource   |
| `!Sub`    | String substitution                |
| `!Join`   | Join multiple values               |

---

##  Task-1: EC2 with VPC and Networking

### Architecture

```text
VPC
 │
 ├── Subnet
 │     │
 │     ▼
 │   EC2 Instance
 │
 ├── Internet Gateway
 │
 ├── Route Table
 │
 └── Security Group
```

### Features Implemented

- Custom **VPC** — CIDR `192.168.0.0/20`, DNS enabled, isolated network
- Public **Subnet** — CIDR `192.168.0.0/24`, auto-assign public IP
- **Internet Gateway** — connects VPC to the internet
- **Route Table** — routes traffic from subnet to internet gateway
- **Security Group** — allows SSH (port 22) and HTTP (port 80)
- **EC2 Instance** — custom AMI, parameterized instance type, 20GB EBS volume
- **User Data Script** — automates user creation, password setup, SSH configuration changes

### Template Code

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'AWS CloudFormation Template: EC2 Instance'

Parameters:
  AMIId:
    Type: String
    Default: ami-02dfbd4ff395f2a1b
    Description: The AMI ID for the EC2 instance

  InstanceType:
    Type: String
    Default: t3.small
    Description: The instance type for the EC2 instance (default is t2.micro)

Resources:
  taskVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/20
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: taskVpc

  taskSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.0.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: taskSubnet

  taskInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: taskIGW

  taskVPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref taskVpc
      InternetGatewayId: !Ref taskInternetGateway

  taskRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc
      Tags:
        - Key: Name
          Value: taskRouteTable

  taskRoute:
    Type: AWS::EC2::Route
    DependsOn: taskVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref taskRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref taskInternetGateway

  taskSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref taskSubnet
      RouteTableId: !Ref taskRouteTable

  taskSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP access
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: taskSecurityGroup

  taskEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AMIId
      InstanceType: !Ref InstanceType
      SubnetId: !Ref taskSubnet
      SecurityGroupIds:
        - !Ref taskSecurityGroup
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 20
      Tags:
        - Key: Name
          Value: taskEC2Instance
      UserData: !Base64 |
        #!/bin/bash

        USERNAME="testuser"
        PASSWORD="Temp@123"
        FILE="/etc/ssh/sshd_config"

        # Create user with home directory
        sudo useradd -m $USERNAME

        # Set password
        echo "$USERNAME:$PASSWORD" | sudo chpasswd

        # Force password change on first login
        # sudo chage -d 0 $USERNAME

        # Change PermitRootLogin
        sudo sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' $FILE

        # Enable PasswordAuthentication
        sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' $FILE

        # Enable PAM
        sudo sed -i 's/^#UsePAM no/UsePAM yes/' $FILE

        # Restart SSH service
        sudo systemctl restart sshd

Outputs:
  publicIP:
    Description: Public IP address of the EC2 instance
    Value: !GetAtt taskEC2Instance.PublicIp
```

### Output

```text
Public IP of EC2 instance
```

---

##  Task-2: EC2 with S3 Access using IAM Role

### Architecture

```text
EC2 Instance
     │
     ▼
IAM Role (Instance Profile)
     │
     ▼
S3 Bucket
```

### Features Implemented

- **S3 Bucket** — created using CloudFormation, used for storage access
- **IAM Role** — allows EC2 to call `s3:ListBucket` (least privilege principle)
- **Instance Profile** — wraps the IAM Role so EC2 can attach it
- **EC2 Instance** — uses the IAM role, accesses S3 without hardcoded credentials

### Template Code

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'AWS CloudFormation Template: EC2 Instance'

Parameters:
  AMIId:
    Type: String
    Default: ami-02dfbd4ff395f2a1b
    Description: The AMI ID for the EC2 instance

  InstanceType:
    Type: String
    Default: t3.small
    Description: The instance type for the EC2 instance (default is t2.micro)

Resources:
  taskVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/20
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: taskVpc

  taskSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.0.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: taskSubnet

  taskInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: taskIGW

  taskVPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref taskVpc
      InternetGatewayId: !Ref taskInternetGateway

  taskRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc
      Tags:
        - Key: Name
          Value: taskRouteTable

  taskRoute:
    Type: AWS::EC2::Route
    DependsOn: taskVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref taskRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref taskInternetGateway

  taskSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref taskSubnet
      RouteTableId: !Ref taskRouteTable

  taskSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP access
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: taskSecurityGroup

  taskS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: task-s3-bucket-17-03-2026
  
  taskEC2S3AccessRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: S3ListPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:ListBucket
                Resource: !Sub arn:aws:s3:::${taskS3Bucket}

  taskInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref taskEC2S3AccessRole


  taskEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AMIId
      InstanceType: !Ref InstanceType
      SubnetId: !Ref taskSubnet
      SecurityGroupIds:
        - !Ref taskSecurityGroup
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 20
      Tags:
        - Key: Name
          Value: taskEC2Instance
      IamInstanceProfile: !Ref taskInstanceProfile
      UserData: !Base64 |
        #!/bin/bash

        USERNAME="awsuser"
        PASSWORD="Awsuser@123"
        FILE="/etc/ssh/sshd_config"

        # Create user with home directory
        sudo useradd -m $USERNAME

        # Set password
        echo "$USERNAME:$PASSWORD" | sudo chpasswd

        # Force password change on first login
        # sudo chage -d 0 $USERNAME

        # Change PermitRootLogin
        sudo sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' $FILE

        # Enable PasswordAuthentication
        sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' $FILE

        # Enable PAM
        sudo sed -i 's/^#UsePAM no/UsePAM yes/' $FILE

        # Restart SSH service
        sudo systemctl restart sshd

Outputs:
  publicIP:
    Description: Public IP address of the EC2 instance
    Value: !GetAtt taskEC2Instance.PublicIp
```

---

##  CloudFormation Stack Creation — Step by Step

### Step 1 — Navigate to CloudFormation

```text
AWS Console → CloudFormation
```

### Step 2 — Create Stack

| Option                  | Description                       |
| ----------------------- | --------------------------------- |
| With new resources      | Create stack using a new template |
| With existing resources | Import already existing resources |

### Step 3 — Upload Template

```text
Upload a local file
Enter template inline
Use an S3 URL
```

### Step 4 — Specify Stack Details

| Field      | Example               |
| ---------- | --------------------- |
| Stack Name | task-stack            |
| Parameters | AMI ID, Instance Type |

### Step 5 — Configure Stack Options

Set tags, permissions, failure behavior, and advanced options.
*(See the detailed reference section below.)*

### Step 6 — Review and Create

Review all configuration and click **Create Stack**.

### Step 7 — Monitor Stack

| State              | Meaning             |
| ------------------ | ------------------- |
| CREATE_IN_PROGRESS | Stack is creating   |
| CREATE_COMPLETE    | Successfully done   |
| ROLLBACK_COMPLETE  | Failed, rolled back |

### Step 8 — Access Outputs

```text
Public IP
Resource IDs
```

---

##  CloudFormation Stack Options — Detailed Reference

This section explains every option available on the **Configure Stack Options** page when creating or updating a CloudFormation stack.

---

###  Tags

**What it is:** Key-value labels attached to your stack and all supported resources inside it.

**Where it is used:**
- Billing dashboards — group costs by team, project, or environment
- Resource searches — filter all resources belonging to a specific stack
- Access control — use tags in IAM policies to restrict who can modify certain resources

**Rules:**
- Up to **50 tags** per stack
- Key: alphanumeric, max 127 characters
- Value: alphanumeric, max 255 characters

| Key         | Value      |
| ----------- | ---------- |
| Environment | production |
| Team        | backend    |
| Project     | task-app   |

> Adding, updating, or removing tags after stack creation **triggers a stack update** and propagates to all resources that support tagging.

---

###  Permissions (IAM Service Role)

**What it is:** An IAM role that CloudFormation **assumes** to create, update, or delete resources — instead of using your own account credentials.

**Where it is used:**
- Organizations where developers have limited AWS permissions
- When CloudFormation needs broader permissions than your user account has
- Enforcing least privilege at the deployment level

**How it works:**

```text
Your Account (limited permissions)
        ↓
CloudFormation assumes the IAM Role
        ↓
Role has the permissions needed to create resources
        ↓
Resources are created using the Role's credentials
```

> If no role is specified, CloudFormation uses your current user or role credentials.

---

###  Stack Failure Options

**What it is:** Defines what CloudFormation does when a resource fails to create or update.

| Option | Description | Best For |
| ------ | ----------- | -------- |
| **Roll back all stack resources** | Undoes everything created in that operation and returns stack to previous clean state | Production environments |
| **Preserve successfully provisioned resources** | Keeps what succeeded; failed resources stay in a failed state until the next update | Debugging failed deployments |

---

###  Stack Policy *(Advanced)*

**What it is:** A JSON document that **prevents specific resources** from being accidentally modified or replaced during a stack update.

**Where it is used:**
- Protecting production databases from replacement
- Blocking critical resources from being deleted during routine updates

**Example — Block RDS replacement:**

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "Update:Replace",
      "Principal": "*",
      "Resource": "LogicalResourceId/MyDatabase"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    }
  ]
}
```

> By default (no policy), **all resources can be updated**. A stack policy only restricts update actions — it does not affect stack creation or deletion.

---

###  Rollback Configuration *(Advanced)*

**What it is:** Links **CloudWatch Alarms** to your stack. If any alarm enters an `ALARM` state during or after deployment, CloudFormation automatically rolls back.

**Where it is used:**
- CI/CD pipelines needing automatic rollback on degraded performance
- Production deployments where CPU spikes, error rate increases, or latency jumps should trigger a rollback

**How it works:**

```text
Stack update begins
        ↓
CloudFormation monitors CloudWatch Alarms
        ↓
Alarm enters ALARM state during deployment or monitoring window
        ↓
CloudFormation automatically rolls back the stack
```

| Setting                   | Description                                                   |
| ------------------------- | ------------------------------------------------------------- |
| Rollback triggers         | The CloudWatch Alarms to monitor                              |
| Monitoring time (minutes) | How long to monitor after deployment before declaring success |

---

###  Notification Options *(Advanced)*

**What it is:** An **Amazon SNS topic** that receives a notification for every stack event.

**Where it is used:**
- Email or SMS alerts when a stack starts, finishes, or fails
- Integration with Slack, PagerDuty, or other tools via SNS subscriptions
- Audit logging of every resource-level event

**Events that trigger notifications:**

```text
Stack creation started
Resource created / failed
Stack update started
Stack rollback triggered
Stack deletion completed
```

> You can create a new SNS topic directly in this option or select an existing one.

---

###  Timeout *(Stack Creation Only)*

**What it is:** A time limit (in minutes) for the entire stack creation process.

**Where it is used:**
- Preventing a stack from running indefinitely due to a hung resource
- Useful when provisioning resources that occasionally stall (e.g., custom resources, slow AMI initializations)

**Behavior:**
- If the stack is not created within the timeout, it **fails and rolls back** automatically
- By default, **there is no timeout**
- Individual resources may have their own timeouts regardless of this setting

---

###  Termination Protection *(Stack Creation Only)*

**What it is:** A safety lock that **prevents the stack from being accidentally deleted**.

**Where it is used:**
- All production stacks — should always be enabled
- Any stack managing critical data, databases, or network infrastructure

**Behavior:**
- If anyone runs `delete-stack` on a protected stack, **the deletion fails** with an error
- Stack and all resources remain unchanged
- Termination protection is **Disabled by default**

> To delete a protected stack, you must first **disable termination protection** in the stack settings, then delete.

---

###  DeletionPolicy and UpdateReplacePolicy *(In Template)*

These are **not console options** — they are properties written directly on each resource in your template to protect data during deletion or replacement.

| Property              | When It Applies                                                                  |
| --------------------- | -------------------------------------------------------------------------------- |
| `DeletionPolicy`      | When the stack is deleted or the resource is removed from the template           |
| `UpdateReplacePolicy` | When a stack update causes the resource to be replaced with a new one            |

>  You need **both** — `DeletionPolicy` alone does not protect during updates.

**Available values:**

| Value      | Behaviour                                               | Use For                             |
| ---------- | ------------------------------------------------------- | ----------------------------------- |
| `Delete`   | Resource and data permanently deleted (default)         | Non-production, throwaway resources |
| `Retain`   | Resource is kept as-is even after removal               | S3 buckets, most stateful resources |
| `Snapshot` | A backup is taken before the resource is deleted        | RDS, EBS, ElastiCache, Redshift     |

**Example:**

```yaml
MyS3Bucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain          
  UpdateReplacePolicy: Retain     
  Properties:
    BucketName: my-app-bucket

MyDatabase:
  Type: AWS::RDS::DBInstance
  DeletionPolicy: Snapshot        # Take a snapshot before deleting
  UpdateReplacePolicy: Snapshot   # Take a snapshot before replacing during update
  Properties:
    DBInstanceIdentifier: my-app-db
    Engine: mysql
    DBInstanceClass: db.t3.micro
    AllocatedStorage: 20
```

>  Both properties sit at the **same level as `Type` and `Properties`** — never inside `Properties`.

---

###  All Stack Options — Quick Reference

| Option                 | Where Set | Purpose                                          | Best Used For                           |
| ---------------------- | --------- | ------------------------------------------------ | --------------------------------------- |
| Tags                   | Console   | Cost tracking and resource organization          | All stacks                              |
| Permissions (IAM Role) | Console   | Use a scoped role instead of user credentials    | Multi-team or restricted environments   |
| Stack Failure Options  | Console   | Rollback vs preserve on failure                  | Production = Rollback, Debug = Preserve |
| Stack Policy           | Console   | Block Replace/Delete on specific resources       | Protecting critical resources           |
| Rollback Configuration | Console   | Auto-rollback on CloudWatch alarm                | Production CI/CD pipelines              |
| Notification Options   | Console   | Send stack events to SNS                         | Team alerts and audit logging           |
| Timeout                | Console   | Stop hung stack creation automatically           | Avoiding indefinitely stuck stacks      |
| Termination Protection | Console   | Prevent accidental stack deletion                | Always enable for production            |
| DeletionPolicy         | Template  | Protect resource data on stack deletion          | S3 (Retain), RDS (Snapshot)             |
| UpdateReplacePolicy    | Template  | Protect resource data when replaced in an update | S3 (Retain), RDS (Snapshot)             |

---