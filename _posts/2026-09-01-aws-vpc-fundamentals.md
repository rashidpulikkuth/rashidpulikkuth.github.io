---
layout: post
title: "AWS VPC Fundamentals: A Practical Guide to Networking in the Cloud"
date: 2026-09-01
tags: [AWS, VPC, Networking, Cloud, AWS Security]
comments: true
published: true
---

Many AWS workloads, such as **EC2** and **RDS**, run within a **Virtual Private Cloud (VPC)**. **Lambda** functions can optionally be connected to a VPC when they need access to private VPC resources. Get the networking wrong and nothing connects. Get it right once, and the same patterns scale from a single server to production multi-tier architectures.

This guide walks through VPC networking in three layers:

1. **IP addressing** — private ranges, CIDR math, subnet masks
2. **Routing** — public vs private subnets, Internet Gateway, NAT Gateway
3. **Security** — Network ACLs and Security Groups

---

## Part 1 — IP Addressing

### 1.1 Private IPv4 Ranges (RFC 1918)

VPCs use **private IP space** — addresses reserved for internal networks, not routable on the public internet:

| Class | CIDR | Address range | Best for |
|:-----:|------|---------------|----------|
| **A** | `10.0.0.0/8` | 10.0.0.0 – 10.255.255.255 | Large cloud deployments |
| **B** | `172.16.0.0/12` | 172.16.0.0 – 172.31.255.255 | Medium networks |
| **C** | `192.168.0.0/16` | 192.168.0.0 – 192.168.255.255 | Small networks |

> **Watch out:** Class B is **`172.16.0.0/12`**, not `172.1.0.0`. Only `172.16.0.0` through `172.31.255.255` is private.

Pick a range that **does not overlap** with on-premises networks, peered VPCs, or planned expansions. **`10.0.0.0/16`** is the most common starting point in AWS.

---

### 1.2 AWS CIDR Limits

AWS enforces the same bounds on VPC and subnet CIDR blocks:

| | Smallest block | Largest block | IP addresses |
|---|:---:|:---:|:---:|
| **VPC** | `/28` | `/16` | 16 – 65,536 |
| **Subnet** | `/28` | `/16` | 16 – 65,536 |

Additional rules:

- Subnet CIDR must be a **subset** of the VPC CIDR (never larger).
- AWS **reserves 5 IPs** per subnet → a `/28` gives **11 usable** addresses (16 − 5).
- A VPC supports up to **5 IPv4 CIDR blocks** (primary + secondary) for expansion without rebuilding.

| Prefix | Total IPs | Usable in AWS | Typical use |
|:------:|:---------:|:-------------:|-------------|
| `/16` | 65,536 | 65,531 | Entire VPC |
| `/24` | 256 | 251 | One subnet per tier/AZ |
| `/28` | 16 | 11 | Minimal / test subnet |

---

### 1.3 CIDR Notation — The Core Formula

Every IPv4 address is **32 bits**. In `/N` notation, **N bits = network**, the rest = **host**:

```
32 − N  =  host bits
2^(host bits)  =  total IP addresses
```

**Example — `10.0.0.0/20`:**

| | Value |
|---|:---:|
| Network bits | 20 |
| Host bits | 32 − 20 = **12** |
| Total addresses | 2^12 = **4,096** |

---

### 1.4 Subnet Masks — Converting `/N` to Dotted Decimal

A subnet mask shows which bits are network vs host. To convert `/N`:

**Method 1 — Bit line**

Each octet is 8 bits. Every bit position has a fixed **decimal weight** — add the weights where the bit is `1`:

```
Bit position:   128   64   32   16    8    4    2    1
                  ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
Binary:           1    1    1    1    0    0    0    0   =  128+64+32+16  =  240
```

Mark the first N bits as `1` (network), the rest as `0` (host), split into four octets:

```
/20  →  20 network bits, 12 host bits

Bit weights:  128 64 32 16  8  4  2  1  (repeat per octet)

  1st octet              2nd octet              3rd octet              4th octet
 11111111            .  11111111            .  11110000            .  00000000
 128+64+32+16+8+4+2+1   (same → 255)            128+64+32+16=240         0
    = 255             .     255               .     240               .     0

Subnet mask:  255 . 255 . 240 . 0   →   255.255.240.0
```

Full octets (`11111111`) always sum to **255**. The 3rd octet is the interesting one — only the first 4 bits are network:

| Bit | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
|:---:|:---:|:--:|:--:|:--:|:-:|:-:|:-:|:-:|
| Value | 1 | 1 | 1 | 1 | 0 | 0 | 0 | 0 |
| Sum | 128 + 64 + 32 + 16 | | | | | | | **= 240** |

**Method 2 — Octet shortcut**

| Octet type | Mask value |
|------------|:----------:|
| All network bits | **255** |
| Mixed (partial) | **256 − 2^host_bits_in_octet** |
| All host bits | **0** |

For `/20`, the boundary is in the 3rd octet → 256 − 2^4 = **240** → mask is **`255.255.240.0`**.

**Quick reference:**

| CIDR | Subnet mask | Host bits | Total IPs |
|:----:|:-----------:|:---------:|:---------:|
| `/16` | 255.255.0.0 | 16 | 65,536 |
| `/20` | 255.255.240.0 | 12 | 4,096 |
| `/24` | 255.255.255.0 | 8 | 256 |
| `/26` | 255.255.255.192 | 6 | 64 |
| `/28` | 255.255.255.240 | 4 | 16 |

> **Exam tip:** Memorize the bit weights left to right: **128 · 64 · 32 · 16 · 8 · 4 · 2 · 1**. Partial-octet mask values map directly — `/25`=128, `/26`=192, `/27`=224, `/28`=240, `/29`=248, `/30`=252.

---

### 1.5 Network Increments — Where the Next Block Starts

Block size (`2^host_bits`) tells you **how many IPs** fit in a range. The **increment** tells you **where the next range begins** — and depends on **which octet** the prefix boundary falls in.

For `/20` (mask `255.255.240.0`):

```
256 − 240 = 16   →   increment by 16 in the 3rd octet
```

```
10.0.0.0/20
10.0.16.0/20
10.0.32.0/20
10.0.48.0/20
...
```

**Three-step mental model:**

1. `32 − prefix = host bits` → `2^host_bits` = block size
2. Find the **boundary octet** (where network bits end)
3. In that octet: **`256 − mask_value = increment`**

---

## Part 2 — VPC Architecture and Routing

### 2.1 Core Components

| Component | What it does |
|-----------|-------------|
| **VPC** | Isolated network boundary (e.g. `10.0.0.0/16`) |
| **Subnet** | A slice of the VPC CIDR, tied to one Availability Zone |
| **Route table** | Directs traffic — decides *where* packets go |
| **Internet Gateway (IGW)** | Connects the VPC to the internet (bidirectional, public resources) |
| **NAT Gateway** | Outbound-only internet for private subnets (via Elastic IP) |
| **NACL** | Subnet-level firewall — decides *if* traffic is allowed (stateless) |
| **Security Group** | Instance-level firewall — decides *if* traffic is allowed (stateful) |

Every subnet is linked to **one route table** and **one NACL**.

---

### 2.2 Public vs Private Subnets

| | Public subnet | Private subnet |
|---|--------------|----------------|
| **Route to internet** | `0.0.0.0/0 → IGW` | `0.0.0.0/0 → NAT Gateway` (if outbound needed) |
| **Instance IP** | Public IPv4 or Elastic IP | Private IP only |
| **Inbound from internet** | Possible (if SG/NACL allow) | Blocked |
| **Typical workloads** | ALB, NAT Gateway, bastion | App servers, databases |

**Internet Gateway setup:**

1. Create and attach an IGW to the VPC.
2. Add to the subnet route table: **`0.0.0.0/0 → igw-xxxxx`**
3. Assign the EC2 instance a **public IPv4 address** or Elastic IP.

> **Key point:** The IGW serves **public subnets**. Private subnets use a **NAT Gateway** for outbound-only access.

---

### 2.3 NAT Gateway — Private Subnet Internet Access

When private instances need outbound internet (updates, API calls) without accepting inbound connections:

```
  Private Subnet                    Public Subnet                 Internet
 ┌─────────────────┐              ┌─────────────────┐
 │  EC2 (private)  │              │  NAT Gateway    │
 │  10.0.2.50      │──0.0.0.0/0──▶│  + Elastic IP   │──0.0.0.0/0──▶ IGW ──▶
 └─────────────────┘              └─────────────────┘
        │                                   │
   route table:                        route table:
   10.0.0.0/16 → local                 10.0.0.0/16 → local
   0.0.0.0/0   → NAT                   0.0.0.0/0   → IGW
```

**Setup checklist:**

| Step | Action |
|:----:|--------|
| 1 | Create public subnet with route `0.0.0.0/0 → IGW` |
| 2 | Attach an Internet Gateway to the VPC |
| 3 | Create NAT Gateway in the public subnet (requires Elastic IP) |
| 4 | Add route `0.0.0.0/0 → NAT Gateway` to the **private** subnet route table |
| 5 | Launch private EC2 — no public IP needed |

**Traffic flow:** Private IP → NAT (SNAT to Elastic IP) → IGW → Internet. Responses follow the reverse path. Inbound connections from the internet cannot reach the private instance.

**Common mistakes:**

| Mistake | Correct approach |
|---------|-----------------|
| `0.0.0.0/16` instead of `0.0.0.0/0` | Default internet route is always **`0.0.0.0/0`** |
| NAT Gateway in a private subnet | NAT must sit in a **public** subnet with IGW route |
| Missing Elastic IP on NAT | NAT Gateway requires an EIP |
| Wrong route table on subnet | Verify each subnet → correct route table |

---

### 2.4 Reference Architecture

```
VPC: 10.0.0.0/16
│
├── Public Subnet   10.0.1.0/24  (AZ-a)   → 0.0.0.0/0 → IGW
│   ├── NAT Gateway
│   └── ALB / Bastion
│
├── Private Subnet  10.0.2.0/24  (AZ-a)   → 0.0.0.0/0 → NAT
│   └── App servers
│
└── Private Subnet  10.0.3.0/24  (AZ-b)   → 0.0.0.0/0 → NAT
    └── RDS / databases
```

For production: spread subnets across **multiple AZs**. Use one NAT Gateway per AZ for high availability, or share one NAT to reduce cost.

---

## Part 3 — Network Security

Route tables control **where** traffic goes. **NACLs** and **Security Groups** control **whether** it is permitted.

### 3.1 Network ACLs (NACLs)

A NACL is a **stateless, subnet-level** firewall. It filters traffic entering or leaving a subnet — every resource in that subnet is affected.

#### Rule evaluation

Rules are numbered **1–32766**. AWS evaluates from **lowest to highest**. **First match wins** — evaluation stops immediately.

```
Traffic → Rule 50?  → match → APPLY & STOP
              ↓ no
          Rule 100? → match → APPLY & STOP
              ↓ no
          Rule 200? → match → APPLY & STOP
              ↓ no
          Rule *    → APPLY & STOP   (implicit default)
```

#### The `*` rule

Every NACL ends with an implicit rule shown as **`*`** in the console. You cannot edit or remove it.

| NACL type | Rule `*` behavior |
|-----------|-------------------|
| **Default NACL** | ALLOW everything not matched above |
| **Custom NACL** | DENY everything not matched above |

On a custom NACL, traffic you forget to explicitly allow is **silently denied**.

#### Correct rule set — HTTPS inbound

| Rule # | Direction | Protocol | Port | Source | Action |
|:------:|:---------:|:--------:|:----:|:------:|:------:|
| 100 | Inbound | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110 | Inbound | TCP | 1024–65535 | 0.0.0.0/0 | ALLOW |
| * | Inbound | All | All | 0.0.0.0/0 | DENY |

Rule 100 allows HTTPS. Rule 110 allows return traffic on ephemeral ports (required because NACLs are **stateless**). Everything else is denied by `*`.

#### Danger — allow-all at a low rule number

A broad **ALLOW** at rule 50 matches **all traffic first**. Every rule above it is never evaluated:

```
Traffic → Rule 50 ALLOW all? → YES → ALLOW & STOP
              ↓ (never reached)
          Rule 200 DENY bad IP
          Rule 300 ALLOW 443 only
```

| Rule # | Action | What actually happens |
|:------:|:------:|-----------------------|
| **50** | ALLOW all | Matches everything — evaluation stops |
| 200 | DENY `203.0.113.50` | Never evaluated |
| 300 | ALLOW TCP 443 | Never evaluated |

The subnet looks protected but is **wide open**. Fix by placing specific rules at the **lowest** numbers:

| Rule # | Action |
|:------:|--------|
| 50 | DENY `203.0.113.50/32` |
| 100 | ALLOW TCP 443 from `0.0.0.0/0` |
| * | DENY (implicit) |

---

### 3.2 NACL vs Security Group

| | NACL | Security Group |
|---|------|----------------|
| **Scope** | Subnet | Instance (ENI) |
| **State** | Stateless | **Stateful** |
| **Rules** | Allow **and** Deny | **Allow only** |
| **Evaluation** | Lowest rule # first, first match wins | All rules checked; any allow match permits traffic |
| **Return traffic** | Must allow ephemeral ports explicitly | Automatically allowed |
| **Default** | Default NACL: allow all | Default SG: allow all outbound, deny inbound |

**Defense in depth:**

```
Internet  →  NACL (subnet edge)  →  Security Group (instance)  →  EC2
```

Use **Security Groups** for everyday access control (e.g. allow port 443 from the ALB security group). Use **NACLs** as a coarse subnet boundary — block IP ranges, add a deny layer, or meet compliance requirements.

---

## Cheat Sheet

| Topic | Remember |
|-------|----------|
| Private ranges | `10.0.0.0/8` · `172.16.0.0/12` · `192.168.0.0/16` |
| AWS CIDR limits | `/16` (largest) to `/28` (smallest) · 5 IPs reserved per subnet |
| Address count | `32 − prefix = host bits` → `2^host_bits` addresses |
| Subnet mask | Network bits = `1`, host bits = `0` → convert octets to decimal |
| Increment | In boundary octet: `256 − mask_value` |
| Public subnet | Route `0.0.0.0/0 → IGW` + public IP on instance |
| Private outbound | Route `0.0.0.0/0 → NAT Gateway` in public subnet |
| NACL evaluation | Lowest number first · **first match wins** · `*` catches the rest |
| NACL danger | ALLOW-all at a low rule # makes every rule above it useless |

---

## Key Takeaways

1. **Plan IP space first.** Choose a private range with no overlaps; size your VPC (`/16`) and subnets (`/24`) before deploying resources.
2. **CIDR has two calculations.** Block size (`2^host_bits`) tells you capacity; increment (`256 − mask`) tells you where the next subnet starts.
3. **Routing defines public vs private.** A single route entry (`0.0.0.0/0 → IGW` or `→ NAT`) determines how a subnet reaches the internet.
4. **NAT Gateway lives in a public subnet.** Private instances get outbound-only access; inbound from the internet is blocked.
5. **NACL rule order matters.** Lowest number wins. Put DENY rules first, then ALLOW rules, and never place a broad ALLOW at the top.

Master these five concepts and the standard AWS VPC diagram — subnets, route tables, IGW, NAT, NACL, Security Group — becomes straightforward to read, design, and troubleshoot.
