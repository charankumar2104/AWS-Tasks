# 📘 Day-21 Lab Documentation

## Topic: AWS Organizations, Budgets, SCP, RCP and IAM Concepts

**Date:** 02/03/2026 – 03/03/2026

This lab focused on understanding **AWS account structure, organizational hierarchy, governance policies, cost management, and IAM fundamentals**.

---

# 1️⃣ AWS Standalone Account

An **AWS Standalone Account** is the account created by an individual user using their own email address.

Characteristics:

* Independent account
* Not part of any AWS Organization
* Managed entirely by the account owner
* No centralized governance

When a standalone account is converted into an organization, it becomes the **Management Account**.

---

# 2️⃣ AWS Organization Structure

AWS Organizations allows centralized management of multiple AWS accounts.

### Hierarchy Flow

```
AWS Organization
│
├── Management Account (Parent / Root of Org)
│   │
│   ├── IAM User (Admin)
│   │
│   └── Organization Root
│
└── Organizational Unit (OU)
    │
    ├── Member Account 1
    │   └── IAM Users / Roles / Resources
    │
    └── Member Account 2
        └── IAM Users / Roles / Resources
```

### Explanation

* The **Management Account** sits at the top of the organization.
* Resources are generally **not deployed in the management account**.
* It is used for **governance, billing, and policy management**.
* **Member accounts** contain the actual workloads and resources.
* **Organizational Units (OU)** group multiple member accounts.

---

# 3️⃣ Root Account

The **Root Account** is the original login identity of every AWS account.

### Characteristics

* Created when the AWS account is created
* Has **full unrestricted permissions**
* Can perform critical administrative tasks

### Capabilities

* Add or remove member accounts
* Enable or disable MFA
* Manage billing
* Modify organization settings

### Best Practice

Root account should be used **only for critical tasks** and protected using **MFA**.

---

# 4️⃣ Management Account

The **Management Account** is the main account that creates and manages the AWS Organization.

### Responsibilities

* Controls organization-wide billing
* Creates and manages member accounts
* Manages Organizational Units (OU)
* Creates and attaches **Service Control Policies (SCP)**

### Important Note

Resources are generally **not deployed here** to maintain governance separation.

---

# 5️⃣ Organizational Unit (OU)

An **Organizational Unit** is a logical container used to group multiple AWS accounts.

### Purpose

* Organize accounts by department, environment, or team
* Apply policies to multiple accounts at once
* Manage access centrally

### Example

```
Organization
│
├── OU: Development
│   ├── Dev Account 1
│   └── Dev Account 2
│
└── OU: Production
    ├── Prod Account 1
    └── Prod Account 2
```

---

# 6️⃣ Member Account

A **Member Account** is a regular AWS account inside an organization.

### Characteristics

* Cannot manage the organization
* Cannot create additional member accounts
* Cannot modify SCP policies
* Controlled by policies defined in Management Account

### Responsibilities

* Hosts application resources
* Runs workloads
* Managed through organization policies

---

# 7️⃣ AWS Budgets

AWS Budgets help monitor and control cloud spending.

---

## 1. Zero Spend Budget

Purpose: Ensure usage stays within **AWS Free Tier**.

Characteristics:

* Budget limit set to **$0**
* Alerts if spending occurs
* Prevents unexpected charges

---

## 2. Monthly Cost Budget

Tracks the **total spending for a month** against a defined budget limit.

Features:

* Cost tracking
* Forecast prediction
* Alerts when budget threshold is exceeded

---

## 3. Daily Savings Plans Coverage Budget

Savings Plans provide discounts for committing to certain usage levels.

Coverage measures:

```
How much of your total workload is covered by savings plans
```

Purpose:

* Check if workloads are optimized for discounts

---

## 4. Daily Reservation Utilization Budget

Reserved Instances provide discounted pricing.

Utilization measures:

```
How much of the reserved capacity is actually used
```

Goal:

* Ensure reserved instances are not wasted

---

# 8️⃣ Service Control Policies (SCP)

A **Service Control Policy (SCP)** is an organization-level guardrail that defines the **maximum permissions** available to accounts.

### Key Characteristics

* Applies to **Organization Root, OUs, and Member Accounts**
* Does **not grant permissions**
* Defines **maximum allowed permissions**

### SCP Can Restrict

* AWS services usage
* Specific actions
* Regions
* Conditions

### SCP Evaluation Logic

| IAM   | SCP   | Result  |
| ----- | ----- | ------- |
| Allow | Allow | Allowed |
| Allow | Deny  | Denied  |
| Deny  | Allow | Denied  |

---

# 9️⃣ Resource Control Policies (RCP)

A **Resource Control Policy (RCP)** controls access to resources across an organization regardless of the account making the request.

### Characteristics

* Applies to:

  * Organization Root
  * OUs
  * Member Accounts
  * Resources

### Capabilities

* Restrict cross-account access
* Restrict external access
* Enforce organization-wide resource protection

### Important Notes

* Does **not grant permissions**
* Does **not replace resource policies**
* Does **not override explicit deny rules**

---

# 🔟 IAM (Identity and Access Management)

IAM controls **who can access AWS and what actions they can perform**.

---

# IAM User

An **IAM User** represents a person or application.

### Components

* Username
* Password (Console login)
* Access Keys (CLI / SDK)

### Permissions

Permissions are attached through **IAM policies**.

Best Practice:

Use **IAM roles for applications instead of IAM users**.

---

# IAM Role

An **IAM Role** is an identity with temporary permissions.

### Characteristics

* No username
* No password
* Temporary credentials
* Assumed by users or services

### Common Use Cases

* EC2 accessing S3
* Cross-account access
* Dev environment accessing production
* Lambda service permissions

---

# IAM Policies

An **IAM Policy** defines permissions in JSON format.

Policies specify:

* Allowed actions
* Resources
* Conditions

### Example Policy Structure

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

---

# Types of IAM Policies

### 1️⃣ Identity Based Policies

Attached to:

* Users
* Groups
* Roles

Control what the identity can do.

---

### 2️⃣ Resource Based Policies

Attached directly to resources.

Example:

* S3 bucket policies
* SNS policies
* SQS policies

---

### 3️⃣ Permission Boundaries

Defines the **maximum permissions an IAM identity can receive**.

Acts as a guardrail.

---

### 4️⃣ Session Policies

Temporary policies applied during role assumption.

Used for limiting temporary sessions.

---

All usage from these accounts appears in **one consolidated bill**.

---


All usage from these accounts appears in **one consolidated bill**.

---

# AWS Cost Management Tools

AWS provides several tools for cost monitoring.

| Tool | Purpose |
|-----|---------|
AWS Billing Dashboard | Shows current charges |
AWS Cost Explorer | Analyze cost trends |
AWS Budgets | Set spending alerts |
AWS Cost & Usage Report | Detailed billing data |
AWS Savings Plans | Discounted compute usage |

---

# AWS Budgets

AWS Budgets help track spending and send alerts when costs exceed defined limits.

---

## Types of Budgets

### 1️⃣ Zero Spend Budget

Budget limit is set to **$0**.

Purpose:

- Ensure usage remains within **AWS Free Tier**
- Alert if spending begins

---

### 2️⃣ Monthly Cost Budget

Tracks monthly spending against a defined limit.

Features:

- Forecast future costs
- Send alerts when limits approach

---

### 3️⃣ Daily Savings Plans Coverage Budget

Measures how much of your usage is covered by **Savings Plans discounts**.

Purpose:

- Ensure workloads utilize savings plans effectively.

---

### 4️⃣ Daily Reservation Utilization Budget

Tracks whether **Reserved Instances are being fully used**.

Purpose:

- Prevent wasted reserved capacity.

---

# Creating an AWS Budget

### Steps

1. Login to **AWS Management Console**

2. Navigate to:


3. Click **Create Budget**

4. Select budget type

Examples:

- Cost Budget
- Usage Budget
- Savings Plans Budget
- Reservation Budget

---

## Configure Budget

Example configuration:

| Setting | Example |
|-------|--------|
Budget Name | MonthlyBudget |
Budget Period | Monthly |
Budget Amount | $10 |

---

## Configure Alerts

Example alert configuration:

| Alert Type | Percentage |
|-----------|------------|
Actual Cost Alert | 80% |
Forecast Cost Alert | 100% |

Notifications can be sent through:

- Email
- Amazon SNS

---

# Example Budget Monitoring


AWS sends notification when spending reaches **$8**.

---

# 7️⃣ Service Control Policies (SCP)

A **Service Control Policy (SCP)** is an organization-level guardrail.

### Purpose

Defines **maximum permissions an AWS account can have**.

### Applies To

- Organization Root
- Organizational Units
- Member Accounts

### SCP Rules

- Does **not grant permissions**
- Only **restricts permissions**
- Works with IAM policies

### Permission Evaluation

| IAM | SCP | Result |
|----|----|----|
Allow | Allow | Allowed |
Allow | Deny | Denied |
Deny | Allow | Denied |

---

# 8️⃣ Resource Control Policies (RCP)

A **Resource Control Policy (RCP)** controls resource access across an organization.

### Characteristics

Applies to:

- Organization Root
- Organizational Units
- Member Accounts
- Resources

### Capabilities

- Restrict cross-account access
- Restrict external access
- Provide organization-wide resource protection

### Important Notes

- Does **not grant permissions**
- Does **not override explicit deny**

---

# 9️⃣ IAM (Identity and Access Management)

IAM controls:

---

# IAM User

Represents a **person or application**.

### Components

- Username
- Console password
- Access keys for CLI

### Permissions

Controlled using **IAM policies**.

Best practice:

Use **IAM Roles instead of IAM Users for applications**.

---

# IAM Role

An **IAM Role** is an identity with temporary permissions.

### Characteristics

- No username
- No password
- Temporary credentials

### Common Use Cases

- EC2 accessing S3
- Cross-account access
- Dev environment accessing production
- Lambda accessing other services

---

# IAM Policies

Policies define permissions using **JSON format**.

Example:

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