# Day-22 Lab

## Topic: AWS CloudWatch, IAM Users, Policies, Permission Boundaries & Cross-Account Access

---

#  Objective

In this lab we explored:

* **AWS CloudWatch monitoring service**
* **IAM Users and IAM Policies**
* **Region restriction policy**
* **Permission Boundaries for fine-grained access**
* **Cross-account access using IAM Roles**
* **Governance using SCP and RCP**

These concepts are essential for **monitoring infrastructure and managing secure access in AWS environments**.

---

# AWS CloudWatch

Amazon **CloudWatch** is the primary monitoring service in AWS.

It collects data from AWS resources and provides tools to **monitor, visualize, alert, and automate responses** based on system activity.

CloudWatch helps in:

* Monitoring infrastructure
* Detecting failures
* Sending alerts
* Logging application events
* Automating responses

---

# 🔹 Core Components of CloudWatch

---

## Metrics

A **Metric** represents a time-ordered set of data points.

Examples:

* CPU Utilization
* Network In / Out
* Disk Read / Write
* Memory usage

### Types of Metrics

| Metric Type      | Description                             |
| ---------------- | --------------------------------------- |
| Standard Metrics | Automatically collected by AWS services |
| Custom Metrics   | User defined metrics from applications  |

---

## Logs

CloudWatch Logs store and monitor logs from AWS resources.

Sources include:

* EC2 instances (via CloudWatch agent)
* Lambda functions
* Application logs
* VPC Flow Logs

### Log Structure

```
Log Group
   │
   ├── Log Stream (Instance 1)
   └── Log Stream (Instance 2)
```

| Component  | Description                          |
| ---------- | ------------------------------------ |
| Log Group  | Collection of log streams            |
| Log Stream | Sequence of log events from a source |
| Log Event  | Individual log entry                 |

---

## Alarms

CloudWatch **Alarms monitor metrics and trigger actions** when conditions are met.

### Alarm Types

**Static Alarm**

Triggers when metric crosses threshold.

Example:

```
CPU Utilization > 80%
```

**Anomaly Detection**

Uses machine learning to detect unusual behavior.

Example:

```
Traffic spike beyond normal range
```

### Alarm Actions

* Send SNS notification
* Trigger Auto Scaling
* Execute Lambda
* Stop or terminate EC2 instance

---

## Dashboards

CloudWatch Dashboards are **custom monitoring panels**.

They allow monitoring multiple AWS services from a single place.

Example Dashboard:

```
EC2 CPU Usage
Lambda Errors
S3 Request Metrics
Application Logs
```

---

# IAM (Identity and Access Management)

IAM controls:

```
Who can access AWS
What actions they can perform
```

---

# IAM Users

An **IAM User** represents a person or application accessing AWS.

### Components

* Username
* Console Password
* Access Keys (CLI / API)

Permissions are assigned through **IAM policies**.

Best Practice:

Use **IAM Roles for applications instead of IAM users**.

---

# IAM Policies

IAM policies define permissions in **JSON format**.

Example policy:

```json
{
 "Version": "2012-10-17",
 "Statement": [
   {
     "Effect": "Allow",
     "Action": "s3:ListBucket",
     "Resource": "*"
   }
 ]
}
```

Policies specify:

* Effect (Allow / Deny)
* Actions
* Resources
* Conditions

---

# Task-1: Region Restriction Policy

### Objective

Create a policy that allows actions only in:

```
us-east-1
ap-south-1
```

---

## Add Policy JSON Below

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "Statement1",
			"Effect": "Allow",
			"Action": [
				"*"
			],
			"Resource": [
				"*"
			],
			"Condition": {
				"StringEquals": {
					"aws:RequestedRegion": [
						"us-east-1",
						"ap-south-1"
					]
				}
			}
		}
	]
}
```

---

# Task-2: IAM User with Permission Boundary

### Objective

Create an IAM user with **Administrator access but restrict permissions using a Permission Boundary**.

---

## Step-1

Create a user with:

```
AdministratorAccess
```

or add the user to an **Admin Group**.

---

## Step-2

Create a policy that defines **maximum permissions allowed**.

Example:

```
Allow only S3 or EC2 access
```

---

## Step-3

Attach this policy as a **Permission Boundary** to the user.

---

## Step-4

Now the user's permissions become restricted.

Permission boundary acts as **maximum permission guardrail**.

---

## Permission Evaluation

```
Effective Permissions =
User / Group Policies
INTERSECTION
Permission Boundary
```

Example:

```
Admin Policy → Full Access
Permission Boundary → Only S3

Final Result → Only S3 Access
```

Important rule:

```
If a user has ONLY permission boundary
and no policies attached

→ User cannot perform any actions
```

---

# Task-3: Cross-Account Access Using IAM Roles

Cross-account access allows a user from one AWS account to access resources in another account.

This is implemented using **IAM Roles and AWS STS (Security Token Service)**.

---

## Step-1

Create resources in the **Source Account**.

Example resources:

```
S3 Bucket
EC2 Instance
DynamoDB Table
```

---

## Step-2

Create a **policy that allows access to the resources**.

Attach it as a **permission boundary if required**.

---

## Step-3

Create an IAM Role in the **Source Account** with the following **Trust Policy**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<Destination-account-id>:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "charan-to-cherry-s3"
        }
      }
    }
  ]
}
```

---

## Step-4

Attach required policies to the role.

Examples:

```
AmazonS3ReadOnlyAccess
Custom policies
AdministratorAccess
```

Attach permission boundary if required.

---

## Step-5

Create a policy in the **Destination Account**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<Source-account-id>:role/CharanS3ReadRole"
    }
  ]
}
```

---

## Step-6

Attach this policy to a **User or Group in Destination Account**.

---

## Step-7

Switch Role

User logs in and selects **Switch Role**.

Provide:

```
Source Account ID
Role Name
External ID (if required)
```

---

#  AWS STS AssumeRole

```
sts:AssumeRole
```

Allows a trusted identity to **temporarily assume permissions of another IAM role**.

Common use cases:

* Cross-account access
* Temporary credentials
* Secure delegation of permissions

---

# Service Control Policies (SCP)

SCPs are **organization-level guardrails** in AWS Organizations.

### Characteristics

* Apply to **Organization Root, OUs, and Member Accounts**
* Define **maximum allowed permissions**
* Do **not grant permissions**

Example evaluation:

```
IAM Policy → Allow EC2
SCP → Deny EC2

Final Result → DENIED
```

---

# Resource Control Policies (RCP)

RCP controls **access to resources across the organization**.

### Characteristics

Applies to:

* Organization Root
* Organizational Units
* Member Accounts
* Resources

### Capabilities

* Restrict cross-account access
* Restrict external access
* Protect resources organization-wide

Important:

```
RCP does not grant permissions
and does not override explicit deny rules
```

---