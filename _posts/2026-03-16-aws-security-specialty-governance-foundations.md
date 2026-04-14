---
layout: post
title: "The Comprehensive Guide to AWS Security Specialty (SCS-C03): Domain 6 – Security Foundations & Governance"
date: 2026-03-16
tags: [AWS, AWS Security, Cloud Security, Governance, Compliance, AWS Security Specialty, SCS-C03, SOC3]
comments: true
published: false
---

Domain 6, **Security Foundations and Governance**, is a cornerstone of the **AWS Certified Security – Specialty (SCS-C03)** exam. This guide provides an exhaustive deep dive into the three core tasks of this domain, mapping directly to the official exam skills. If you master the concepts below, you will be prepared to handle any governance or foundation question on the exam.

---

# 🏗️ Task 6.1: Develop a strategy to centrally deploy and manage AWS accounts

Modern cloud security requires managing hundreds or thousands of accounts. Centralization is not optional; it is the standard.

### 🧩 Skill 6.1.1: Deploy and configure organizations by using AWS Organizations

AWS Organizations is the primary service for centralizing management.

- **Hierarchical Structure:** Use **Organizational Units (OUs)** to group accounts by purpose (e.g., Security, Infrastructure, Sandbox, Workloads).
- **Management Account:** The root of the organization. **Best Practice:** Limit resources in this account to billing and cross-account administration only.
- **Enable All Features:** Ensure "All Features" are enabled (not just Consolidating Billing) to use governance policies like SCPs.
- **Consolidated Billing:** Track spending across all accounts and benefit from volume discounts (e.g., S3 storage tiers).

### 🗼 Skill 6.1.2: Implement and manage AWS Control Tower

Control Tower provides the easiest way to set up a "Landing Zone"—a pre-configured, secure environment.

- **New vs. Existing Environments:** You can launch Control Tower in a fresh account or bring existing AWS Organizations under its governance.
- **Guardrails:**
  - **Mandatory:** Automatically applied (e.g., "Disallow changes to IAM roles created by Control Tower").
  - **Optional/Custom:** You can deploy your own SCPs and Config Rules as guardrails.
- **Account Factory:** A standardized way to provision new accounts with pre-defined security baselines (VPC, default IAM, logging).

### 📜 Skill 6.1.3: Implement organization policies to manage permissions

This is a critical area for the SCS-C03 exam.

- **Service Control Policies (SCPs):**
  - Act as a **guardrail**, defining the "maximum" permissions.
  - **Key Tip:** SCPs affect the **Member Account Root User**. They do NOT affect the Management Account.
- **Resource Control Policies (RCPs):**
  - A newer policy type that centrally manages what AWS resources allow (e.g., "Only allow access to S3 buckets from within our Organization ID").
- **AI Service Opt-Out Policies:** Centrally opt-out of AWS using your data to train their AI models.
- **Declarative Policies (Tag/Backup):**
  - **Tag Policies:** Enforce consistent tagging keys and values (e.g., the key `CostCenter` must be present).
  - **Backup Policies:** Enforce backup schedules via AWS Backup across the entire organization.

### 🛡️ Skill 6.1.4: Centrally manage security services

Security should be managed from a dedicated "Security Account," not the Management Account.

- **Delegated Administrator:** Almost every security service (GuardDuty, Macie, Inspector, Security Hub, IAM Access Analyzer) supports a Delegated Admin.
- **Registry:** You register a member account as the admin, allowing it to view findings and manage configurations for all other accounts in the organization.

### 🔑 Skill 6.1.5: Manage AWS account root user credentials

Root users are the "keys to the kingdom."

- **Centralized Root Management:** AWS Organizations now enables centralized management of member account root user credentials.
- **Management Account Root:** Must have hardware MFA. The password should be stored in a physical safe.
- **Break-Glass Procedures:** Create IAM users/roles with `AdministratorAccess` for emergencies. These should have **CloudWatch Alarms** triggered on login to alert the security team immediately.

---

# 🚀 Task 6.2: Implement a secure and consistent deployment strategy

Security must be automated into the "factory" that builds your resources.

### 🛠️ Skill 6.2.1: Use Infrastructure as Code (IaC)

Consistency is the enemy of security vulnerabilities.

- **CloudFormation StackSets:** Use these to deploy a "Security Baseline" (e.g., enabling GuardDuty, creating an IAM Auditor role) to every account in your organization across all regions.
- **Policy as Code (cfn-guard):** Use **CloudFormation Guard** to write rules that validate your templates _before_ deployment (e.g., "All S3 buckets must have versioning enabled").
- **Linting (cfn-lint):** Use **cfn-lint** to check for syntax errors and common security misconfigurations during the CI/CD phase.

### 🏷️ Skill 6.2.2: Use tags to organize AWS resources

Tags are the metadata that drive logic.

- **Grouping:** Group resources by `Department`, `Project`, `CostCenter`, or `Environment` (Dev/Prod).
- **Attribute-Based Access Control (ABAC):** Use tags in IAM policies. (e.g., "Allow user to stop instance ONLY if user.tag.Project == resource.tag.Project").

### 🧱 Skill 6.2.3: Deploy and enforce policies from a central source

- **AWS Firewall Manager:** The essential service for managing firewalls at scale.
  - Centrally manage **AWS WAF**, **AWS Shield Advanced**, **VPC Security Groups**, and **AWS Network Firewall**.
  - **Compliance:** If a new Load Balancer is created without a WAF, Firewall Manager can automatically associate the corporate WAF rules.

### 🤝 Skill 6.2.4: Securely share resources across AWS accounts

- **AWS Resource Access Manager (RAM):** Share resources like Subnets, Transit Gateways, and License Manager configs across accounts. This prevents "VPC Sprawl" by allowing multiple accounts to share a single, well-managed VPC.
- **AWS Service Catalog:** Create "Portfolios" of approved templates. Instead of letting developers create anything, they can only "order" a pre-vetted database or web server from the catalog.

---

# 📊 Task 6.3: Evaluate the compliance of AWS resources

Compliance is the evidence that your governance is actually working.

### 🕵️ Skill 6.3.1: Detect and remediate noncompliant resources

- **AWS Config:** The heart of compliance detection.
  - **Config Rules:** Evaluate whether resources match desired settings (e.g., "Is S3 public?").
  - **Aggregators:** Collect compliance status from every account and region into one central "Audit" account dashboard.
  - **Remediation:** Use **SSM Automation Documents** to fix issues automatically (e.g., "Delete any unapproved Security Group rule").
- **AWS Security Hub:** Aggregates findings from GuardDuty, Macie, Inspector, and Config. It maps these findings to standards like **CIS AWS Foundations** or **PCI DSS**.

### 📁 Skill 6.3.2: Use AWS audit services to collect evidence

- **AWS Audit Manager:** Automates the collection of evidence for audits (HIPAA, SOC 2, etc.). It creates "Assessment Reports" mapping AWS configurations to specific compliance controls.
- **AWS Artifact:** A self-service portal to download audit reports.
  - **SOC 3:** A public-facing summary of the SOC 2 report (often requested by clients).
  - **Agreements:** Accept the Business Associate Addendum (BAA) for HIPAA here.

### 🏗️ Skill 6.3.3: Use AWS services to evaluate architecture

- **AWS Well-Architected Framework Tool:** Helps you review your workload against AWS best practices. Focus on the **Security Pillar** during the exam.
- **Recommendations:** The tool provides a list of "High Risk" items that need immediate attention to align with the framework.

---

# 🎓 Exam Readiness: Fast Facts for Domain 6

| Requirement                                         | Best Service / Feature                          |
| :-------------------------------------------------- | :---------------------------------------------- |
| **Prevent accounts from leaving an Org**            | SCP (`Deny organizations:LeaveOrganization`)    |
| **Centrally manage root MFA for member accounts**   | AWS Organizations (Centralized Root Management) |
| **Enforce S3 encryption before deployment**         | CloudFormation Guard (`cfn-guard`)              |
| **Share subnets without VPC Peering**               | AWS Resource Access Manager (RAM)               |
| **Map AWS configs to HIPAA controls**               | AWS Audit Manager                               |
| **Publicly share AWS's security posture**           | AWS Artifact (SOC 3 Report)                     |
| **Automatically fix a public S3 bucket**            | AWS Config + SSM Automation                     |
| **Centrally manage WAF rules for 200 accounts**     | AWS Firewall Manager                            |
| **Group resources by Cost Center for policy logic** | Tagging + Resource Groups                       |

### 🏁 Summary for Success

To pass Domain 6, shift your thinking from "Single Account" to **"Enterprise Scale."**

1. Use **Organizations/Control Tower** to build the house.
2. Use **SCPs/RCPs** to set the walls.
3. Use **IaC/Firewall Manager** to build the rooms consistently.
4. Use **Config/Security Hub/Audit Manager** to check that everything is still safe.

Good luck with your **SCS-C03** exam! This domain is your ticket to a high score in the Foundations and Governance section.
