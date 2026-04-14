---
layout: post
title: "AWS Security Specialty: Domain – Data Protection"
date: 2026-03-13
tags: [AWS, AWS Security, Cloud Security, DevSecOps, AWS Security Specialty, Data Protection]
comments: true
published: false
---

Data protection is one of the core domains in the **AWS Certified Security – Specialty** exam. It covers encryption strategies, key management, certificate management, and services that help you discover, classify, and protect sensitive data in the cloud. This post dives deep into the key concepts, services, and best practices you need to master this domain.

---

## 1. Data Protection Concepts

AWS data protection is broadly split into two areas:

- **Encryption at Rest** – Protecting data stored on disk, S3, RDS, EBS, etc.
- **Encryption in Transit** – Protecting data moving over the network using TLS/SSL.

The AWS **Shared Responsibility Model** applies here: AWS secures the underlying infrastructure, while customers are responsible for encrypting their data and managing their keys correctly.

---

## 2. AWS Key Management Service (KMS)

AWS KMS is the backbone of data encryption in AWS. It allows you to create, manage, and control cryptographic keys used to encrypt your data.

### Key Types

| Key Type                        | Description                                                          |
| ------------------------------- | -------------------------------------------------------------------- |
| **AWS Managed Keys**            | Created and managed by AWS on your behalf (e.g., `aws/s3`). No cost. |
| **Customer Managed Keys (CMK)** | You create, manage, and rotate them. Full control over policies.     |
| **AWS Owned Keys**              | Owned and managed entirely by AWS. Not visible to customers.         |

### Key Policies

Every CMK has a **key policy** – this is the primary access control mechanism. Unlike IAM policies, KMS key policies are **resource-based** and must explicitly allow the root account or an IAM principal.

```json
{
  "Sid": "Allow use of the key",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123456789012:role/MyRole" },
  "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
  "Resource": "*"
}
```

### Envelope Encryption

KMS uses **envelope encryption** to protect large amounts of data efficiently:

1. KMS generates a **Data Encryption Key (DEK)**.
2. The DEK encrypts your data.
3. KMS encrypts the DEK itself using the CMK (the DEK never leaves KMS unencrypted).
4. The encrypted DEK is stored alongside the encrypted data.

> **Exam Tip:** `GenerateDataKey` returns a plaintext key AND an encrypted copy. The plaintext key is used to encrypt data locally; the encrypted copy is stored with the data.

### Key Rotation

- **Automatic rotation** is available for CMKs (rotates every year).
- AWS replaces the backing key material but keeps the same Key ID.
- Old key material is **retained** to decrypt data encrypted before the rotation.

### Multi-Region Keys

KMS **Multi-Region Keys** allow you to replicate keys across multiple regions to support disaster recovery or cross-region encryption scenarios (e.g., DynamoDB Global Tables, Global Aurora).

---

## 3. AWS CloudHSM

**CloudHSM** provides dedicated, single-tenant Hardware Security Modules (HSMs) in the AWS cloud. Use it when:

- You need **FIPS 140-2 Level 3** compliance.
- You require full control over the HSM hardware and key material.
- You cannot use AWS-managed keys due to regulatory requirements.

### KMS vs CloudHSM

| Feature         | AWS KMS             | CloudHSM                          |
| --------------- | ------------------- | --------------------------------- |
| Key management  | AWS manages HSMs    | You manage HSMs                   |
| Multi-tenancy   | Multi-tenant        | Single-tenant                     |
| FIPS compliance | FIPS 140-2 Level 2  | FIPS 140-2 Level 3                |
| Cost            | Per-request pricing | Hourly per HSM                    |
| Integration     | Native AWS services | Custom apps via PKCS#11, JCE, CNG |

> **Exam Tip:** If the question mentions **full control**, **single-tenant**, or **FIPS Level 3**, the answer is CloudHSM.

---

## 4. S3 Data Protection

S3 offers multiple layers of encryption and access control.

### Server-Side Encryption (SSE)

| Option       | Description                                                                                                                 |
| ------------ | --------------------------------------------------------------------------------------------------------------------------- |
| **SSE-S3**   | AWS manages keys using AES-256. Header: `x-amz-server-side-encryption: AES256`                                              |
| **SSE-KMS**  | Uses a CMK from KMS. Supports key policies and audit trails via CloudTrail. Header: `x-amz-server-side-encryption: aws:kms` |
| **SSE-C**    | Customer provides their own key. AWS encrypts but does NOT store the key.                                                   |
| **DSSE-KMS** | Dual-layer server-side encryption using KMS (two layers of encryption).                                                     |

### Client-Side Encryption

Data is encrypted **before** being uploaded to S3. The client manages the encryption library and keys (using AWS Encryption SDK or KMS).

### S3 Bucket Policies for Encryption Enforcement

To enforce encryption at upload, use a bucket policy that **denies** `s3:PutObject` unless the correct encryption header is present:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

### S3 Block Public Access

Always enable **S3 Block Public Access** at the account and bucket level to prevent accidental public exposure. This overrides any bucket ACLs or policies that would allow public access.

### S3 Object Lock (WORM)

- Enforces **Write-Once-Read-Many (WORM)** semantics.
- Useful for compliance (SEC Rule 17a-4, FINRA).
- Two modes:
  - **Governance Mode** – Users with special permissions can override.
  - **Compliance Mode** – Nobody (not even root) can delete or overwrite until retention expires.

---

## 5. AWS Certificate Manager (ACM)

ACM simplifies provisioning, managing, and deploying **SSL/TLS certificates**.

### Key Points

- Certificates are **free** for use with AWS services (ALB, CloudFront, API Gateway, etc.).
- ACM **automatically renews** certificates before expiry.
- Private certificates can be issued via **ACM Private CA** (formerly ACM PCA).

### ACM vs IAM Certificate Store

| Feature          | ACM                     | IAM Certificate Store   |
| ---------------- | ----------------------- | ----------------------- |
| Use case         | AWS-integrated services | EC2-hosted applications |
| Auto-renewal     | Yes                     | No                      |
| Certificate type | Public & Private        | Public only             |

> **Exam Tip:** ACM certificates **cannot** be exported for use outside AWS (except from ACM Private CA). For EC2 custom TLS, use IAM or upload the cert manually.

---

## 6. AWS Secrets Manager

Secrets Manager helps you **store, rotate, and manage** sensitive configuration data such as database passwords, API keys, and OAuth tokens.

### Key Features

- **Automatic Rotation** – Built-in Lambda rotation for RDS (MySQL, PostgreSQL, Aurora), Redshift, and DocumentDB. Custom Lambda for other secrets.
- **Fine-grained Access Control** – IAM policies control who can access each secret.
- **Cross-account Access** – Resource-based policies allow sharing secrets across accounts.
- **KMS Integration** – All secrets are encrypted using KMS (CMK or AWS managed).

### Secrets Manager vs Parameter Store

| Feature       | Secrets Manager  | SSM Parameter Store    |
| ------------- | ---------------- | ---------------------- |
| Cost          | Per secret/month | Free (Standard)        |
| Auto-rotation | Yes (native)     | Manual Lambda required |
| Cross-account | Yes              | Limited                |
| Secret size   | Up to 65KB       | Up to 8KB (Advanced)   |
| Versioning    | Yes              | Yes                    |

> **Exam Tip:** If the scenario involves **automatic rotation of database credentials**, always choose Secrets Manager.

---

## 7. AWS Systems Manager Parameter Store

Parameter Store is a lightweight secret/config storage service. It supports two tiers:

- **Standard** – Free, up to 10,000 parameters, 4KB each.
- **Advanced** – Paid, larger values (8KB), parameter policies (TTL, expiry notifications).

### SecureString Parameters

Use `SecureString` type to encrypt parameter values with KMS. Useful for storing credentials that don't need auto-rotation.

---

## 8. Amazon Macie

Amazon Macie is a **data security and data privacy service** that uses machine learning to automatically discover and protect sensitive data in S3.

### What Macie Detects

- **PII** – Names, addresses, credit card numbers, SSNs, passport numbers.
- **Credentials** – AWS secret keys, private keys.
- **Financial data** – Bank account numbers, routing numbers.
- **Healthcare data** – PHI covered under HIPAA.

### How It Works

1. Enable Macie and point it at your S3 buckets.
2. Macie runs **discovery jobs** (one-time or scheduled).
3. Findings appear in the Macie console and can be forwarded to **EventBridge** for automated remediation.
4. Macie integrates with **AWS Security Hub** for centralized finding management.

> **Exam Tip:** Macie is the go-to answer for "automatically discover sensitive/PII data in S3."

---

## 9. AWS Shield & WAF (Data in Transit Protection)

While primarily DDoS/web protection services, Shield and WAF also contribute to data protection by ensuring availability and integrity of data in transit.

- **AWS Shield Standard** – Automatic, always-on DDoS protection (free).
- **AWS Shield Advanced** – Enhanced protection with 24/7 DDoS Response Team (DRT) and cost protection.
- **AWS WAF** – Filter web traffic using rules (SQLi, XSS, rate limits, geo-blocks).

---

## 10. Key Data Protection Best Practices

- ✅ Always encrypt data at rest using SSE-KMS with customer managed keys for auditable access.
- ✅ Use **CloudTrail** to audit all KMS API calls (`Decrypt`, `Encrypt`, `GenerateDataKey`).
- ✅ Enable **S3 Block Public Access** at the organization level via SCP.
- ✅ Rotate CMKs annually and enable automatic rotation where supported.
- ✅ Use **Secrets Manager** for all database credentials; never hardcode secrets in code.
- ✅ Enable **Macie** across all S3 buckets to detect accidental PII exposure.
- ✅ Use **VPC Endpoints** for S3 and KMS to keep traffic off the public internet.
- ✅ Enable **S3 Object Lock** (Compliance mode) for regulatory workloads requiring immutability.
- ✅ Use **ACM** for all public-facing TLS certificates to benefit from auto-renewal.
- ✅ Apply **least privilege** key policies in KMS – never use `kms:*` in production.

---

## Quick Reference: Which Service for Which Use Case?

| Use Case                                    | Service                            |
| ------------------------------------------- | ---------------------------------- |
| Encrypt S3 objects with auditable key usage | SSE-KMS + CloudTrail               |
| Store and rotate DB passwords automatically | Secrets Manager                    |
| Store application config with encryption    | SSM Parameter Store (SecureString) |
| FIPS 140-2 Level 3 / dedicated HSM          | CloudHSM                           |
| Discover PII in S3                          | Amazon Macie                       |
| Manage TLS certificates for ALB/CloudFront  | ACM                                |
| WORM compliance for S3                      | S3 Object Lock (Compliance Mode)   |
| Cross-region encryption for global apps     | KMS Multi-Region Keys              |
| Prevent accidental public S3 exposure       | S3 Block Public Access             |

---

## Summary

The Data Protection domain requires a solid understanding of **KMS**, **S3 encryption options**, **Secrets Manager**, **Macie**, and **ACM**. Focus on knowing when to use each service, the difference between key types, and how policies enforce encryption. Envelope encryption and the distinction between SSE-S3, SSE-KMS, and SSE-C are common exam topics. When in doubt, default to **SSE-KMS + CloudTrail** for encrypted and auditable storage.
