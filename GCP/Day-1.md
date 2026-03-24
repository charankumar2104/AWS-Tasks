# 📘 GCP Identity & Access Management Guide

## (Simple Explanation with Flow & Best Practices)

---

# 🎯 Objective

This document explains how **user management and access control works in Google Cloud Platform (GCP)** in a simple and structured way.

You will learn:

* Difference between Admin Console & Cloud Console
* How users and groups are created
* How permissions are assigned using IAM
* Best practices for managing access
* Flow of how everything connects

---

# 🧭 1. The Tale of Two Consoles

In GCP, there are **two main consoles**, and each has a different purpose.

---

## 🏢 Google Admin Console

👉 URL: `admin.google.com`

### Purpose: **Identity Management**

This is where **users and groups are created and managed**.

### What you do here:

* Create users
* Reset passwords
* Enable 2FA (Two-Factor Authentication)
* Create and manage groups

---

## ☁️ Google Cloud Console

👉 URL: `console.cloud.google.com`

### Purpose: **Resource & Access Management**

This is where:

* You create infrastructure (VMs, networks, etc.)
* You assign permissions using IAM

---

## 🔄 Flow Between Both Consoles

```text
Admin Console (Create Users & Groups)
                ↓
Cloud Console (Assign Permissions via IAM)
                ↓
Users Access Resources
```

---

# 👥 2. Managing Users (Admin Console)

---

## 🔹 Creating Users

Steps:

1. Login to Admin Console
2. Go to:

   ```
   Directory → Users
   ```
3. Click **Add New User**
4. Enter details:

   * Name
   * Email (e.g., [user@company.com](mailto:user@company.com))
   * Password

---

## 🔹 Important Note

In real-world companies:

* Users are NOT created manually
* They are synced using:

  * **SSO (Single Sign-On)**
  * **GCDS (Google Cloud Directory Sync)**
  * External providers like:

    * Okta
    * Microsoft Entra ID

---

# 👥 3. Managing Groups (Best Practice ⭐)

---

## 🔹 Why Groups?

Instead of giving access to individual users:

❌ Bad Practice:

```text
User1 → Role
User2 → Role
User3 → Role
```

✅ Best Practice:

```text
Group → Role
Users → Added to Group
```

---

## 🔹 Creating Groups

Steps:

1. Go to:

   ```
   Directory → Groups
   ```
2. Click **Create Group**
3. Provide:

   * Group Name
   * Email (e.g., [dev-team@company.com](mailto:dev-team@company.com))
4. Add members

---

## 🔹 Example Groups

| Group Name                                                      | Purpose       |
| --------------------------------------------------------------- | ------------- |
| [network-admins@company.com](mailto:network-admins@company.com) | Network team  |
| [dev-team@company.com](mailto:dev-team@company.com)             | Developers    |
| [security-team@company.com](mailto:security-team@company.com)   | Security team |

---

# 🔐 4. IAM (Identity and Access Management)

IAM is used to **control access to cloud resources**.

---

# 🧠 IAM Formula

```text
WHO → WHAT → WHICH RESOURCE
```

| Component | Meaning                     |
| --------- | --------------------------- |
| WHO       | User or Group               |
| WHAT      | Role (permissions)          |
| RESOURCE  | Project / Folder / Resource |

---

# 🔁 IAM Working Flow

```text
User → Part of Group → Assigned Role → Access Resource
```

---

# 🧩 IAM Components

---

## 🔹 1. Principal (WHO)

* User email
* Group email

Example:

```text
dev-team@company.com
```

---

## 🔹 2. Role (WHAT)

A role is a **set of permissions**.

Types of roles:

| Type             | Description            |
| ---------------- | ---------------------- |
| Basic Roles      | Owner, Editor, Viewer  |
| Predefined Roles | Specific service roles |
| Custom Roles     | User-defined           |

---

### Example Roles:

* Compute Instance Admin
* Storage Admin
* Network Admin

---

## 🔹 3. Resource (WHERE)

GCP has a hierarchy:

```text
Organization
   ↓
Folder
   ↓
Project
   ↓
Resources (VM, Storage, etc.)
```

---

# 🔄 IAM Hierarchy Flow

```text
Role given at Organization
        ↓
Inherited by Folder
        ↓
Inherited by Project
        ↓
Inherited by Resources
```

---

# 🛠️ 5. How to Grant Access (Step-by-Step)

---

### Step-1: Go to Cloud Console

```text
IAM & Admin → IAM
```

---

### Step-2: Click "Grant Access"

---

### Step-3: Enter Details

| Field     | Example                                             |
| --------- | --------------------------------------------------- |
| Principal | [dev-team@company.com](mailto:dev-team@company.com) |
| Role      | Compute Instance Admin                              |

---

### Step-4: Save

✔ Now all users in that group get access

---

# 🔁 Access Flow Diagram

```text
Admin creates User → Adds to Group
            ↓
Cloud Admin assigns Role to Group
            ↓
Group gets permission
            ↓
User gets access automatically
```

---

# 🔒 6. Security Best Practices

---

## ✅ Always Follow

* Use **Groups instead of users**
* Enable **2FA**
* Use **least privilege principle**
* Regularly audit IAM roles
* Use **custom roles** if needed

---

## ❌ Avoid

* Giving **Owner role to everyone**
* Assigning roles directly to users
* Using shared accounts

---

# 🔍 7. Additional Important Concepts

---

## 🔹 Least Privilege Principle

Give only the permissions required.

Example:

```text
Developer → No need for Billing Access
```

---

## 🔹 IAM Policy

IAM policies define:

```text
Who → What → Resource
```

---

## 🔹 Audit & Monitoring

Use:

* Cloud Audit Logs
* IAM Recommender

---

## 🔹 Service Accounts

Used by applications instead of users.

Example:

```text
VM → Access Storage
```

---