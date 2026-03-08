# Day-23 Lab

## Topic: Advanced IAM Policy for EC2 Control, IAM Role Governance, and Region-Based Access Control

---

# Objective

In this lab we implemented a **complex IAM policy** that enforces multiple governance rules in AWS.

The policy was designed to:

* Restrict operations to specific **AWS regions**
* Allow **controlled EC2 instance creation**
* Restrict **instance types**
* Limit **EBS volume size and type**
* Enforce **IAM permission boundaries**
* Control **IAM role creation and management**
* Allow **read-only access to multiple AWS services**

This type of policy is commonly used in **enterprise cloud governance environments**.

---

# Key Governance Rules Implemented

The policy enforces the following restrictions:

| Control Area          | Restriction                                |
| --------------------- | ------------------------------------------ |
| Region Access         | Only `us-east-1` and `us-east-2` allowed   |
| EC2 Instance Creation | Allowed only in `us-east-1`                |
| Instance Types        | Only `t3.micro` and `t3.nano`              |
| EBS Volume            | Only `gp2` volumes up to **30GB**          |
| IAM Roles             | Must include **Permission Boundary**       |
| Role Naming           | Only roles containing **Lambda** or **S3** |
| Role Policies         | Only Lambda or S3 related policies         |
| PassRole              | Only Lambda/S3 roles to EC2                |
| Other Services        | Read-only access                           |

---

# Task: Create and Attach the Governance Policy

---

## Step-1: Navigate to IAM

Open AWS Console:

```
IAM → Policies
```

Click:

```
Create Policy
```

Choose:

```
JSON Editor
```

---

## Step-2: Add the IAM Policy

Paste the following policy JSON.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EC2ReadBothRegions",
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:Get*",
                "ec2:List*"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": [
                        "us-east-1",
                        "us-east-2"
                    ]
                }
            }
        },
        {
            "Sid": "EC2RunInstancesOnInstance",
            "Effect": "Allow",
            "Action": "ec2:RunInstances",
            "Resource": "arn:aws:ec2:us-east-1:*:instance/*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "us-east-1",
                    "ec2:InstanceType": [
                        "t3.micro",
                        "t3.nano"
                    ]
                }
            }
        },
        {
            "Sid": "EC2RunInstancesOnVolume",
            "Effect": "Allow",
            "Action": "ec2:RunInstances",
            "Resource": "arn:aws:ec2:us-east-1:*:volume/*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "us-east-1"
                }
            }
        },
        {
            "Sid": "EC2RunInstancesOnSupportingResources",
            "Effect": "Allow",
            "Action": "ec2:RunInstances",
            "Resource": [
                "arn:aws:ec2:us-east-1::image/*",
                "arn:aws:ec2:us-east-1:*:subnet/*",
                "arn:aws:ec2:us-east-1:*:network-interface/*",
                "arn:aws:ec2:us-east-1:*:security-group/*",
                "arn:aws:ec2:us-east-1:*:key-pair/*"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "us-east-1"
                }
            }
        },
        {
            "Sid": "EC2VolumeCreateUSEast1Constrained",
            "Effect": "Allow",
            "Action": "ec2:CreateVolume",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "us-east-1",
                    "ec2:VolumeType": "gp2"
                },
                "NumericLessThanEquals": {
                    "ec2:VolumeSize": "30"
                }
            }
        },
        {
            "Sid": "EC2ManageExistingResourcesUSEast1",
            "Effect": "Allow",
            "Action": [
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:RebootInstances",
                "ec2:TerminateInstances",
                "ec2:AttachVolume",
                "ec2:DetachVolume",
                "ec2:DeleteVolume",
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:ModifyInstanceAttribute",
                "ec2:AssociateIamInstanceProfile",
                "ec2:DisassociateIamInstanceProfile",
                "ec2:ReplaceIamInstanceProfileAssociation"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "us-east-1"
                }
            }
        },
        {
            "Sid": "IAMRoleManagementForLambdaAndS3",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:UpdateRole",
                "iam:GetRole",
                "iam:ListRoles",
                "iam:TagRole",
                "iam:UntagRole",
                "iam:UpdateAssumeRolePolicy",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRolePolicy",
                "iam:ListRolePolicies",
                "iam:ListAttachedRolePolicies"
            ],
            "Resource": [
                "arn:aws:iam::*:role/*Lambda*",
                "arn:aws:iam::*:role/*lambda*",
                "arn:aws:iam::*:role/*S3*",
                "arn:aws:iam::*:role/*s3*"
            ],
            "Condition": {
                "StringEquals": {
                    "iam:PermissionsBoundary": "arn:aws:iam::590716168639:policy/LambdaS3PermissionBoundary"
                }
            }
        },
        {
            "Sid": "DenyRoleCreationWithoutBoundary",
            "Effect": "Deny",
            "Action": "iam:CreateRole",
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "iam:PermissionsBoundary": "arn:aws:iam::590716168639:policy/LambdaS3PermissionBoundary"
                }
            }
        },
        {
            "Sid": "DenyRemovingPermissionBoundary",
            "Effect": "Deny",
            "Action": "iam:DeleteRolePermissionsBoundary",
            "Resource": "*"
        },
        {
            "Sid": "IAMPolicyAttachDetachForLambdaAndS3Roles",
            "Effect": "Allow",
            "Action": [
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy"
            ],
            "Resource": [
                "arn:aws:iam::*:role/*Lambda*",
                "arn:aws:iam::*:role/*lambda*",
                "arn:aws:iam::*:role/*S3*",
                "arn:aws:iam::*:role/*s3*"
            ],
            "Condition": {
                "ArnLike": {
                    "iam:PolicyARN": [
                        "arn:aws:iam::aws:policy/AWSLambda*",
                        "arn:aws:iam::aws:policy/AmazonS3*",
                        "arn:aws:iam::*:policy/*Lambda*",
                        "arn:aws:iam::*:policy/*lambda*",
                        "arn:aws:iam::*:policy/*S3*",
                        "arn:aws:iam::*:policy/*s3*"
                    ]
                }
            }
        },
        {
            "Sid": "IAMPolicyReadAndCreate",
            "Effect": "Allow",
            "Action": [
                "iam:CreatePolicy",
                "iam:GetPolicy",
                "iam:ListPolicies",
                "iam:CreatePolicyVersion",
                "iam:GetPolicyVersion",
                "iam:ListPolicyVersions"
            ],
            "Resource": "*"
        },
        {
            "Sid": "PassRoleLambdaS3RolesToEC2",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": [
                "arn:aws:iam::*:role/*Lambda*",
                "arn:aws:iam::*:role/*lambda*",
                "arn:aws:iam::*:role/*S3*",
                "arn:aws:iam::*:role/*s3*"
            ],
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "ec2.amazonaws.com"
                }
            }
        },
        {
            "Sid": "ReadOnlyAllOtherServicesBothRegions",
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*",
                "lambda:Get*",
                "lambda:List*",
                "rds:Describe*",
                "rds:List*",
                "dynamodb:Get*",
                "dynamodb:List*",
                "dynamodb:Describe*",
                "dynamodb:Scan",
                "dynamodb:Query",
                "cloudwatch:Get*",
                "cloudwatch:List*",
                "cloudwatch:Describe*",
                "logs:Get*",
                "logs:List*",
                "logs:Describe*",
                "sns:Get*",
                "sns:List*",
                "sqs:Get*",
                "sqs:List*",
                "elasticloadbalancing:Describe*",
                "autoscaling:Describe*",
                "cloudformation:Describe*",
                "cloudformation:Get*",
                "cloudformation:List*",
                "ecs:Describe*",
                "ecs:List*",
                "eks:Describe*",
                "eks:List*",
                "elasticache:Describe*",
                "elasticache:List*",
                "ssm:Describe*",
                "ssm:Get*",
                "ssm:List*",
                "kms:Describe*",
                "kms:Get*",
                "kms:List*",
                "secretsmanager:Describe*",
                "secretsmanager:List*",
                "iam:Get*",
                "iam:List*",
                "route53:Get*",
                "route53:List*"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": [
                        "us-east-1",
                        "us-east-2"
                    ]
                }
            }
        }
    ]
}
```

*(Replace the placeholder with the full policy JSON provided in the lab.)*

---

## Step-3: Name the Policy

Example:

```
EnterpriseEC2IAMGovernancePolicy
```

Description:

```
Controls EC2 creation, IAM role management, and read-only access across services with strict regional and permission boundary constraints.
```

---

## Step-4: Attach Policy to IAM User or Group

Navigate to:

```
IAM → Users
```

Select the user and attach the policy.

Alternatively attach to a group:

```
IAM → Groups → Attach Policy
```

---

# Policy Breakdown

---

# EC2 Read Access in Two Regions

Allows read-only access to EC2 resources in:

```
us-east-1
us-east-2
```

Permissions include:

* Describe
* Get
* List

These allow visibility into EC2 resources without modification rights.

---

#  Controlled EC2 Instance Creation

The policy allows launching instances only when:

| Condition      | Requirement             |
| -------------- | ----------------------- |
| Region         | `us-east-1`             |
| Instance Types | `t3.micro` or `t3.nano` |

This prevents launching expensive instance types.

---

#  Controlled EBS Volume Creation

The policy restricts volume creation using:

| Parameter    | Allowed Value |
| ------------ | ------------- |
| Region       | `us-east-1`   |
| Volume Type  | `gp2`         |
| Maximum Size | **30GB**      |

This prevents users from provisioning expensive storage.

---

# EC2 Resource Management

Users are allowed to manage existing EC2 instances in `us-east-1`.

Allowed actions include:

* StartInstances
* StopInstances
* RebootInstances
* TerminateInstances
* AttachVolume
* DetachVolume
* CreateTags
* ModifyInstanceAttribute

These permissions enable instance lifecycle management.

---

# IAM Role Governance

This policy restricts IAM role creation and modification.

Rules enforced:

* Role name must contain **Lambda** or **S3**
* Roles must include **Permission Boundary**
* Roles without boundary are denied
* Permission boundaries cannot be removed

This prevents privilege escalation.

---

# Policy Attachment Restrictions

Only policies related to:

```
Lambda
S3
```

can be attached or detached from the roles.

This ensures roles cannot gain additional unrelated privileges.

---

# PassRole Restriction

The policy allows:

```
iam:PassRole
```

only when the role is passed to:

```
EC2 Service
```

This prevents misuse of IAM roles across services.

---

# Read-Only Access to Other AWS Services

The policy allows read-only access to multiple AWS services including:

* S3
* Lambda
* DynamoDB
* RDS
* CloudWatch
* SNS
* SQS
* ECS
* EKS
* CloudFormation
* Secrets Manager
* KMS
* Route53

Only **Get, List, and Describe operations** are allowed.

This ensures visibility without modification rights.

---

#  Permission Evaluation Logic

Access decisions follow AWS evaluation order:

```
Explicit Deny
     ↓
SCP / RCP
     ↓
IAM Policies
     ↓
Permission Boundaries
     ↓
Final Decision
```

Example:

```
IAM Policy → Allow
Permission Boundary → Deny
Final Result → DENIED
```

---

#  Validation Tests

After attaching the policy, the following tests were performed.

| Test                         | Expected Result |
| ---------------------------- | --------------- |
| Launch EC2 in us-east-1      | Allowed         |
| Launch EC2 in eu-west-1      | Denied          |
| Launch t3.micro              | Allowed         |
| Launch m5.large              | Denied          |
| Create 20GB gp2 volume       | Allowed         |
| Create 100GB volume          | Denied          |
| Create role without boundary | Denied          |
| Attach unrelated policy      | Denied          |

---