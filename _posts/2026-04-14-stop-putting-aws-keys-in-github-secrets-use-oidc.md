---
layout: post
title: "Stop Putting AWS Keys in GitHub Secrets — Use OIDC Instead"
date: 2026-04-14
tags: [AWS, GitHub Actions, OIDC, Security, DevOps, CI/CD, IAM]
comments: true
published: true
---

> *You've been doing it wrong. Here's how to fix it before it costs you.*

### The Problem Nobody Talks About Until It's Too Late

It usually starts innocently. You're setting up your first GitHub Actions deployment pipeline. You generate an AWS IAM user, copy the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` into GitHub Secrets, paste the `aws-actions/configure-aws-credentials` step into your YAML, push — and it works.

Done. Ship it.

But what you've just done is created a **long-lived credential** that:

- Never expires unless you manually rotate it
- Has no awareness of where it's being used from
- Lives in GitHub's secret store, accessible to anyone with repo write access
- Can be accidentally printed in logs, leaked in a PR, or exfiltrated if GitHub itself is compromised
- Needs to be manually rotated across every repo that uses it

This is how breaches happen. A misplaced `echo`, a third-party Action with a supply chain compromise, an over-permissioned IAM user that was "temporary" two years ago — and suddenly your AWS account is mining crypto.

**There's a better way. It's called OIDC, and it takes about 15 minutes to set up.**

### What Is OIDC and Why Should You Care?

**OpenID Connect (OIDC)** is an identity protocol that lets GitHub Actions prove its identity to AWS *without any stored credentials at all*.

Here's the flow at a glance:

1. GitHub Actions runs your workflow
2. It requests an OIDC token (JWT) from GitHub
3. GitHub signs it with your repo's identity info
4. AWS verifies the token against your IAM trust policy
5. AWS returns **temporary** credentials valid for up to 1 hour

In more detail:

```text
GitHub Actions Runner
        │
        │  1. "I am workflow run #1234, on repo YOUR_ORG/YOUR_REPO,
        │      triggered by push to main"
        ▼
  GitHub's OIDC Provider  ──► Signs a short-lived JWT token
        │
        │  2. Sends JWT to AWS STS
        ▼
    AWS STS (AssumeRoleWithWebIdentity)
        │
        │  3. "Does this token match the trust policy
        │      on any of my IAM roles?"
        ▼
    IAM Role  ──► YES ──► Returns temporary credentials
                          (valid for 1 hour, max)
```

The credentials that arrive in your workflow are **temporary**, **scoped**, and **automatically expire**. There is no secret to steal, rotate, or accidentally commit.

### Setting It Up: Step by Step

#### Step 1: Create the OIDC Identity Provider in AWS

First, tell AWS to trust GitHub as an identity provider. Do this once per AWS account.

**Via AWS Console:**

1. Go to **IAM → Identity Providers → Add Provider**
2. Select **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click **Add Provider**

**Via AWS CLI:**

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

#### Step 2: Create the IAM Role

Now create a role that GitHub Actions can assume. The trust policy is where the magic happens — this is where you lock down *exactly* which repos and branches can use this role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

> **The `sub` condition is critical.** Without it, *any* GitHub repo could assume your role. Lock it to your specific org, repo, and branch.

**Common `sub` patterns:**

```text
# Specific branch only
repo:acme/api:ref:refs/heads/main

# Any branch (less secure, use with caution)
repo:acme/api:ref:refs/heads/*

# Pull requests only
repo:acme/api:pull_request

# Specific environment
repo:acme/api:environment:production

# Wildcard — all refs in a repo (not recommended for prod)
repo:acme/api:*
```

**Attach a least-privilege permissions policy to the role.** For example, if you're deploying to ECS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Pro tip:** Scope the `Resource` to specific ARNs wherever possible. `"Resource": "*"` is a shortcut you'll regret.

**Create the role via CLI:**

```bash
# Save trust policy to a file first
aws iam create-role \
  --role-name GitHub-NestJS-Deploy-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach your permissions policy
aws iam attach-role-policy \
  --role-name GitHub-NestJS-Deploy-Role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/YourDeployPolicy
```

Note the role ARN — you'll need it in the next step.

#### Step 3: Update the GitHub Actions Workflow

Now wire it all together. The key changes from the "old way":

1. Add `id-token: write` permission
2. Remove `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` — completely
3. Use `role-to-assume` instead

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 24
          cache: pnpm

      - name: Install pnpm
        uses: pnpm/action-setup@v6
        with:
          version: 10.32.0
          cache: true
          run_install: |
            - args: [--frozen-lockfile, --ignore-scripts=false]

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v6.1.0
        with:
          audience: sts.amazonaws.com
          role-to-assume: arn:aws:iam::YOUR_ACCOUNT_ID:role/GitHub-NestJS-Deploy-Role
          aws-region: us-east-1

      - name: Verify AWS Identity
        run: aws sts get-caller-identity

      - name: Build Docker Image
        run: # docker build ...

      - name: Login to ECR
        run: # aws ecr get-login-password | docker login ...

      - name: Tag Image
        run: # docker tag <image> <ecr-repo-uri>:latest

      - name: Push to ECR
        run: # docker push ...

      - name: Deploy to ECS
        run: # aws ecs update-service ...
```

> **This workflow is an example** to illustrate the concept — not a production-ready pipeline. The deploy steps are intentionally left as comments. The key point: look at what's **not** there — no `AWS_ACCESS_KEY_ID`, no `AWS_SECRET_ACCESS_KEY`, no secrets at all. Just a role ARN, which is a plain identifier, not a credential.

###### Why those two `permissions` lines matter

**`id-token: write`** — This is the critical one for OIDC. It allows the workflow to request a signed JWT token from GitHub. Without it, GitHub will not issue the token and AWS authentication will fail entirely.

> "write" here does **not** mean writing to your repo. It means "allow generating the OIDC token."

**`contents: read`** — Required by `actions/checkout`. The checkout action needs permission to read your repository code. Without it, the checkout step will fail.

###### What `configure-aws-credentials` does under the hood

When this step runs:

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v6.1.0
```

It automatically:

1. Requests the OIDC token from GitHub (using `id-token: write`)
2. Sends it to AWS STS
3. AWS validates the token against your IAM role's trust policy
4. Returns temporary credentials scoped to that role

All of this happens invisibly — no keys, no secrets, no manual token handling.

#### Step 4: Delete Your Old IAM User Credentials

Don't just stop using them. **Delete them.**

```bash
# List access keys for the old IAM user
aws iam list-access-keys --user-name github-actions-user

# Delete each one
aws iam delete-access-key \
  --user-name github-actions-user \
  --access-key-id <AccessKeyId>

# Remove them from GitHub Secrets too
# Settings → Secrets and variables → Actions → Delete
```

#### Before vs After

| | Static IAM Keys | OIDC |
| --- | --- | --- |
| **Credential lifetime** | Until manually rotated | ~15 min to 1 hr |
| **Stored in GitHub** | Yes (Secrets) | No |
| **Rotation required** | Yes, manually | Never |
| **Breach blast radius** | Full key access until rotated | Nothing to steal |
| **Repo-scoped** | No | Yes |
| **Branch-scoped** | No | Yes |
| **Audit trail** | Limited | Full STS call logs in CloudTrail |
| **Setup complexity** | Low | Medium (one-time) |

#### Wrapping Up

Switching to OIDC is one of those rare security improvements that is also simpler to maintain than what it replaces. No secrets to rotate. No IAM users to audit. No keys sitting in GitHub waiting to be leaked.

The 15-minute setup cost pays off the first time you don't have to explain a credentials rotation to your team, or worse — a breach.

**Quick summary of what you did:**

1. Created a GitHub OIDC provider in your AWS account (once per account)
2. Created an IAM role with a trust policy scoped to your exact repo and branch
3. Updated your workflow to use `role-to-assume` with `id-token: write`
4. Deleted the old IAM user keys

That's it. Your CI pipeline now has zero long-lived credentials, and AWS knows exactly where every request is coming from.
