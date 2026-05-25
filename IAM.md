# AWS IAM — Identity and Access Management

> IAM is the **gatekeeper** of AWS. Every single API call to AWS — whether from the console, CLI, SDK, or another AWS service — goes through IAM first. Master this, and the rest of AWS becomes much less scary.

---

## Table of Contents

1. [What is IAM?](#1-what-is-iam)
2. [The Four Core Building Blocks](#2-the-four-core-building-blocks)
3. [The Root User](#3-the-root-user)
4. [IAM Users](#4-iam-users)
5. [IAM Groups](#5-iam-groups)
6. [IAM Roles](#6-iam-roles)
7. [IAM Policies — The JSON Structure](#7-iam-policies--the-json-structure)
8. [Types of Policies](#8-types-of-policies)
9. [Permission Evaluation Logic](#9-permission-evaluation-logic)
10. [Authentication Methods](#10-authentication-methods)
11. [STS — Temporary Credentials](#11-sts--temporary-credentials)
12. [Cross-Account Access](#12-cross-account-access)
13. [Federation and SSO](#13-federation-and-sso)
14. [IAM Tools and Auditing](#14-iam-tools-and-auditing)
15. [Best Practices](#15-best-practices)
16. [Common Scenarios](#16-common-scenarios)
17. [Self-Check](#17-self-check)
18. [Interview Questions](#18-interview-questions)

---

## 1. What is IAM?

IAM is the AWS service that lets you control:

- **Who** can access your AWS account (authentication)
- **What** they can do once inside (authorization)

It's **global** — not tied to a region. An IAM user you create in `us-east-1` works in `ap-south-1` and everywhere else.

It's **free** — no charge for IAM itself, only for the AWS resources users consume.

### The IAM model in one sentence

> A **principal** (user, role, or service) makes a **request** to perform an **action** on a **resource**, and IAM checks **policies** to allow or deny it.

```
┌────────────┐                    ┌─────────────────┐
│ Principal  │  ── request ──►    │   AWS Service   │
│ (user/role)│                    │   (S3, EC2…)    │
└────────────┘                    └────────┬────────┘
                                           │
                                  ┌────────▼────────┐
                                  │   IAM checks    │
                                  │    policies     │
                                  └────────┬────────┘
                                           │
                                  Allow ◄──┴──► Deny
```

---

## 2. The Four Core Building Blocks

Everything in IAM comes down to four concepts:

| Concept | What it is | Example |
|---|---|---|
| **User** | A person or application with long-term credentials | `utkarsh-dev` |
| **Group** | A collection of users sharing permissions | `Developers`, `Admins` |
| **Role** | A set of permissions that can be *assumed temporarily* | `EC2-S3-Access-Role` |
| **Policy** | A JSON document that defines permissions | `AmazonS3ReadOnlyAccess` |

> 💡 **Mental model:** Policies are the *rules*. Users, groups, and roles are the *things* the rules apply to.

---

## 3. The Root User

When you first create an AWS account, you get a **root user** — the account owner with unrestricted access to everything.

### Why root is dangerous

- It can do **anything** — including deleting the entire account
- Its credentials can't be limited by policies
- If compromised, your whole AWS account is gone

### Root user rules (memorize these)

1. **Use it once** to set up the account, then lock it away
2. **Enable MFA** on root immediately
3. **Never create access keys** for root
4. **Create an IAM admin user** for daily admin tasks
5. **Don't use root for anything routine**

### What root *must* be used for (exam favorites)

A small list of tasks that ONLY root can do:

- Closing the AWS account
- Changing the account name, email, or root password
- Restoring IAM user permissions (if an admin locks themselves out)
- Changing AWS support plan
- Enabling MFA delete on S3 buckets
- Some billing-related changes

---

## 4. IAM Users

An IAM user represents **one person or one application** that needs long-term access to AWS.

### Properties of an IAM user

- Has a **unique name** within the account (e.g., `utkarsh-dev`)
- Gets an **ARN** (Amazon Resource Name): `arn:aws:iam::123456789012:user/utkarsh-dev`
- Belongs to **one AWS account**
- Can have **two types of credentials**:
  - **Console password** — for web UI login
  - **Access keys** (Access Key ID + Secret Access Key) — for CLI/SDK/API

### Limits

- Up to **5,000 IAM users** per AWS account
- A user can belong to up to **10 groups**
- A user can have up to **2 active access keys** (useful for key rotation)

> ⚠️ If you need to create more than ~5,000 identities, you're in **federation** territory (covered later).

---

## 5. IAM Groups

A group is a **collection of users**. You attach policies to the group, and every user in it inherits those permissions.

### Why groups matter

Without groups:
```
User A ◄── attach S3 policy, EC2 policy, RDS policy
User B ◄── attach S3 policy, EC2 policy, RDS policy
User C ◄── attach S3 policy, EC2 policy, RDS policy
        (nightmare to maintain)
```

With groups:
```
                 ┌─── User A
Developers ─────┼─── User B
(has policies)   └─── User C
        (one change updates everyone)
```

### Rules about groups

- Groups contain **only users** — they cannot contain other groups (no nesting)
- A user can be in **multiple groups**
- Groups **cannot be principals** — you can't assign permissions *to* a group as the actor; you assign permissions *via* the group's policy to its users

---

## 6. IAM Roles

This is the concept that trips people up the most. Pay attention.

A **role** is an identity with permissions, but it has **no permanent credentials**. Instead, anyone (or anything) authorized can **assume** the role to get **temporary credentials**.

### When to use a role vs a user

| Use a Role when… | Use a User when… |
|---|---|
| An AWS service needs to act (EC2 calling S3) | A human needs long-term console access |
| Another AWS account needs access | An application outside AWS needs static creds |
| A user from another identity system (Google, Okta) needs in | (And even then, federation + roles is better) |

### Examples of role use

- **EC2 Role** — an EC2 instance assumes a role to read from S3, write to DynamoDB, etc. No credentials stored on the instance.
- **Lambda Execution Role** — Lambda assumes this to access other AWS services.
- **Cross-Account Role** — Account A creates a role; users in Account B assume it.
- **Federated Role** — A user logs in via Google/SAML; AWS gives them a role's permissions.

### Two parts of every role

Every IAM role has **two policy attachments**:

1. **Trust Policy** — *Who can assume this role?* (defines the **principal** allowed to assume)
2. **Permissions Policy** — *What can they do once they assume it?* (the actual permissions)

```
┌────────────────────────────────────────────────────┐
│  Role: EC2-S3-ReadOnly                             │
│                                                    │
│  Trust Policy:                                     │
│    "Who can assume me?" → EC2 service              │
│                                                    │
│  Permissions Policy:                               │
│    "What can I do?" → s3:GetObject, s3:ListBucket  │
└────────────────────────────────────────────────────┘
```

> 💡 **No trust policy = no one can assume the role**, no matter how many permissions you grant.

---

## 7. IAM Policies — The JSON Structure

A policy is a JSON document that declares permissions. Every policy follows the same structure.

### Basic anatomy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

### Field-by-field

| Field | What it does |
|---|---|
| `Version` | Policy language version. Always use `"2012-10-17"` |
| `Statement` | An array of permission rules. A policy can have many statements |
| `Sid` | Statement ID — optional, just a human-readable label |
| `Effect` | `"Allow"` or `"Deny"` |
| `Action` | What API calls are covered (e.g., `s3:GetObject`, `ec2:RunInstances`) |
| `Resource` | Which AWS resources the actions apply to (ARNs) |
| `Principal` | *(in resource-based policies)* — who the rule applies to |
| `Condition` | *(optional)* — extra rules (IP, MFA, time of day, tags, etc.) |

### Wildcards

- `"Action": "s3:*"` — all S3 actions
- `"Action": "s3:Get*"` — all S3 read actions starting with "Get"
- `"Resource": "*"` — all resources (use sparingly)

### Example: Policy with a Condition

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:TerminateInstances",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

Translation: "Allow terminating EC2 instances, but only if the user has MFA enabled AND is calling from this office IP range."

---

## 8. Types of Policies

There are several categories of policies — knowing the difference matters for the exam.

### By attachment target

| Type | Attached to | Example |
|---|---|---|
| **Identity-based** | Users, groups, roles | "User X can read S3" |
| **Resource-based** | A resource itself | "Bucket Y can be read by user X" |

> 💡 **S3 bucket policy** is a resource-based policy. So is a Lambda function policy. So is an SQS queue policy.

### By management

| Type | Description | When to use |
|---|---|---|
| **AWS Managed** | Pre-built by AWS (e.g., `AmazonS3ReadOnlyAccess`) | Quick start, common use cases |
| **Customer Managed** | You create and maintain | Custom permissions, reusable across users |
| **Inline** | Embedded directly in one user/role/group | One-off permissions, tight 1:1 binding |

> 🎯 **Best practice:** Use AWS managed policies for common cases, customer managed for your own reusable policies, inline only when you need a strict 1:1 relationship between a policy and an identity.

### Other special policies (advanced — know they exist)

| Type | What it does |
|---|---|
| **Permissions Boundary** | Sets the *maximum* permissions a user/role can have, even if other policies allow more |
| **Service Control Policy (SCP)** | Applies to entire AWS accounts in an Organization. Sets the boundary for everyone, including root |
| **Session Policy** | Passed in when assuming a role — further restricts the session |
| **ACL (Access Control List)** | Legacy XML-based permissions (mostly S3) |

---

## 9. Permission Evaluation Logic

This is one of the **most tested concepts** on the SAA exam. Memorize the flow.

### The rules in order

1. **By default, everything is denied** (implicit deny)
2. **An explicit Allow** in any applicable policy overrides the default deny
3. **An explicit Deny** overrides ANY allow — always wins

```
                ┌─────────────────────┐
                │ Is there an         │
                │ explicit DENY?      │
                └──────────┬──────────┘
                           │
                  ┌────────┴────────┐
                Yes              No
                  │                │
                  ▼                ▼
              ┌───────┐    ┌─────────────────┐
              │ DENY  │    │ Is there an     │
              └───────┘    │ explicit ALLOW? │
                           └────────┬────────┘
                                    │
                            ┌───────┴───────┐
                          Yes             No
                            │              │
                            ▼              ▼
                        ┌───────┐     ┌─────────────────┐
                        │ ALLOW │     │ DENY (implicit) │
                        └───────┘     └─────────────────┘
```

### The golden rule

> **Explicit Deny > Explicit Allow > Implicit Deny**

### Example

If `User A` is:
- In a group with a policy that **Allows** `s3:*`
- But has an inline policy that **Denies** `s3:DeleteObject`

→ User A can do everything in S3 **except delete objects**. The explicit deny wins.

---

## 10. Authentication Methods

Different ways principals prove who they are.

### For human users

| Method | Used for |
|---|---|
| **Username + Password** | Console login |
| **MFA (Multi-Factor Authentication)** | Extra factor on top of password |
| **Federation** | Log in via your company's identity provider |

#### MFA options

- **Virtual MFA** — Google Authenticator, Authy on your phone
- **Hardware MFA** — physical key fob
- **U2F security key** — YubiKey
- **SMS** — discouraged, not as secure

### For programmatic access

| Method | Used for |
|---|---|
| **Access Keys** (Access Key ID + Secret Access Key) | CLI, SDK calls from IAM users |
| **IAM Role + STS** | EC2, Lambda, ECS, anything inside AWS |
| **Temporary credentials via STS** | Federated users, cross-account access |

### Access Key best practices

- **Rotate keys regularly** (every 90 days is common)
- **Never commit them to git** — use git-secrets, AWS's own tools, or environment variables
- **Don't share keys** — one key per user/app
- **Prefer roles over keys** wherever possible

> ⚠️ A leaked access key in a public GitHub repo can lead to thousands of dollars in fraudulent compute charges within minutes. AWS bots scan GitHub for leaked keys and will email you — but attackers scan faster.

---

## 11. STS — Temporary Credentials

**STS (Security Token Service)** issues short-lived credentials, typically valid for 15 minutes to 12 hours.

### How a role assumption works

```
1. User/Service calls sts:AssumeRole
                 │
                 ▼
2. STS checks the role's trust policy
                 │
                 ▼
3. If allowed, STS returns 3 things:
   - Access Key ID (temporary)
   - Secret Access Key (temporary)
   - Session Token (temporary)
                 │
                 ▼
4. Caller uses these for the role's duration
                 │
                 ▼
5. Credentials expire → done
```

### Key STS API calls

| API | Use |
|---|---|
| `AssumeRole` | Cross-account, EC2, Lambda → role |
| `AssumeRoleWithSAML` | SAML-based federation (corporate SSO) |
| `AssumeRoleWithWebIdentity` | Federation from Google, Facebook, Cognito |
| `GetSessionToken` | MFA-protected operations for IAM users |
| `GetFederationToken` | For applications brokering identity |

> 💡 **Why temporary creds are better:** Even if leaked, they expire quickly. Limits blast radius.

---

## 12. Cross-Account Access

A very common AWS pattern — one company often has multiple AWS accounts (dev, staging, prod, billing, security).

### The pattern

```
┌─────────────────────┐                ┌─────────────────────┐
│  Account A (Dev)    │                │  Account B (Prod)   │
│                     │                │                     │
│  User: utkarsh      │  ── assume ──► │  Role: ProdReader   │
│                     │                │  Trust: Account A   │
└─────────────────────┘                │  Perms: S3 read     │
                                       └─────────────────────┘
```

### Steps

1. **In Account B (target):** Create a role. Trust policy specifies Account A (or specific users in it) as the principal.
2. **In Account A (source):** Give your user `sts:AssumeRole` permission for that role's ARN.
3. **User in Account A** calls `AssumeRole`, gets temporary credentials, acts in Account B.

### Why this is powerful

- No need to create duplicate users in every account
- Centralized identity in one account
- Each account stays isolated (blast radius)

---

## 13. Federation and SSO

For organizations with existing identity systems (Active Directory, Okta, Google Workspace), IAM lets you log in via those — no separate IAM users needed.

### Federation types

| Type | Identity Source | Use case |
|---|---|---|
| **SAML 2.0** | Corporate IdP (AD FS, Okta, Azure AD) | Enterprise SSO |
| **Web Identity** | Google, Facebook, Amazon, Cognito | Consumer-facing mobile/web apps |
| **AWS IAM Identity Center** (formerly AWS SSO) | AWS-managed SSO across multiple accounts | Multi-account orgs |

### The flow (SAML, simplified)

```
1. User opens internal SSO portal
2. Authenticates with corporate AD
3. AD returns a SAML assertion
4. Portal calls sts:AssumeRoleWithSAML
5. AWS returns temporary credentials
6. User is logged into AWS console
```

> 🎯 **For SAA exam:** Know that for >5,000 users or enterprise auth, federation is the answer. AWS recommends IAM Identity Center for multi-account setups.

---

## 14. IAM Tools and Auditing

AWS gives you several built-in tools for IAM hygiene.

| Tool | What it does |
|---|---|
| **Credentials Report** | CSV of all users + status of their credentials (last used, MFA, keys) |
| **Access Advisor** | Per user/role/group — which services have they actually used, and when |
| **IAM Access Analyzer** | Finds resources shared externally (S3, IAM roles, KMS, etc.) |
| **Policy Simulator** | Test what a policy will allow/deny before deploying it |
| **CloudTrail** | Logs every API call — who did what, when, from where |

> 💡 Use **Access Advisor** to enforce least privilege — if a role hasn't used a service in 6 months, you probably don't need to grant it.

---

## 15. Best Practices

The AWS-recommended IAM rules. These show up on the exam **and** in real interviews.

1. **Lock down the root user** — MFA, no access keys, use rarely
2. **Create individual IAM users** — never share accounts
3. **Use groups to assign permissions** — not individual users
4. **Grant least privilege** — start with nothing, add only what's needed
5. **Use AWS managed policies** for common cases; customer-managed for your patterns
6. **Use roles, not access keys, for EC2 and Lambda** — never store keys on instances
7. **Enable MFA** for privileged users
8. **Rotate credentials regularly** — passwords and access keys
9. **Use strong password policies** — enforce length, complexity, rotation
10. **Audit with Access Advisor + CloudTrail + Access Analyzer** — review regularly
11. **Use permission boundaries** for delegation — let teams create roles within limits
12. **Use IAM Identity Center for multi-account orgs** — not raw IAM users

---

## 16. Common Scenarios

Quick patterns you'll see in real work and on the exam.

### Scenario 1: EC2 needs to read from S3

❌ **Wrong:** Store access keys on the EC2.
✅ **Right:** Create an IAM role with S3 read permission, attach it to the EC2 instance (via instance profile).

### Scenario 2: Lambda needs to write to DynamoDB

✅ Create an execution role for the Lambda with `dynamodb:PutItem` permission. Lambda automatically assumes it.

### Scenario 3: A developer needs to manage EC2 but not RDS

✅ Create a `Developers` group. Attach `AmazonEC2FullAccess`. Add user to group. Don't attach RDS policies.

### Scenario 4: A contractor needs temporary access to one S3 bucket

✅ Create a role with permission on that bucket only, set max session duration to a few hours, share role assumption with the contractor (via cross-account or federation). Avoid creating a permanent IAM user.

### Scenario 5: A mobile app needs to upload to S3

✅ Use **Cognito** for user authentication, then **Web Identity Federation** to assume a role with the right S3 permissions. Never embed AWS keys in the mobile app.

### Scenario 6: An admin user gets locked out of their account

✅ The **root user** can reset their permissions. This is one of the few legitimate uses of root.

### Scenario 7: You want to ensure no one in your org can disable CloudTrail

✅ Use an **SCP (Service Control Policy)** at the Organization level that denies `cloudtrail:StopLogging`. Even root can't override an SCP.

---

## 17. Self-Check

Can you answer these without scrolling up?

- [ ] What's the difference between an IAM user and an IAM role?
- [ ] What two policies does every role have, and what does each do?
- [ ] In what order does IAM evaluate permissions? Which wins — explicit allow or explicit deny?
- [ ] What's the difference between an identity-based and a resource-based policy?
- [ ] When would you use STS?
- [ ] How would you let an EC2 instance access S3 without storing credentials?
- [ ] What is a permission boundary, and how does it differ from a regular policy?
- [ ] What does an SCP do, and at what level does it apply?
- [ ] Why is federation needed at large organizations?
- [ ] What's the difference between `AssumeRole`, `AssumeRoleWithSAML`, and `AssumeRoleWithWebIdentity`?

---

## 18. Interview Questions

### Conceptual (warm-up)

1. **What is IAM, and why is it important in AWS?**
2. **Explain the difference between authentication and authorization in IAM.**
3. **What are the four main components of IAM?**
4. **What is the difference between an IAM user, group, and role?**
5. **What is the root user, and what tasks require it?**
6. **What is an IAM policy, and what's its basic structure?**
7. **What's the difference between a managed policy and an inline policy?**
8. **What's the difference between an identity-based policy and a resource-based policy?**
9. **What is the principle of least privilege?**
10. **What is MFA, and why should it be enabled?**

### Roles and Trust

11. **What's the difference between an IAM user and an IAM role?**
12. **What is a trust policy, and how is it different from a permissions policy?**
13. **How does an EC2 instance access an S3 bucket without hardcoded credentials?**
14. **What is an instance profile?**
15. **Can two AWS services assume the same role? How would you configure that?**
16. **You attached the right policy to a role, but the EC2 still can't access S3. What might be wrong?**
   *(Hint: trust policy, instance profile attachment, bucket policy, KMS, etc.)*

### Policies and Evaluation

17. **Walk me through how IAM evaluates a request when multiple policies apply.**
18. **What happens when an Allow and a Deny are both present for the same action?**
19. **What is a permissions boundary? Give a real use case.**
20. **What is an SCP (Service Control Policy)? How is it different from a regular IAM policy?**
21. **Can an SCP override a root user's permissions?**
   *(Yes — SCPs apply to everyone in the account, including root.)*
22. **Write a policy that allows reading from an S3 bucket only if the request comes from a specific IP range.**
23. **What's the difference between `"Resource": "*"` and `"Resource": "arn:aws:s3:::*"`?**

### STS and Federation

24. **What is AWS STS, and when would you use it?**
25. **What's the difference between `AssumeRole`, `AssumeRoleWithSAML`, and `AssumeRoleWithWebIdentity`?**
26. **How long do STS credentials last? Can you configure the duration?**
27. **What is identity federation? When would you use it?**
28. **What is AWS IAM Identity Center (formerly AWS SSO)?**
29. **Your company has 10,000 employees who need AWS console access. Do you create 10,000 IAM users? Why or why not?**

### Cross-Account and Real-World

30. **How would you set up access for a user in Account A to read S3 buckets in Account B?**
31. **A developer accidentally pushed AWS access keys to a public GitHub repo. What do you do?**
   *(Hint: rotate immediately, check CloudTrail, audit billing, delete the old key.)*
32. **How would you detect that an IAM user hasn't used their access keys in 90 days?**
   *(Hint: Credentials Report, Access Advisor.)*
33. **A user complains they can't access an S3 bucket. Walk through your troubleshooting steps.**
   *(Hint: identity policy → bucket policy → ACL → block public access → KMS → VPC endpoint policy.)*
34. **How do you ensure that newly-created S3 buckets in your org are never publicly accessible?**
   *(Hint: S3 Block Public Access at account level, SCP denying changes, AWS Config rules.)*

### Scenario-Based (Senior level)

35. **You need to give a third-party vendor temporary access to your AWS account. How do you do it securely?**
   *(Hint: cross-account role with external ID condition, limited permissions, short session.)*
36. **You're designing a multi-account AWS setup for a startup. Walk me through your IAM strategy.**
   *(Hint: AWS Organizations, IAM Identity Center, SCPs, per-account roles, no IAM users in member accounts.)*
37. **How would you implement a "break-glass" admin role that's only used in emergencies?**
   *(Hint: separate role, MFA required, alerts on use, restricted trust policy, CloudTrail monitoring.)*
38. **A Lambda function needs to access a database in another VPC. How do you handle the authentication?**
   *(Hint: Lambda execution role, VPC config, Secrets Manager + IAM, or IAM database auth for RDS.)*
39. **Explain how you'd rotate access keys for hundreds of IAM users with zero downtime.**
   *(Hint: create second key → distribute → deactivate old → verify → delete. Use Secrets Manager or Parameter Store.)*
40. **How is IAM different in AWS Organizations vs a standalone account?**

### Tricky Conceptual

41. **Can you attach a policy to a group? Can a group be a principal in a policy?**
   *(Yes to first, no to second.)*
42. **Can an IAM user belong to multiple groups? Can a group contain another group?**
   *(Yes, no — groups are flat.)*
43. **What's the maximum size of an IAM policy?**
   *(Managed policy: 6,144 chars. Inline: varies by entity. Worth knowing.)*
44. **Why is the principle of least privilege difficult to achieve in practice, and how do you approach it?**
45. **Can a resource-based policy grant access to a principal in a different AWS account, without that account explicitly allowing it?**
   *(No — cross-account always requires both sides.)*

---

**Next up:** `02-ec2.md` — EC2 fundamentals, instance types, AMIs, key pairs, security groups, and how it all ties back to the IAM concepts here.
