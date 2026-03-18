# 📘 Day-33 Lab

## Topic: Advanced AWS CloudFormation (CFT) – Multi-Tier Architecture, Conditional Deployments, Mappings, and Metadata

---

#  Objective

In this lab we explored **advanced CloudFormation concepts** by building two powerful templates:

###  Task-1

* Create a **3-tier architecture**

  * VPC
  * Public EC2 (Web Layer)
  * Private RDS (Database Layer)
* Secure DB access using **Security Groups**
* Use **Parameters for sensitive data (NoEcho)**



###  Task-2

* Create **Linux / Windows / Both EC2 instances dynamically**
* Use **Conditions, Mappings, and Metadata**
* Configure **user-based login with username/password**
* Deploy **web servers automatically using User Data**

Usage:

```yaml
!FindInMap [AMIIdMapping, Linux, AMIId]
```

Benefits:

* Region-specific configurations
* OS-based configurations
* Cleaner templates

---

#  Conditions

Conditions allow **conditional resource creation**.

Example:

```yaml
IsLinux: !Equals [!Ref OSType, Linux]
```

Used with:

```yaml
Condition: IsLinux
```

Benefits:

* Deploy resources dynamically
* Reduce template duplication
* Flexible deployments

---

#  Metadata

Metadata enhances **UI experience in CloudFormation Console**.

Example:

```yaml
AWS::CloudFormation::Interface:
```

Features:

* Group parameters
* Label parameters
* Improve readability

Benefits:

* User-friendly interface
* Organized parameter input

---

#  Task-1: Multi-Tier Architecture (EC2 + RDS)

---

# Architecture Overview

```text
Internet
   │
   ▼
Public Subnet (EC2)
   │
   ▼
Private Subnets (RDS)
```

---

# Components Created

| Resource         | Purpose           |
| ---------------- | ----------------- |
| VPC              | Network isolation |
| Public Subnet    | Web server        |
| Private Subnets  | Database layer    |
| Internet Gateway | Internet access   |
| Route Tables     | Traffic routing   |
| Security Groups  | Access control    |
| EC2 Instance     | Web layer         |
| RDS Instance     | Database layer    |

---

# Security Design

---

## Web Security Group

Allows:

* SSH (22) → Public
* HTTP (80) → Public

---

## DB Security Group

Allows:

```text
MySQL (3306) only from Web Security Group
```

This ensures:

✔ No public DB access
✔ Secure internal communication

---

# RDS Configuration

* Engine: MySQL 8.0
* Subnet Group: Private subnets
* Public Access: Disabled
* Credentials via Parameters

---

# Outputs

Provides:

```text
EC2 Public IP
RDS Endpoint
RDS Port
```
---
``` yaml
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
    Description: The instance type for the EC2 instance
  
  DBUsername:                          
    Type: String
    Default: admin
    Description: Master username for RDS MySQL  

  DBPassword:                         
    Type: String
    NoEcho: true
    Description: Master password for RDS MySQL

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

  taskPublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.0.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: taskPublicSubnet

  taskPrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: taskPrivateSubnet-1

  taskPrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: taskPrivateSubnet-2

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

  taskPublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc
      Tags:
        - Key: Name
          Value: taskPublicRouteTable

  taskPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: taskVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref taskPublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref taskInternetGateway

  taskPublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref taskPublicSubnet
      RouteTableId: !Ref taskPublicRouteTable

  taskPrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc
      Tags:
        - Key: Name
          Value: taskPrivateRouteTable

  taskPrivate1SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref taskPrivateRouteTable
      SubnetId: !Ref taskPrivateSubnet1    

  taskPrivate2SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref taskPrivateRouteTable
      SubnetId: !Ref taskPrivateSubnet2

  taskWebSecurityGroup:
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
          Value: web-sg

  taskDBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow MySQL access from web security group
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref taskWebSecurityGroup
      Tags:
        - Key: Name
          Value: db-sg

  taskEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AMIId
      InstanceType: !Ref InstanceType
      SubnetId: !Ref taskPublicSubnet
      SecurityGroupIds:
        - !Ref taskWebSecurityGroup
      Tags:
        - Key: Name
          Value: taskEC2Instance

  taskDBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Subnet group for RDS instance
      SubnetIds:                        
        - !Ref taskPrivateSubnet1
        - !Ref taskPrivateSubnet2
      Tags:
        - Key: Name
          Value: taskDBSubnetGroup

  taskRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: taskRDSInstance
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      Engine: mysql
      EngineVersion: "8.0"
      MasterUsername: !Ref DBUsername  
      MasterUserPassword: !Ref DBPassword  
      VPCSecurityGroups:
        - !Ref taskDBSecurityGroup
      DBSubnetGroupName: !Ref taskDBSubnetGroup
      MultiAZ: false
      PubliclyAccessible: false


Outputs:
  EC2PublicIP:
    Description: Public IP address of the EC2 instance
    Value: !GetAtt taskEC2Instance.PublicIp

  RDSEndpointAddress:
    Description: Connection endpoint hostname for the RDS MySQL instance
    Value: !GetAtt taskRDSInstance.Endpoint.Address

  RDSEndpointPort:
    Description: Port for the RDS MySQL instance
    Value: !GetAtt taskRDSInstance.Endpoint.Port
```
---

# Key Learning from Task-1

✔ Multi-tier architecture using CloudFormation
✔ Secure DB deployment in private subnet
✔ Parameterized credentials
✔ Security group referencing

---

#  Task-2: Conditional EC2 Deployment (Linux / Windows / Both)

---

# Architecture Overview

```text
VPC
 │
 └── Public Subnet
       │
       ├── Linux EC2 (optional)
       └── Windows EC2 (optional)
```

---

# Parameters Used

| Parameter        | Purpose                       |
| ---------------- | ----------------------------- |
| OSType           | Select Linux / Windows / Both |
| InstanceType     | Instance size                 |
| AvailabilityZone | Deployment zone               |
| MyIP             | Restrict access               |
| Credentials      | Username & Password           |

---

# Conditional Logic

---

## Linux Condition

```yaml
IsLinux:
  !Or
    - !Equals [!Ref OSType, Linux]
    - !Equals [!Ref OSType, Both]
```

---

## Windows Condition

```yaml
IsWindows:
  !Or
    - !Equals [!Ref OSType, Windows]
    - !Equals [!Ref OSType, Both]
```

---

# Conditional Resource Creation

Example:

```yaml
Condition: IsLinux
```

This ensures:

* Only Linux EC2 is created when selected
* Both instances created when "Both" selected

---

# Linux EC2 Configuration

* Uses Amazon Linux AMI
* Installs **Apache (httpd)**
* Enables password-based SSH login
* Creates custom user

---

# Windows EC2 Configuration

* Uses Windows Server AMI
* Installs **IIS Web Server**
* Creates admin user
* Enables RDP access
* Configures firewall rules

---

# Security Groups

---

## Linux SG

* SSH → Only from MyIP
* HTTP → Open to all

---

## Windows SG

* RDP → Only from MyIP
* HTTP → Open to all

---

# User Data Automation

---

## Linux Script

```text
Install Apache
Start service
Create user
Enable password login
```

---

## Windows Script

```text
Install IIS
Create admin user
Enable RDP
Configure firewall
```

---

# Outputs

| Output          | Description         |
| --------------- | ------------------- |
| LinuxPublicIP   | Linux instance IP   |
| LinuxWebURL     | Apache URL          |
| WindowsPublicIP | Windows instance IP |
| WindowsWebURL   | IIS URL             |

---
``` yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS CloudFormation Template for Linux and Windows EC2 with username/password login

Parameters:
  OSType:
    Type: String
    AllowedValues:
      - Linux
      - Windows
      - Both
    Description: Select the OS type - Linux, Windows, or Both
    Default: Linux

  InstanceType:
    Type: String
    Default: t3.small
    AllowedValues:
      - t3.micro
      - t3.small
      - m7i-flex.large              
    Description: EC2 instance type

  AvailabilityZone:
    Type: AWS::EC2::AvailabilityZone::Name   
    Description: Select the AZ to launch instances in (pick one that supports your instance type)

  MyIP:
    Type: String
    Description: Your public IP in CIDR format e.g. 203.0.113.10/32

  LinuxUsername:
    Type: String
    Default: testuser
    Description: Username to create on the Linux instance

  LinuxPassword:
    Type: String
    NoEcho: true
    Description: Password for the Linux user

  WindowsUsername:
    Type: String
    Default: testuser
    Description: New local user to create on the Windows instance

  WindowsPassword:
    Type: String
    NoEcho: true
    Description: Password for the Windows user

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: OS Selection
        Parameters:
          - OSType
          - InstanceType
          - AvailabilityZone        
          - MyIP
      - Label:
          default: Linux Credentials
        Parameters:
          - LinuxUsername
          - LinuxPassword
      - Label:
          default: Windows Credentials
        Parameters:
          - WindowsUsername
          - WindowsPassword
    ParameterLabels:
      OSType:
        default: Operating System Type
      InstanceType:
        default: Instance Type
      AvailabilityZone:
        default: Availability Zone (pick one that supports your instance type)
      MyIP:
        default: Your IP Address (CIDR)
      LinuxUsername:
        default: Linux Username
      LinuxPassword:
        default: Linux Password
      WindowsUsername:
        default: Windows Username
      WindowsPassword:
        default: Windows Password

Mappings:
  AMIIdMapping:
    Linux:
      AMIId: ami-02dfbd4ff395f2a1b
    Windows:
      AMIId: ami-01a15dfc48279bf55

Conditions:
  IsLinux: !Or
    - !Equals [!Ref OSType, Linux]
    - !Equals [!Ref OSType, Both]

  IsWindows: !Or
    - !Equals [!Ref OSType, Windows]
    - !Equals [!Ref OSType, Both]

Resources:
  taskVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: taskVpc

  taskPublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.0.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Ref AvailabilityZone  
      Tags:
        - Key: Name
          Value: taskPublicSubnet

  taskInternetGateway:
    Type: AWS::EC2::InternetGateway

  taskVPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref taskVpc
      InternetGatewayId: !Ref taskInternetGateway

  taskPublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc

  taskPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: taskVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref taskPublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref taskInternetGateway

  taskSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref taskPublicSubnet
      RouteTableId: !Ref taskPublicRouteTable

  taskLinuxSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Condition: IsLinux
    Properties:
      GroupDescription: Linux SG - SSH from MyIP only, HTTP open to all
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref MyIP
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: taskLinuxSG

  taskWindowsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Condition: IsWindows
    Properties:
      GroupDescription: Windows SG - RDP from MyIP only, HTTP open to all
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3389
          ToPort: 3389
          CidrIp: !Ref MyIP
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: taskWindowsSG

  taskLinuxEC2:
    Type: AWS::EC2::Instance
    Condition: IsLinux
    Properties:
      ImageId: !FindInMap [AMIIdMapping, Linux, AMIId]
      InstanceType: !Ref InstanceType
      AvailabilityZone: !Ref AvailabilityZone   # ✅ instance pinned to selected AZ
      SubnetId: !Ref taskPublicSubnet
      SecurityGroupIds:
        - !Ref taskLinuxSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Linux Web Server</h1>" > /var/www/html/index.html

          USERNAME="${LinuxUsername}"
          PASSWORD="${LinuxPassword}"
          FILE="/etc/ssh/sshd_config"

          useradd -m $USERNAME
          echo "$USERNAME:$PASSWORD" | chpasswd

          sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' $FILE
          sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' $FILE
          sed -i 's/^#UsePAM no/UsePAM yes/' $FILE

          systemctl restart sshd
      Tags:
        - Key: Name
          Value: taskLinuxEC2

  taskWindowsEC2:
    Type: AWS::EC2::Instance
    Condition: IsWindows
    Properties:
      ImageId: !FindInMap [AMIIdMapping, Windows, AMIId]
      InstanceType: !Ref InstanceType
      AvailabilityZone: !Ref AvailabilityZone   
      SubnetId: !Ref taskPublicSubnet
      SecurityGroupIds:
        - !Ref taskWindowsSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          <powershell>
          Install-WindowsFeature -Name Web-Server -IncludeManagementTools
          Set-Content -Path "C:\inetpub\wwwroot\index.html" `
            -Value "<h1>Windows IIS Server</h1>"

          $username = "${WindowsUsername}"
          $password = ConvertTo-SecureString "${WindowsPassword}" `
            -AsPlainText -Force

          New-LocalUser -Name $username `
                        -Password $password `
                        -FullName $username `
                        -PasswordNeverExpires

          Add-LocalGroupMember -Group "Administrators" -Member $username
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member $username

          Set-ItemProperty `
            -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" `
            -Name "fDenyTSConnections" -Value 0

          Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
          </powershell>
      Tags:
        - Key: Name
          Value: taskWindowsEC2

Outputs:

  LinuxPublicIP:
    Condition: IsLinux
    Description: Linux EC2 public IP
    Value: !GetAtt taskLinuxEC2.PublicIp

  LinuxSSHCommand:
    Condition: IsLinux
    Description: SSH command to connect to Linux
    Value: !Sub "ssh ${LinuxUsername}@${taskLinuxEC2.PublicIp}"

  LinuxWebURL:
    Condition: IsLinux
    Description: Linux web server URL (httpd)
    Value: !Sub "http://${taskLinuxEC2.PublicIp}"

  WindowsPublicIP:
    Condition: IsWindows
    Description: Windows EC2 public IP
    Value: !GetAtt taskWindowsEC2.PublicIp

  WindowsWebURL:
    Condition: IsWindows
    Description: Windows web server URL (IIS)
    Value: !Sub "http://${taskWindowsEC2.PublicIp}"
```

---

#  CloudFormation Advanced Concepts

---

# 🔹 Parameters

Parameters allow dynamic input at runtime.

Example:

```yaml
DBPassword:
  Type: String
  NoEcho: true
```

Benefits:

* Avoid hardcoding values
* Improve security
* Reusable templates

---

#  Mappings

Mappings allow defining **static key-value pairs**.

Example:

```yaml
AMIIdMapping:
  Linux:
    AMIId: ami-xxxx
  Windows:
    AMIId: ami-yyyy
```
---

# 🔥 Why These Features Matter

---

# Why Use Conditions?

Without conditions:

* Need multiple templates
* Hard to manage

With conditions:

```text
Single Template → Multiple Scenarios
```

---

# Why Use Mappings?

Without mappings:

* Hardcoded AMIs
* Less flexibility

With mappings:

```text
Dynamic AMI Selection
```

---

# Why Use Metadata?

Without metadata:

* Poor UI experience

With metadata:

```text
Organized and User-Friendly Inputs
```

---