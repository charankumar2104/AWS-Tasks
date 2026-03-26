# Cloud Services Reference Guide
## AWS · Azure · GCP

---

## 1. Cloud Services Comparison Table

### Containers & AI/ML Compute

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Container Registry | Elastic Container Registry (ECR) | Azure Container Registry | Artifact Registry |
| AI/ML Optimized Compute | EC2 (Trn1, Inf1, P5) | VM (H100, H200, M1300x) | Cloud GPUs / Cloud TPU |

---

### Compute

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Virtual Machines | EC2 | Azure Virtual Machines | Compute Engine |
| Auto Scaling | EC2 Auto Scaling | Azure Autoscale / VM Scale Sets | Compute Engine Autoscaler |
| Block Storage | EBS (Elastic Block Store) | Azure Managed Disks | Google Cloud Hyperdisk |
| Browser-based VM Access | EC2 Instance Connect | — | SSH-in-Browser |
| Secure Remote Access / Bastion | EC2 Instance Connect | Azure Bastion | SSH-in-Browser |
| Patch & Update Management | AWS Systems Manager | Azure Update Manager | VM Manager |

---

### Containers & Serverless

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Managed Kubernetes | EKS | AKS (Azure Kubernetes Service) | GKE (Google Kubernetes Engine) |
| Serverless Containers | AWS Fargate | AKS Automatic | GKE Autopilot |
| Serverless Functions | AWS Lambda | Azure Functions | Cloud Run Functions |

---

### Databases & Caching

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| In-Memory Cache | ElastiCache | Azure Cache for Redis | Memorystore |
| NoSQL / Wide Column | DynamoDB | Azure Cosmos DB | Bigtable |
| Managed PostgreSQL (High Perf) | Aurora | Azure Cosmos DB for PostgreSQL | AlloyDB for PostgreSQL |
| Managed Relational DB | RDS | Azure DB for MySQL | Cloud SQL |

---

### Developer & CI/CD Tools

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Cloud Shell | AWS CloudShell | Azure Cloud Shell | Google Cloud SDK Shell |
| Source Code Repository | AWS CodeCommit | Azure Repos | Cloud Source Repositories |
| CI/CD Pipeline | AWS CodePipeline | Azure DevOps | Cloud Deploy |

---

### Messaging & Eventing

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Event Scheduling / Orchestration | Amazon EventBridge | Azure Logic Apps | Cloud Scheduler |
| Message Queue & Pub/Sub | SQS / SNS | Azure Queue Storage / Azure Service Bus | Cloud Tasks / Pub/Sub |

---

### Networking & CDN

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| CDN | CloudFront | Azure Front Door | Cloud CDN / Media CDN |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| Load Balancer | ELB (Elastic Load Balancing) | Azure Load Balancer | Cloud Load Balancing |
| VPN | AWS VPN | Azure VPN Gateway | Cloud VPN |
| Transit / Network Hub | AWS Transit Gateway | Azure Virtual Network Manager | Network Connectivity Center |
| Private Endpoint Connectivity | AWS PrivateLink | Azure Private Link | Private Service Connect |
| Network Monitoring & Diagnostics | AWS Network Manager | Azure Network Watcher | Network Intelligence Center |
| NAT Gateway | AWS NAT Gateway | Azure NAT Gateway | Cloud NAT |
| Virtual Private Network | VPC | Azure VNet | VPC |

---

### Storage

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Object Storage | Amazon S3 | Azure Blob Storage | Cloud Storage |

---

### Observability & Logging

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Audit Logging | AWS CloudTrail | Azure Activity Logs | Cloud Logging |
| Monitoring & Metrics | AWS CloudWatch | Azure Monitor | Cloud Monitoring |
| Log Management | AWS CloudWatch Logs | Azure Monitor Logs | Cloud Logging |
| Container Image Scanning | ECR Image Scanning | Microsoft Defender for Container Registries | Artifact Analysis |

---

### Security & Identity

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Key Management | AWS KMS | Azure Key Vault | Cloud KMS / Cloud HSM |
| Single Sign-On / Identity Platform | AWS IAM Identity Center | Microsoft Entra ID | Cloud Identity |
| IAM | AWS IAM | Azure Identity Management | IAM |
| Policy & Compliance | AWS Config | Azure Policy | Organization Policy Service |
| Management Hierarchy | AWS Organizations | Azure Resource Manager | Resource Manager |
| Secrets & Parameter Management | AWS SSM Parameter Store | Azure Key Vault | Secret Manager |

---

---

## 2. GCP IAM — Components Reference

> A complete breakdown of every component available under **Google Cloud IAM & Admin**, what it does, which level of the resource hierarchy it applies to, and its equivalent in AWS or Azure.

### Resource Hierarchy — Quick Reference

```
Organization  (org level)
    └── Folder  (folder level)
            └── Project  (project level)
                    └── Resource  (resource level)
```

IAM policies set at a higher level are **inherited downward**. For example, a role granted at the Org level applies to all folders, projects, and resources beneath it.

---

### 2.1 IAM (Identity & Access Management)

The core service for controlling **who can do what on which resource** in GCP.

**Used at:** Organization · Folder · Project · Resource level

IAM works on a simple model:

```
Principal  →  Role  →  Resource
(who)         (what)    (where)
```

#### IAM Roles
A collection of permissions bundled together and assigned to a principal on a resource. There are three types:

| Role Type | Description |
|-----------|-------------|
| **Basic Roles** | `Owner`, `Editor`, `Viewer` — broad access, not recommended for production |
| **Predefined Roles** | Fine-grained roles managed by Google (e.g., `roles/storage.objectViewer`, `roles/compute.instanceAdmin`) |
| **Custom Roles** | User-defined roles with exactly the permissions you choose — used for least-privilege access |

**Used at:** Organization · Folder · Project · Resource level

> **Similar to:** IAM Managed Policy / Inline Policy (AWS) · Azure RBAC Role Definition (Azure)

---

#### IAM Allow Policy
A set of bindings that attach one or more roles to one or more principals on a specific resource. Every GCP resource has its own allow policy.

**Used at:** Organization · Folder · Project · Resource level

> **Similar to:** Identity-based / Resource-based Policy (AWS) · Azure Role Assignment (Azure)

---

#### IAM Deny Policy
Explicitly blocks specific permissions for specific principals — even if an allow policy grants them. Deny always takes precedence over allow.

**Used at:** Organization · Folder · Project level

> **Similar to:** Explicit Deny in IAM Policy (AWS) · Azure Deny Assignments (Azure)

---

#### IAM Conditions
Adds attribute-based rules to role bindings so access is only granted when conditions are met — such as time of day, resource tag, or request origin IP.

**Used at:** Organization · Folder · Project · Resource level

> **Similar to:** IAM Policy Condition Keys (AWS) · Azure Conditional Access (Azure)

---

### 2.2 PAM (Privileged Access Management)

GCP PAM provides **just-in-time (JIT) elevated access** to sensitive resources. Instead of permanently assigning powerful roles, users request temporary privileged access for a defined time window. This reduces standing privilege and limits blast radius if an account is compromised.

**Used at:** Organization · Folder · Project level

Key features:
- Time-bound access grants (e.g., elevated access for 2 hours)
- Approval workflows — a manager or peer approves the request before access is activated
- Full audit trail of who requested, who approved, and what was accessed
- Automatically revokes access when the time window expires

> **Similar to:** AWS IAM Identity Center temporary elevated access / AWS Organizations SCP with time conditions (AWS) · Azure Privileged Identity Management — PIM (Azure)

---

### 2.3 Principal Access Boundaries (PAB)

Principal Access Boundaries define **the maximum set of resources a principal can ever access**, regardless of what IAM roles they have been granted. It is a hard ceiling on permissions — even if someone grants a user a powerful role, PAB ensures they cannot access resources outside the defined boundary.

**Used at:** Organization level (applied to principals across the org)

Key use cases:
- Prevent users in one business unit from ever accessing another unit's projects
- Ensure contractors or external users can only operate within a defined project set
- Act as a safety net on top of IAM roles

> **Similar to:** AWS Permission Boundaries on IAM Roles/Users (AWS) · No direct equivalent — closest is Azure Management Group scope restrictions (Azure)

---

### 2.4 Security Insights

Security Insights surfaces **risk signals and recommendations** about your IAM configuration — such as over-permissioned accounts, unused roles, service accounts with excessive privileges, and risky policy bindings.

**Used at:** Organization · Project level

Key features:
- Highlights principals with more permissions than they use (based on actual activity)
- Flags service accounts with owner/editor roles
- Recommends right-sizing roles to least privilege
- Integrates with IAM Recommender for automated suggestions

> **Similar to:** AWS IAM Access Analyzer findings + Trusted Advisor (AWS) · Azure Secure Score Identity Recommendations / Microsoft Entra ID Protection (Azure)

---

### 2.5 Identity and Organisation

This section in the GCP Console manages the **organization-level identity setup** — linking your GCP Organization to a Google Workspace or Cloud Identity domain and managing org-wide identity settings.

**Used at:** Organization level

Key responsibilities:
- Linking the GCP org to a Google Workspace or Cloud Identity account
- Managing domain-wide user provisioning and deprovisioning
- Configuring authentication policies (MFA enforcement, session duration)
- Setting up Cloud Identity for organizations that don't use Google Workspace

> **Similar to:** AWS Organizations + AWS IAM Identity Center configuration (AWS) · Microsoft Entra ID tenant configuration (Azure)

---

### 2.6 Policy Troubleshooter

Policy Troubleshooter answers the question: **"Why does this principal have or not have access to this resource?"** You provide a principal, a resource, and a permission — and it traces through all applicable IAM policies, deny policies, and org policies to explain the result.

**Used at:** Organization · Folder · Project · Resource level

Key use cases:
- Debugging access denied errors
- Verifying that a role binding is working as expected
- Understanding which policy is granting or blocking a specific action

> **Similar to:** AWS IAM Policy Simulator (AWS) · Azure "Check Access" in Role Assignments (Azure)

---

### 2.7 Policy Analyzer

Policy Analyzer lets you **query and audit IAM policies at scale** across your organization. Instead of manually checking every project, you can ask questions like "Which principals have `roles/storage.admin` on any resource in this org?" and get a consolidated answer.

**Used at:** Organization · Folder · Project level

Key use cases:
- Compliance audits — find all users with a specific permission
- Access reviews — list all roles granted to a specific service account
- Security investigations — find who has access to sensitive resources

> **Similar to:** AWS IAM Access Analyzer (AWS) · Azure Resource Graph + Role Assignments query (Azure)

---

### 2.8 Organization Policies

Organization Policies enforce **guardrails on resource configuration** across the hierarchy — independent of IAM. They define what is and is not allowed, regardless of who has what IAM role.

**Used at:** Organization · Folder · Project level

#### Organization Policy
A constraint applied at a hierarchy level that restricts how GCP resources can be configured. Examples:
- Restrict which regions resources can be deployed in
- Prevent public access to Cloud Storage buckets
- Require OS Login on all Compute Engine VMs
- Disable service account key creation
- Enforce uniform bucket-level access on Cloud Storage

#### Constraint
The specific rule enforced by an organization policy. Two types:

| Constraint Type | Description |
|----------------|-------------|
| **List Constraint** | Allow or deny a specific list of values (e.g., allowed regions) |
| **Boolean Constraint** | Enable or disable a specific behavior (e.g., disable VM serial port access) |

> **Similar to:** AWS Service Control Policies — SCPs (AWS) · Azure Policy (Azure)

---

### 2.9 Service Accounts

A Service Account is a **non-human identity** used by applications, Compute Engine VMs, Cloud Run services, and other workloads to authenticate and call GCP APIs programmatically.

**Used at:** Project level (created per project, can be granted roles at any level)

Key characteristics:
- Acts as both an **identity** (can be granted roles) and a **resource** (you can control who can use or manage it)
- Has an email address as its identifier (e.g., `my-sa@my-project.iam.gserviceaccount.com`)
- Authentication via short-lived tokens (recommended) or JSON key files (avoid where possible)
- Can be impersonated by users or other service accounts for delegated access

Best practices:
- Create one service account per workload (not shared)
- Avoid downloading JSON keys — use Workload Identity instead
- Never assign `Owner` or `Editor` basic roles to service accounts

> **Similar to:** IAM Role attached to EC2 Instance Profile / Lambda Execution Role (AWS) · Managed Identity (System-assigned or User-assigned) (Azure)

---

### 2.10 Workload Identity Federation

Workload Identity Federation allows **external workloads** (running outside GCP — on AWS, Azure, GitHub Actions, GitLab, on-prem, or any OIDC/SAML provider) to authenticate to GCP and impersonate a Service Account **without needing a downloadable service account key**.

**Used at:** Project level (Workload Identity Pool is created in a project)

How it works:
1. The external workload presents its own identity token (e.g., an AWS IAM role token or GitHub Actions OIDC token)
2. GCP exchanges it for a short-lived GCP access token
3. The workload can now call GCP APIs as if it were the mapped Service Account

Key use cases:
- GitHub Actions deploying to GCP without storing GCP keys in secrets
- AWS Lambda or EC2 workloads accessing GCP resources
- On-premises applications authenticating to GCP

> **Similar to:** IAM Roles with Web Identity Federation / OIDC provider (AWS) · Azure Workload Identity Federation with Federated Credentials (Azure)

---

### 2.11 Workforce Identity Federation

Workforce Identity Federation allows **human users** from an external Identity Provider (IdP) — such as Okta, Active Directory, Ping Identity, or any SAML/OIDC provider — to sign in to GCP **without needing a Google account**.

**Used at:** Organization level (Workforce Identity Pool is created at org level)

Key difference from Workload Identity Federation:

| | Workload Identity Federation | Workforce Identity Federation |
|--|------------------------------|-------------------------------|
| **Who** | Machines / applications | Human users |
| **Auth source** | AWS, GitHub, GitLab, OIDC/SAML systems | Okta, Azure AD, ADFS, any OIDC/SAML IdP |
| **Level** | Project | Organization |

Key use cases:
- Employees using their corporate SSO (Okta, ADFS) to log in to GCP Console
- Contractors accessing GCP without needing a Google Workspace account
- Organizations that want to use their existing IdP instead of Cloud Identity

> **Similar to:** AWS IAM Identity Center with external IdP (AWS) · Azure Entra ID B2B / External Identities (Azure)

---

### 2.12 Labels

Labels are **key-value metadata pairs** attached to GCP resources for organizing, filtering, and cost tracking purposes.

**Used at:** Project · Resource level

Examples:
```
env = prod
team = data-engineering
cost-center = 1042
```

Key use cases:
- Group resources by environment, team, or application
- Filter resources in the Console or via `gcloud`
- Break down billing reports by label to track spend per team or project
- Used in IAM Conditions to scope access to labeled resources

Labels are **not security boundaries** — they cannot restrict access on their own, but can be used within IAM Conditions.

> **Similar to:** AWS Resource Tags (AWS) · Azure Resource Tags (Azure)

---

### 2.13 Tags (IAM Tags)

Tags in GCP are **strongly typed, policy-enforceable key-value pairs** that attach to the resource hierarchy. Unlike Labels (which are free-form metadata), Tags are controlled through IAM and can be used to enforce Organization Policies and IAM Conditions.

**Used at:** Organization · Folder · Project · Resource level

Key differences from Labels:

| | Labels | Tags |
|--|--------|------|
| **Purpose** | Metadata, billing grouping | Policy enforcement, IAM conditions |
| **Managed by** | Anyone with edit access | Tag Administrators via IAM |
| **Used in Org Policies** | ❌ | ✅ |
| **Used in IAM Conditions** | ❌ | ✅ |
| **Inherited in hierarchy** | ❌ | ✅ |

Key use cases:
- Enforce org policies only on resources tagged `env=prod`
- Grant IAM access conditionally based on resource tags
- Ensure compliance tagging is enforced across all resources

> **Similar to:** AWS Tag Policies + Attribute-Based Access Control (ABAC) with resource tags (AWS) · Azure Resource Tags with Azure Policy tag enforcement (Azure)

---

### 2.14 Privacy and Security

The Privacy and Security section in GCP IAM provides a centralized view of **data access controls, privacy configurations, and security posture** for your organization — including Access Approval, Access Transparency, and related data governance settings.

**Used at:** Organization level

Key sub-features:

**Access Approval** — Requires explicit approval from your team before Google support staff can access your data or resources. You get notified and must approve before any access occurs.

**Access Transparency** — Logs every action taken by Google personnel on your data, providing a near-real-time audit trail of Google staff access.

**Data Access Management** — Manage VPC Service Controls, data residency policies, and customer-managed encryption key (CMEK) enforcement from a unified view.

> **Similar to:** AWS Audit Manager + AWS Artifact for compliance (AWS) · Microsoft Customer Lockbox + Azure Privacy (Azure)

---

### 2.15 Identity-Aware Proxy (IAP)

Identity-Aware Proxy enforces **application-level access control** for web applications, VMs, and internal services hosted on GCP — without needing a VPN. It verifies the identity and context of every request before allowing access to the application.

**Used at:** Project · Resource level

How it works:
- All requests to the application pass through IAP first
- IAP checks the user's Google identity and IAM permissions
- Only principals with the `IAP-secured Web App User` role on that resource are allowed through
- Context-aware access policies can also check device posture, IP address, etc.

Key use cases:
- Secure internal web apps and admin dashboards without a VPN
- Protect Cloud Run, App Engine, GKE, and Compute Engine workloads
- Enable zero-trust access to internal tools for remote employees

> **Similar to:** AWS Verified Access / ALB with Cognito authentication (AWS) · Azure Application Proxy / Azure AD App Proxy (Azure)

---

### 2.16 Audit Logs

Cloud Audit Logs record **who did what, where, and when** across all GCP services. They are the primary tool for security investigation, compliance, and change tracking.

**Used at:** Organization · Folder · Project level

Four types of audit logs:

| Log Type | What it Records | Always On? |
|----------|----------------|------------|
| **Admin Activity** | All admin and configuration changes (e.g., creating VMs, modifying IAM policies) | ✅ Always on |
| **Data Access** | Read/write operations on resource data (e.g., reading a Cloud Storage object) | ❌ Must be enabled |
| **System Event** | GCP-initiated actions (e.g., live migration of a VM) | ✅ Always on |
| **Policy Denied** | When a request is denied due to an org policy or VPC Service Controls | ✅ Always on |

Audit logs are written to Cloud Logging and can be exported to Cloud Storage, BigQuery, or Pub/Sub for long-term retention and analysis.

> **Similar to:** AWS CloudTrail (AWS) · Azure Activity Logs + Diagnostic Logs (Azure)

---

### 2.17 Asset Inventory

Cloud Asset Inventory provides a **complete, searchable snapshot of all GCP resources and IAM policies** across your organization — past and present. It lets you query what resources exist, who has access to them, and how configurations have changed over time.

**Used at:** Organization · Folder · Project level

Key features:
- Search all assets (VMs, buckets, datasets, IAM bindings) across all projects in one query
- Export a point-in-time snapshot of all resources and policies
- Track configuration history — see how a resource's IAM policy changed over the past 35 days
- Feed into Policy Analyzer and Security Command Center for compliance reporting

Key use cases:
- Security audits — find all publicly accessible Cloud Storage buckets across the org
- Change tracking — understand what changed before an incident
- Compliance reporting — generate evidence of resource configurations at a point in time

> **Similar to:** AWS Config + AWS Resource Explorer (AWS) · Azure Resource Graph (Azure)

---

### 2.18 Quotas and System Limits

Quotas define the **maximum amount of a GCP resource or API calls** your project or organization can use. System limits are hard ceilings set by Google that cannot be changed; quotas can often be increased via a request.

**Used at:** Organization · Project level

Two types:

| Type | Description |
|------|-------------|
| **Rate Quotas** | Maximum API calls per second or per minute (e.g., 1000 Compute Engine API requests/min) |
| **Allocation Quotas** | Maximum number of resources that can exist (e.g., 8 CPUs per region, 5 VPCs per project) |

Key use cases:
- Monitor usage to avoid hitting limits that would cause service disruption
- Request quota increases ahead of expected growth or large deployments
- Set budget-style guardrails by keeping quotas intentionally low in dev/test projects

> **Similar to:** AWS Service Quotas (AWS) · Azure Subscription Limits and Quotas (Azure)

---

### 2.19 Essential Contacts

Essential Contacts lets you designate **specific people or groups to receive important notifications** from Google about your GCP organization or project — such as security alerts, billing issues, technical incidents, and compliance notifications.

**Used at:** Organization · Folder · Project level

Notification categories:

| Category | What Triggers It |
|----------|-----------------|
| **Security** | Security advisories, vulnerability alerts, data breach notifications |
| **Technical** | Service disruptions, maintenance notices, deprecation warnings |
| **Billing** | Budget alerts, invoice notifications, billing account issues |
| **Legal** | Terms of service updates, compliance notifications |
| **Suspension** | Project suspension warnings |

By default GCP does not know who to contact in your organization — Essential Contacts must be configured explicitly, otherwise critical alerts may go unnoticed.

> **Similar to:** AWS Account Alternate Contacts (Security, Billing, Operations) (AWS) · Azure Service Health Alerts + Contact information in Azure Account (Azure)

---

### Summary: GCP IAM Components at a Glance

| Component | Purpose | Used At | AWS Equivalent | Azure Equivalent |
|-----------|---------|---------|----------------|-----------------|
| IAM | Core access control — who can do what | Org · Folder · Project · Resource | AWS IAM | Azure IAM / RBAC |
| IAM Roles | Collection of permissions assigned to principals | Org · Folder · Project · Resource | IAM Managed / Inline Policy | RBAC Role Definition |
| IAM Allow Policy | Binds roles to principals on resources | Org · Folder · Project · Resource | Identity / Resource-based Policy | Role Assignment |
| IAM Deny Policy | Explicit deny overriding allow policies | Org · Folder · Project | Explicit Deny in IAM Policy | Deny Assignments |
| IAM Conditions | Attribute-based conditional access rules | Org · Folder · Project · Resource | IAM Policy Condition Keys | Conditional Access |
| PAM | Just-in-time elevated privileged access | Org · Folder · Project | Temporary elevated access via IAM Identity Center | Azure PIM |
| Principal Access Boundaries | Hard ceiling on what a principal can ever access | Org (applied to principals) | IAM Permission Boundaries | Management Group scope restrictions |
| Security Insights | Risk signals and over-permission recommendations | Org · Project | IAM Access Analyzer + Trusted Advisor | Secure Score Identity Recommendations |
| Identity and Organisation | Org-level identity setup and domain linking | Org | AWS Organizations + IAM Identity Center | Entra ID Tenant Configuration |
| Policy Troubleshooter | Debug why access is granted or denied | Org · Folder · Project · Resource | IAM Policy Simulator | Azure "Check Access" |
| Policy Analyzer | Query and audit IAM policies at scale | Org · Folder · Project | IAM Access Analyzer | Azure Resource Graph + Role Assignments |
| Organization Policies | Guardrails on resource configuration | Org · Folder · Project | Service Control Policies (SCPs) | Azure Policy |
| Service Accounts | Non-human / workload identity | Project | IAM Role on EC2 Instance Profile / Lambda | Managed Identity |
| Workload Identity Federation | External machine/app identity without keys | Project | Web Identity Federation / OIDC | Federated Identity Credentials |
| Workforce Identity Federation | External human users via corporate IdP | Org | IAM Identity Center with external IdP | Entra ID External Identities |
| Labels | Free-form metadata for organizing and billing | Project · Resource | AWS Resource Tags | Azure Resource Tags |
| Tags (IAM Tags) | Policy-enforceable key-value pairs | Org · Folder · Project · Resource | ABAC Tags + Tag Policies | Azure Tags + Azure Policy enforcement |
| Privacy and Security | Data access controls and Google staff access visibility | Org | AWS Audit Manager + Artifact | Customer Lockbox + Azure Privacy |
| Identity-Aware Proxy (IAP) | App-level zero-trust access without VPN | Project · Resource | AWS Verified Access / ALB + Cognito | Azure Application Proxy |
| Audit Logs | Full audit trail of all GCP activity | Org · Folder · Project | AWS CloudTrail | Azure Activity + Diagnostic Logs |
| Asset Inventory | Searchable snapshot of all resources and policies | Org · Folder · Project | AWS Config + Resource Explorer | Azure Resource Graph |
| Quotas and System Limits | Max resource usage and API call rate limits | Org · Project | AWS Service Quotas | Azure Subscription Limits |
| Essential Contacts | Designated recipients for GCP critical notifications | Org · Folder · Project | AWS Account Alternate Contacts | Azure Service Health Alerts |