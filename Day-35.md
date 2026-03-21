#  Day-35 Lab

## Topic: AWS Cost Estimation using AWS Pricing Calculator & Custom Google Sheet Model

---

#  Objective

In this lab, I worked on estimating AWS infrastructure cost using:

* **AWS Pricing Calculator**
* A **custom-built Google Sheet model** for cost breakdown

The goal was to:

* Understand how AWS pricing works
* Estimate cost per user and total cost
* Analyze resource-wise pricing
* Build a reusable cost estimation sheet

---

#  Cost Estimation Reference Sheet

👉 **Google Sheet Link:**


[Click here](https://docs.google.com/spreadsheets/d/14hxST5v4vYRlA0bF2c4vzp_qX0BXpnanQQmMp0iziVs/edit?usp=sharing) to open the spreadsheet


---

# 🧠 Understanding AWS Pricing

AWS follows a **pay-as-you-go model**, meaning:

* You only pay for what you use
* No upfront cost (in most services)
* Billing depends on:

  * Usage duration
  * Resource type
  * Region
  * Data transfer

---

# 💰 What is AWS Pricing Calculator?

The **AWS Pricing Calculator** is a tool used to:

* Estimate cost before deployment
* Configure services like EC2, RDS, S3
* Get monthly cost breakdown
* Compare pricing across regions

---

# ⚙️ Key Factors Affecting Cost

| Factor        | Description                              |
| ------------- | ---------------------------------------- |
| Region        | Different regions have different pricing |
| Instance Type | Higher specs = higher cost               |
| Storage       | GB used impacts pricing                  |
| Data Transfer | Outbound traffic costs more              |
| Usage Time    | Hours or minutes used                    |
| Scaling       | More users = higher cost                 |

---

# 📊 Custom Cost Estimation Model (Google Sheet)

The sheet created includes:

---

## 🔹 Input Parameters

| Parameter       | Value     |
| --------------- | --------- |
| Lab Duration    | 1440      |
| VM Uptime       | 50        |
| Number of Users | 100       |
| Region          | us-east-1 |

---

## 🔹 Resources Considered

| S No | Resource      | Description    |
| ---- | ------------- | -------------- |
| 1    | EC2           | t3.xlarge      |
| 2    | EBS           | 128GB gp3      |
| 3    | S3            | 150GB storage  |
| 4    | RDS Compute   | db.t3.small    |
| 5    | RDS Storage   | 20GB           |
| 6    | ELB           | Load balancing |
| 7    | Data Transfer | 10GB outbound  |

---

## 🔹 Cost Calculation Logic

Each resource includes:

* Monthly pricing
* Whether resource can be paused
* Count of resources
* Cost per user

---

## 🧮 Sample Formula Used

```excel
=IF(LOWER(E13)="yes",(D13/720)*C8*F13,(D13/720)*C7*F13)
```

Explanation:

* If resource **can be paused**, cost is calculated based on uptime
* Else full duration is considered

---

# 📈 Cost Summary

---

## 💵 Total Cost per User

```text
~ $99.72 per user
```

---

## 💰 Total Cost for All Users

```text
~ $9972.22
```

---

# 🧠 Insights from Estimation

* **ELB and EC2** contribute highest cost
* **Pause-able resources reduce cost significantly**
* **RDS compute cost is moderate but storage is low**
* **Data transfer is minimal but scales with usage**

---

# 🚀 Why Use a Custom Cost Sheet?

AWS Pricing Calculator is useful, but a custom sheet provides:

| Advantage        | Description                     |
| ---------------- | ------------------------------- |
| Flexibility      | Customize formulas              |
| Per User Cost    | Not available in AWS calculator |
| Scenario Testing | Change inputs easily            |
| Automation       | Pre-built calculations          |

---

# 🔄 AWS Pricing Calculator vs Custom Sheet

| Feature           | AWS Calculator | Google Sheet     |
| ----------------- | -------------- | ---------------- |
| Accuracy          | High           | Depends on input |
| Customization     | Limited        | High             |
| Per User Cost     | ❌              | ✅                |
| Automation        | ❌              | ✅                |
| Scenario Analysis | Medium         | High             |

---

# 📌 Best Practices for Cost Optimization

* Use **t3/t4 burstable instances**
* Enable **Auto Scaling**
* Stop unused resources
* Use **Reserved Instances** for long-term
* Monitor with **CloudWatch + Budgets**
* Use **S3 lifecycle policies**

---