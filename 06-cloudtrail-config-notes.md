# AWS CloudTrail & AWS Config — Complete Notes

> **Scope:** SAA-C03 exam prep + DevOps/Solutions Architect interviews
> **Theme:** Governance, Auditing, Compliance, and Change Management

---

## Table of Contents

1. [Why These Two Services Matter](#1-why-these-two-services-matter)
2. [AWS CloudTrail](#2-aws-cloudtrail)
   - 2.1 [What It Is](#21-what-it-is)
   - 2.2 [Event Types](#22-event-types)
   - 2.3 [Trails vs Event History](#23-trails-vs-event-history)
   - 2.4 [Architecture & Delivery](#24-architecture--delivery)
   - 2.5 [Log File Integrity Validation](#25-log-file-integrity-validation)
   - 2.6 [CloudTrail Lake](#26-cloudtrail-lake)
   - 2.7 [Security & Encryption](#27-security--encryption)
   - 2.8 [Multi-Account / Organization Trails](#28-multi-account--organization-trails)
3. [AWS Config](#3-aws-config)
   - 3.1 [What It Is](#31-what-it-is)
   - 3.2 [Core Concepts](#32-core-concepts)
   - 3.3 [Config Rules](#33-config-rules)
   - 3.4 [Remediation](#34-remediation)
   - 3.5 [Conformance Packs](#35-conformance-packs)
   - 3.6 [Aggregators (Multi-Account)](#36-aggregators-multi-account)
   - 3.7 [Pricing Model](#37-pricing-model)
4. [CloudTrail vs Config vs CloudWatch](#4-cloudtrail-vs-config-vs-cloudwatch)
5. [Common Architecture Patterns](#5-common-architecture-patterns)
6. [Exam Callouts (High-Yield)](#6-exam-callouts-high-yield)
7. [Self-Check Questions](#7-self-check-questions)
8. [Interview Questions (Tiered)](#8-interview-questions-tiered)

---

## 1. Why These Two Services Matter

These are the two pillars of **AWS governance**. The exam constantly tests the distinction:

| Question you'll hear | Service |
|----------------------|---------|
| **"Who did what, when, from where?"** (API audit trail) | **CloudTrail** |
| **"What does my resource configuration look like, and is it compliant over time?"** | **AWS Config** |

Mnemonic: **CloudTrail = the security camera (actions)**. **Config = the inspector (state & compliance)**.

---

## 2. AWS CloudTrail

### 2.1 What It Is

CloudTrail records **API calls and account activity** across your AWS account. Every call to an AWS API (via Console, CLI, SDK, or another AWS service) is logged as an **event**.

- **Enabled by default** — the last 90 days of management events are visible in **Event History** with no setup.
- Answers the audit question: *who, what, when, where (source IP), and what response*.
- Global service for most events; events are region-scoped but management events can be aggregated.

```
 User / Service
      |
      | makes API call (e.g. ec2:TerminateInstances)
      v
 +-------------+      +------------------+      +----------------+
 | AWS API     | ---> | CloudTrail       | ---> | S3 Bucket      |
 | (any svc)   |      | (captures event) |      | CloudWatch Logs|
 +-------------+      +------------------+      | EventBridge    |
                                                +----------------+
```

### 2.2 Event Types

| Event Type | What It Captures | Cost | Logged by Default? |
|------------|------------------|------|--------------------|
| **Management Events** | Control-plane ops: create/configure/delete resources (e.g., `RunInstances`, `CreateBucket`, `AttachRolePolicy`) | First copy free | Yes (read/write) |
| **Data Events** | Data-plane ops: high-volume resource activity (e.g., `s3:GetObject`, `lambda:Invoke`, DynamoDB item ops) | **Extra cost**, disabled by default | **No** |
| **Insights Events** | Detects **unusual API activity** (spikes in error rates / call volume) via ML | Extra cost, opt-in | No |

> **Key exam trap:** S3 object-level activity (`GetObject`, `PutObject`) and Lambda `Invoke` are **data events** → must be **explicitly enabled** and cost extra.

**Management events split further into:**
- **Read events** — `Describe*`, `List*`, `Get*`
- **Write events** — `Create*`, `Delete*`, `Modify*`

### 2.3 Trails vs Event History

| Feature | Event History | Trail |
|---------|---------------|-------|
| Setup | None (always on) | Must be created |
| Retention | 90 days | **Indefinite** (in S3) |
| Event types | Management only | Management + Data + Insights |
| Destination | Console view only | S3, CloudWatch Logs, EventBridge |
| Multi-region | No | Yes (single trail can be multi-region) |

> To retain logs **beyond 90 days**, you **must create a trail** that delivers to S3.

### 2.4 Architecture & Delivery

A **Trail** delivers events to:
1. **Amazon S3** — long-term, durable storage (most common). Logs delivered as gzipped JSON roughly every 5 minutes.
2. **CloudWatch Logs** — for real-time monitoring, metric filters, and alarms.
3. **EventBridge** — for automated, event-driven responses (e.g., trigger Lambda when a security group is opened).

**Real-time alerting pattern:**
```
CloudTrail --> CloudWatch Logs --> Metric Filter --> CloudWatch Alarm --> SNS --> Email/Slack
```
or
```
CloudTrail --> EventBridge Rule --> Lambda / SNS / Step Functions
```

> **Latency note:** CloudTrail delivery to S3 is **not instant** (~5–15 min). For near-real-time, use **EventBridge** (faster) rather than waiting on S3 delivery.

### 2.5 Log File Integrity Validation

- Optional feature proving logs were **not tampered with, deleted, or modified** after delivery.
- Uses **SHA-256 hashing** + **SHA-256 with RSA** for digital signing.
- Produces a **digest file** (hourly) in S3 referencing the log files and their hashes.
- Critical for **forensics and compliance** (non-repudiation).

### 2.6 CloudTrail Lake

- A **managed data lake** for capturing, storing, querying, and analyzing CloudTrail events.
- Uses **SQL-based queries** directly — no need to set up Athena.
- Default retention up to **7 years** (configurable, up to 10 years).
- Converts events into **ORC format** optimized for analytics.
- Replaces the older "send to S3 → query with Athena" pattern for many use cases.

### 2.7 Security & Encryption

- Logs in S3 are **encrypted with SSE-S3 by default**; can upgrade to **SSE-KMS** for customer-managed control.
- Protect against deletion with **S3 bucket policies**, **MFA Delete**, and **S3 Object Lock** (WORM).
- Restrict who can `StopLogging` / delete trails via **IAM** and **SCPs**.

### 2.8 Multi-Account / Organization Trails

- **Organization Trail** — created in the management account, automatically logs all member accounts into a **central S3 bucket**. Members can't modify/delete it.
- Best practice: deliver all org logs to a **dedicated, locked-down log-archive account** (AWS Landing Zone / Control Tower pattern).

---

## 3. AWS Config

### 3.1 What It Is

AWS Config continuously **records the configuration of your AWS resources** and lets you **evaluate them against desired states (rules)**. It answers:
- *What does this resource's configuration look like right now?*
- *What did it look like last week?* (point-in-time config history)
- *Is it compliant with my policies?*
- *What changed, and what else was affected?*

> CloudTrail tells you **the API call was made**. Config tells you **the resulting configuration state and whether it's compliant**.

### 3.2 Core Concepts

| Term | Meaning |
|------|---------|
| **Configuration Item (CI)** | Point-in-time snapshot of a resource's attributes, relationships, and metadata |
| **Configuration Recorder** | The engine that detects changes and records CIs (must be turned on) |
| **Configuration History** | Timeline of all CIs for a resource (stored in S3) |
| **Delivery Channel** | Where Config sends snapshots/history (S3 bucket + optional SNS) |
| **Relationships** | Config maps how resources relate (e.g., EC2 → attached SG, EBS, ENI) |

- **Regional service** — must be enabled **per region** (with optional multi-region aggregation).
- Records changes, **not** the API caller (use CloudTrail for that — they're complementary).

### 3.3 Config Rules

Rules evaluate whether resources are **COMPLIANT** or **NON_COMPLIANT**.

**Two sources:**
1. **AWS Managed Rules** — predefined (e.g., `s3-bucket-public-read-prohibited`, `encrypted-volumes`, `required-tags`, `restricted-ssh`).
2. **Custom Rules** — backed by **AWS Lambda** or **Guard (CloudFormation Guard / policy-as-code)**.

**Two trigger types:**
| Trigger | When It Evaluates |
|---------|-------------------|
| **Configuration changes** | Whenever a matching resource changes |
| **Periodic** | On a fixed schedule (e.g., every 1, 3, 6, 12, 24 hrs) |

> **Exam note:** Config rules **detect** non-compliance — they do **not** prevent the action. To *prevent*, use SCPs, IAM, or permission boundaries. To *auto-fix*, use **remediation**.

### 3.4 Remediation

- Config integrates with **AWS Systems Manager (SSM) Automation Documents** to auto-remediate non-compliant resources.
- Can be **automatic** or **manual** (requires approval/trigger).
- Example: a rule detects an unencrypted S3 bucket → triggers SSM doc to enable encryption.

```
Resource changes --> Config Rule evaluates --> NON_COMPLIANT
        --> SSM Automation Document (remediation action)
        --> Resource brought back to compliant state
```

### 3.5 Conformance Packs

- A **collection of Config rules + remediation actions** packaged as a single deployable unit (YAML template).
- Deploy across an account or an **entire Organization**.
- Pre-built packs exist for **PCI-DSS, HIPAA, CIS Benchmarks, NIST**, etc.

### 3.6 Aggregators (Multi-Account)

- A **Configuration Aggregator** collects Config + compliance data from **multiple accounts and regions** into a single view.
- Enables centralized compliance dashboards across an Organization.

### 3.7 Pricing Model

- Charged **per configuration item recorded** + **per rule evaluation**.
- High-churn environments can get expensive → scope the recorder to only the resource types you care about.

---

## 4. CloudTrail vs Config vs CloudWatch

| Dimension | CloudTrail | AWS Config | CloudWatch |
|-----------|-----------|------------|------------|
| **Primary question** | Who made the API call? | What's the resource state / is it compliant? | Is the system performing/healthy? |
| **Records** | API activity (events) | Resource configuration over time | Metrics, logs, performance data |
| **Granularity** | Per API call | Per resource change | Per metric data point |
| **Use case** | Security audit, forensics | Compliance, change tracking | Monitoring, alerting, ops |
| **Time orientation** | Historical event log | Config history + current state | Real-time + historical metrics |

> **The classic exam combo:** *"Who deleted the security group AND was the change compliant?"* → **CloudTrail (who) + Config (compliance/state)** together.

---

## 5. Common Architecture Patterns

**Pattern 1 — Tamper-proof central audit log**
```
All accounts (Org Trail) --> central S3 in log-archive account
   + Log File Validation + S3 Object Lock (WORM) + SSE-KMS
```

**Pattern 2 — Real-time security response**
```
CloudTrail --> EventBridge rule (e.g. AuthorizeSecurityGroupIngress 0.0.0.0/0)
   --> Lambda auto-revokes the rule + SNS alert
```

**Pattern 3 — Continuous compliance**
```
AWS Config recorder ON --> Managed/Custom rules --> NON_COMPLIANT
   --> SSM Automation remediation --> Config Aggregator dashboard (Org-wide)
```

**Pattern 4 — Long-term analytics**
```
CloudTrail Lake (SQL queries, 7-yr retention)
   OR  CloudTrail --> S3 --> Athena --> QuickSight
```

---

## 6. Exam Callouts (High-Yield)

- ✅ **Event History = 90 days, management events only, always on.** Need more → **create a Trail to S3**.
- ✅ **S3 object-level & Lambda Invoke = data events** → must be explicitly enabled, cost extra.
- ✅ **CloudTrail Insights** = detects abnormal API activity via ML.
- ✅ **Log File Validation** uses **SHA-256** to prove integrity (non-repudiation).
- ✅ **Organization Trail** centralizes logs from all member accounts; members can't disable it.
- ✅ **Config = state & compliance**, not the API actor. **CloudTrail = the actor**.
- ✅ Config rules **detect**, they don't **prevent** (use SCP/IAM to prevent, SSM to remediate).
- ✅ **Conformance Packs** = bundle of rules + remediation for standards (PCI/HIPAA/CIS).
- ✅ Config is **per-region**; use **Aggregator** for multi-account/region visibility.
- ✅ Near-real-time alerting → **CloudTrail → EventBridge** (faster than waiting on S3 delivery).
- ✅ Config can show you **resource relationships** (what else is affected by a change).

---

## 7. Self-Check Questions

1. You need to retain API audit logs for 3 years for compliance. What do you configure?
2. A teammate says "CloudTrail isn't logging my S3 GetObject calls." Why, and how do you fix it?
3. How do you prove to an auditor that your CloudTrail logs weren't altered?
4. A security group was just opened to `0.0.0.0/0`. Design a near-real-time auto-remediation.
5. You must enforce that all EBS volumes are encrypted across 50 accounts. Which services and in what roles?
6. What's the difference between *preventing* a misconfiguration and *detecting* one in AWS?

<details>
<summary>Answers</summary>

1. Create a **CloudTrail trail → S3** (Event History only keeps 90 days). Add Object Lock + validation for compliance.
2. `s3:GetObject` is a **data event**, disabled by default. Enable data events on the trail for that bucket.
3. Enable **Log File Integrity Validation** (SHA-256 digest files).
4. **CloudTrail → EventBridge rule** matching the API event → **Lambda** to revoke + **SNS** alert. (Config + SSM also valid for slower remediation.)
5. **AWS Config managed rule `encrypted-volumes`** + **SSM Automation remediation**, deployed org-wide via **Conformance Pack** + **Aggregator** for visibility.
6. **Prevent** = SCP / IAM / permission boundary (blocks the action). **Detect** = AWS Config rules (flags after the fact, optionally remediate).
</details>

---

## 8. Interview Questions (Tiered)

### Tier 1 — Fundamentals
- What is CloudTrail and what problem does it solve?
- Difference between management events and data events?
- What does AWS Config record, and how is it different from CloudTrail?

### Tier 2 — Applied
- How would you build a tamper-evident, centralized audit log across a multi-account org?
- Walk me through detecting and auto-remediating a public S3 bucket.
- Why might CloudTrail logs not appear instantly in S3, and what do you use for real-time needs?
- How does AWS Config pricing work, and how would you control costs in a high-churn account?

### Tier 3 — Architecture / Trade-offs
- Design a compliance pipeline that detects, alerts, remediates, and reports across 100 accounts. Which services play which role?
- Compare querying CloudTrail data via **CloudTrail Lake** vs **S3 + Athena**. When choose which?
- How do CloudTrail, Config, and CloudWatch overlap, and where do they each draw the line? Give a scenario needing all three.
- A resource is non-compliant but the action was technically allowed by IAM. How do you reconcile prevention vs detection in your security posture?

---

*End of notes — good luck with SAA-C03.* 🎯
