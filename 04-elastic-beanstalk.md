# AWS Elastic Beanstalk — Deployment & Orchestration

> Elastic Beanstalk is AWS's **Platform-as-a-Service (PaaS)** offering. You hand it your code; it provisions and manages the EC2 instances, load balancer, Auto Scaling group, and health monitoring for you. It's "managed infrastructure without giving up control" — everything it creates is still standard AWS resources you can see and tweak.

---

## Table of Contents

1. [What is Elastic Beanstalk?](#1-what-is-elastic-beanstalk)
2. [Core Concepts and Hierarchy](#2-core-concepts-and-hierarchy)
3. [What Beanstalk Provisions For You](#3-what-beanstalk-provisions-for-you)
4. [Configuration Presets](#4-configuration-presets)
5. [The High Availability Preset](#5-the-high-availability-preset)
6. [Custom Configuration](#6-custom-configuration)
7. [Deployment Policies — The Big Topic](#7-deployment-policies--the-big-topic)
8. [Blue/Green Deployment](#8-bluegreen-deployment)
9. [Choosing a Deployment Strategy](#9-choosing-a-deployment-strategy)
10. [Environment Types — Web Server vs Worker](#10-environment-types--web-server-vs-worker)
11. [Beanstalk and Other AWS Services](#11-beanstalk-and-other-aws-services)
12. [Best Practices](#12-best-practices)
13. [Common Scenarios](#13-common-scenarios)
14. [Self-Check](#14-self-check)
15. [Interview Questions](#15-interview-questions)

---

## 1. What is Elastic Beanstalk?

Beanstalk sits between raw EC2 (full control, full responsibility) and fully serverless (no control, no responsibility):

```
  More control                                   Less control
  More work                                      Less work
  ◄──────────────────────────────────────────────────────►
  EC2          Elastic Beanstalk          ECS/Fargate      Lambda
 (DIY)         (managed EC2/PaaS)         (containers)    (serverless)
```

### The pitch

You upload your application code (or a Docker image), and Beanstalk automatically handles:
- Capacity provisioning (EC2 instances)
- Load balancing
- Auto Scaling
- Application health monitoring
- Deployment orchestration

### Key principles

- **You only pay for the underlying resources** — Beanstalk itself is free. The EC2, ELB, etc. it creates are billed normally.
- **No loss of control** — every resource it creates is a standard AWS resource you can inspect and modify.
- **Supports many platforms** — Java, .NET, Node.js, Python, Ruby, PHP, Go, and Docker.

> 💡 **Mental model:** Beanstalk is an *orchestration layer* on top of services you already know (EC2, ELB, ASG, CloudWatch). It's not a new compute engine — it's automation that wires them together.

---

## 2. Core Concepts and Hierarchy

Beanstalk has a clear object hierarchy. Get this straight first.

```
Application  (the logical project — e.g., "OrderService")
   │
   ├── Application Version  (a deployable build — v1.0, v1.1, v1.2…)
   │       │  (stored as a bundle in S3)
   │
   └── Environment  (a running instance of a version)
           │  e.g., "OrderService-prod", "OrderService-staging"
           │
           └── Environment Configuration  (settings: instance type, scaling, etc.)
```

| Term | Meaning |
|---|---|
| **Application** | The top-level container for everything related to one app |
| **Application Version** | A specific, labeled build of your code (a bundle in S3) |
| **Environment** | A running deployment of one version + its AWS resources |
| **Environment Tier** | Web Server tier or Worker tier (covered later) |
| **Configuration** | The settings that define how the environment behaves |

> 💡 You can run **multiple environments** of the same application — e.g., `dev`, `staging`, `prod` — each running a different version.

---

## 3. What Beanstalk Provisions For You

When you create an environment, Beanstalk can spin up:

| Resource | Role |
|---|---|
| **EC2 instances** | Run your application |
| **Auto Scaling Group** | Add/remove instances based on load |
| **Elastic Load Balancer** | Distribute traffic (in load-balanced environments) |
| **Security Groups** | Network firewall rules |
| **CloudWatch alarms** | Monitor health and trigger scaling |
| **S3 bucket** | Store application versions and logs |
| **Host manager (on each instance)** | Reports health, runs deployments, rotates logs |

All of this is created from your chosen **preset** or **custom config**.

---

## 4. Configuration Presets

When you create an environment, Beanstalk offers **presets** — pre-built bundles of configuration so you don't have to set every option manually. Think of them as starting templates.

### The main presets

| Preset | What it sets up | Best for |
|---|---|---|
| **Single instance** | One EC2 instance, **no load balancer** | Dev/test, low cost, non-critical |
| **Single instance (Spot)** | One EC2 on Spot pricing | Cheapest dev/test |
| **High availability** | Load balancer + Auto Scaling across multiple AZs | Production |
| **High availability (Spot + On-Demand)** | Mixed pricing, load-balanced, multi-AZ | Cost-optimized production |
| **Custom** | You define everything yourself | Specific requirements |

### Single instance vs load-balanced — the core trade-off

```
Single Instance:                  High Availability (load-balanced):

   ┌──────────┐                      ┌──────────────┐
   │  1 EC2   │                      │ Load Balancer│
   │          │                      └──────┬───────┘
   └──────────┘                    ┌────────┼────────┐
                                    ▼        ▼        ▼
   No load balancer            ┌──────┐ ┌──────┐ ┌──────┐
   Cheapest                    │ EC2  │ │ EC2  │ │ EC2  │
   No HA — single point        │ AZ-a │ │ AZ-b │ │ AZ-c │
   of failure                  └──────┘ └──────┘ └──────┘
                               Auto Scaling across AZs
                               Survives instance/AZ failure
```

| | Single Instance | High Availability |
|---|---|---|
| Load balancer | ❌ No | ✅ Yes |
| Auto Scaling | Min/max = 1 | Scales across AZs |
| Cost | Lowest | Higher |
| Resilience | Single point of failure | Survives failures |
| Use case | Dev, test, demos | Production |

> 🎯 **Exam/interview framing:** Single instance = cheap, not resilient. High availability = resilient, costs more. Pick based on whether downtime is acceptable.

---

## 5. The High Availability Preset

This is the production-grade preset. It's worth understanding what it actually wires up, because it's the same HA architecture you saw in the EC2 chapter — Beanstalk just builds it for you.

### What it creates

- An **Elastic Load Balancer** (ALB by default) as the single entry point
- An **Auto Scaling Group** with min > 1, spread across **multiple Availability Zones**
- **CloudWatch alarms** to scale out/in based on metrics
- **Health checks** so unhealthy instances are replaced automatically

### Why it's "highly available"

- **Multiple AZs** → if one AZ fails, others keep serving
- **Multiple instances** → if one instance dies, the ELB routes around it and the ASG replaces it
- **No single point of failure** at the compute layer

```
                  ┌──────────────┐
   Users ───────► │ Load Balancer│  (entry point, health checks)
                  └──────┬───────┘
            ┌────────────┼────────────┐
            ▼            ▼            ▼
        ┌──────┐     ┌──────┐     ┌──────┐
        │ EC2  │     │ EC2  │     │ EC2  │   ◄── Auto Scaling Group
        │ AZ-a │     │ AZ-b │     │ AZ-c │       (min 2+, multi-AZ)
        └──────┘     └──────┘     └──────┘
```

> 💡 The HA preset is essentially "ALB + multi-AZ ASG + health checks" — the textbook resilient web architecture — assembled with a few clicks instead of manual setup.

---

## 6. Custom Configuration

When presets don't fit, you customize. Beanstalk lets you override almost any setting.

### What you can customize

- **Instance type** and number (min/max in the ASG)
- **Load balancer type** (ALB, NLB, CLB) and listener/SSL settings
- **Scaling triggers** (which metric, thresholds, cooldowns)
- **VPC, subnets, and security groups**
- **Environment variables** for your app
- **Managed platform updates** schedule
- **Rolling deployment settings** (batch size, health thresholds)
- **Database** (Beanstalk can create an RDS instance — but see the warning below)

### `.ebextensions` — configuration as code

You can include a `.ebextensions/` folder in your app bundle with `.config` files (YAML/JSON) that customize the environment — install packages, set env vars, run commands, configure resources.

```yaml
# .ebextensions/01-packages.config
packages:
  yum:
    git: []
    htop: []
option_settings:
  aws:elasticbeanstalk:application:environment:
    APP_ENV: production
```

> ⚠️ **Important production warning:** Don't let Beanstalk create your **RDS database inside the environment**. If you do, deleting the environment deletes the database. For production, create RDS **separately** and connect to it via environment variables — so the DB survives environment teardown/recreation.

---

## 7. Deployment Policies — The Big Topic

When you deploy a new application version, Beanstalk needs to replace the old code with the new code across your instances. **Deployment policies** define *how* that rollout happens — controlling the trade-off between speed, cost, and risk/downtime.

### The five policies

| Policy | How it works | Downtime | Extra cost | Rollback |
|---|---|---|---|---|
| **All at once** | Deploys to all instances simultaneously | ❌ Yes (brief) | None | Manual redeploy |
| **Rolling** | Deploys in batches; some instances serve old, some new | No (reduced capacity) | None | Redeploy old version |
| **Rolling with additional batch** | Adds a temp batch first, keeps full capacity during deploy | No (full capacity) | Small (temp instances) | Redeploy old version |
| **Immutable** | Launches a brand-new set of instances, swaps when healthy | No | High (double instances briefly) | Easy (terminate new) |
| **Traffic splitting** | Canary — sends a % of traffic to new instances | No | High (extra instances) | Easy (shift traffic back) |

### Detailed breakdown

#### 1. All at once
```
[v1][v1][v1]  →  (all updated together)  →  [v2][v2][v2]
                  ⚠️ brief outage
```
- **Fastest, simplest, cheapest**
- **All instances go down briefly** — total outage during deploy
- A failed deploy means **all** instances are broken
- ✅ Use for: dev/test, where short downtime is fine

#### 2. Rolling
```
[v1][v1][v1]  →  [v2][v1][v1]  →  [v2][v2][v1]  →  [v2][v2][v2]
                  (batch by batch — capacity reduced during deploy)
```
- Deploys in **batches** (you set batch size)
- **No full downtime**, but **reduced capacity** while batches roll
- During deploy, some users hit v1, some hit v2 (mixed versions live)
- A failed batch leaves you in a mixed state
- ✅ Use for: when you can tolerate temporarily lower capacity

#### 3. Rolling with additional batch
```
            +[v2]  (temp extra batch added first)
[v1][v1][v1] → [v2][v1][v1][v2] → ... → [v2][v2][v2]  → remove temp
                  (full capacity maintained throughout)
```
- Like rolling, but **adds a temporary extra batch first** so full capacity is maintained the whole time
- Slightly more cost (the temporary instances)
- Still has **mixed versions** live during deploy
- ✅ Use for: production where you can't afford reduced capacity

#### 4. Immutable
```
Existing:  [v1][v1][v1]   (untouched)
New ASG:   [v2][v2][v2]   (fresh instances, health-checked)
                │
   if healthy → swap traffic, terminate old
   if unhealthy → just terminate new, v1 never touched
```
- Launches a **completely new set of instances** in a new ASG
- Only swaps over once the new instances pass health checks
- **Safest rolling option** — the old instances are never modified, so rollback is trivial (just kill the new ones)
- **Most expensive during deploy** (double the instances briefly)
- ✅ Use for: production where safety matters most

#### 5. Traffic splitting (canary)
```
[v1][v1][v1]  +  [v2] new instances
                  │
   send 10% traffic to v2, watch health/metrics
                  │
   healthy → shift to 100% v2
   unhealthy → shift back to v1, terminate v2
```
- A **canary deployment** — sends a configurable **percentage** of live traffic to the new version
- Lets you validate the new version on real traffic before full rollout
- Easy, automatic rollback if the canary is unhealthy
- ✅ Use for: production where you want to test new code on a small slice of real users first

> 🎯 **Exam memory aid:**
> - Cheapest + has downtime → **All at once**
> - In-place batches, reduced capacity → **Rolling**
> - In-place batches, full capacity → **Rolling with additional batch**
> - New instances, safe, pricey → **Immutable**
> - Canary, % of traffic → **Traffic splitting**

### The key distinction: in-place vs fresh instances

- **All at once / Rolling / Rolling+batch** → update code on **existing** instances (in-place)
- **Immutable / Traffic splitting** → deploy to **new** instances, then swap

In-place is cheaper but riskier (a bad deploy corrupts running instances). Fresh-instance deploys are safer and roll back cleanly but cost more during the deploy.

---

## 8. Blue/Green Deployment

This is a separate concept from the deployment *policies* above, and a favorite interview topic. **Beanstalk does not have a built-in "blue/green" policy** — you achieve it using **two environments + a URL swap.**

### The idea

Run **two identical environments**:
- **Blue** = the current live production environment
- **Green** = a new environment running the new version

You test Green privately. When you're confident, you **swap the CNAME (URL)** so traffic instantly flows to Green. Blue stays intact as an instant rollback.

```
Step 1 — Blue is live:
   users ──► [Blue env: v1]  ◄── live URL points here
             [Green env]      (doesn't exist yet)

Step 2 — Deploy Green:
   users ──► [Blue env: v1]  ◄── still live
             [Green env: v2]  (new, tested privately on its own URL)

Step 3 — Swap CNAMEs:
   users ──► [Green env: v2] ◄── live URL now points here (instant)
             [Blue env: v1]   (kept as rollback)

Step 4 — Rollback (if needed):
   Just swap the CNAME back to Blue. Instant recovery.
```

### How Beanstalk does it: "Swap Environment URLs"

Beanstalk has a built-in **Swap Environment URLs** action. It swaps the CNAMEs of two environments, so the DNS name your users hit now resolves to the other environment. The swap is near-instant (DNS-level).

### Why blue/green is powerful

| Benefit | Why |
|---|---|
| **Zero downtime** | The new environment is fully running before the swap |
| **Instant rollback** | Swap the URL back — Blue is still there, untouched |
| **Full pre-prod testing** | Test Green on real infrastructure before going live |
| **No mixed versions** | Unlike rolling, users are 100% on one version at a time |

### Trade-offs

- **Cost** — you run two full environments during the cutover
- **Database migrations** — tricky, because Blue and Green may share a DB. Schema changes need to be backward-compatible so both versions work during the transition. (This is the hard part in real life.)

### Blue/Green vs Immutable vs Traffic Splitting

These get conflated — here's the clean distinction:

| | Blue/Green | Immutable | Traffic Splitting |
|---|---|---|---|
| Scope | Two **separate environments** | New instances **within one** environment | New instances **within one** environment |
| Switch | Swap CNAME/URL | Swap ASG inside the env | Shift % of traffic |
| Rollback | Swap URL back (instant) | Terminate new instances | Shift traffic back |
| Granularity | 100% cutover | 100% cutover | Gradual (canary %) |
| Best for | Major releases, safe cutover | Safe in-env deploy | Validating on real traffic |

> 💡 **One-liner for interviews:** *"Blue/green uses two environments and a DNS swap for instant cutover and rollback; immutable does the same swap but within a single environment using a fresh ASG; traffic splitting is a canary that gradually shifts a percentage of traffic."*

---

## 9. Choosing a Deployment Strategy

A practical decision guide:

| Situation | Choose |
|---|---|
| Dev/test, downtime OK, lowest cost | **All at once** |
| Want gradual in-place rollout, can lose some capacity | **Rolling** |
| Production, must keep full capacity, in-place OK | **Rolling with additional batch** |
| Production, safety first, easy rollback | **Immutable** |
| Want to test new code on a small % of real users | **Traffic splitting (canary)** |
| Major release, want instant cutover + instant rollback | **Blue/Green (swap URLs)** |

---

## 10. Environment Types — Web Server vs Worker

Beanstalk environments come in two **tiers**:

### Web Server tier
- Handles **HTTP(S) requests** from clients
- Sits behind a load balancer
- The standard tier for websites and APIs

### Worker tier
- Processes **background jobs** from an **SQS queue**
- No public-facing endpoint
- A daemon on each instance reads messages from SQS and passes them to your app
- ✅ Use for: long-running tasks, async processing, scheduled jobs

```
Web tier:     Users → LB → EC2 (handle requests)
Worker tier:  SQS queue → daemon → EC2 (process jobs in background)
```

> 💡 A common pattern: a **web tier** receives requests and drops jobs onto SQS; a **worker tier** picks them up and processes them. This decouples the front end from heavy background work.

---

## 11. Beanstalk and Other AWS Services

Beanstalk is an orchestrator, so it leans on services you already know:

| Service | Role in Beanstalk |
|---|---|
| **EC2** | The compute that runs your app |
| **Auto Scaling** | Scales the instance fleet |
| **ELB (ALB/NLB)** | Load balancing in HA environments |
| **CloudWatch** | Health metrics, alarms, scaling triggers (ties to your monitoring chapter) |
| **S3** | Stores app versions and logs |
| **RDS** | Database (best created *outside* the environment for prod) |
| **IAM** | Two roles: the **service role** (Beanstalk manages resources) and the **instance profile** (your app's permissions — recall the EC2 chapter) |
| **SNS** | Notifications on environment events |
| **Route 53** | DNS, used in the blue/green URL swap |

> 🎯 Note the IAM tie-in: just like a raw EC2 instance, Beanstalk instances use an **instance profile** to grant your app permissions — no hardcoded keys. Same pattern from the IAM and EC2 chapters.

---

## 12. Best Practices

1. **Use the HA preset for production** — single instance has no resilience
2. **Create RDS outside the environment** for production, so the DB survives environment changes
3. **Use immutable or blue/green** for production deploys — clean rollback
4. **Keep schema changes backward-compatible** during blue/green so both versions work
5. **Use `.ebextensions`** for repeatable, version-controlled config
6. **Use environment variables** for config/secrets (or pull from SSM Parameter Store / Secrets Manager)
7. **Set up CloudWatch alarms** and enable enhanced health reporting
8. **Use separate environments** for dev/staging/prod
9. **Use instance profiles** for app permissions — never hardcode credentials
10. **Enable managed platform updates** to stay patched
11. **Tag environments** for cost tracking
12. **Don't over-rely on Beanstalk** for complex container orchestration — that's ECS/EKS territory

---

## 13. Common Scenarios

### Scenario 1: Cheapest possible dev environment
✅ Single instance preset (or single instance on Spot). No load balancer.

### Scenario 2: Production web app that must survive an AZ failure
✅ High availability preset — ALB + multi-AZ Auto Scaling group.

### Scenario 3: Deploy with zero downtime and instant rollback
✅ Blue/green — deploy to a second environment, test, then swap URLs.

### Scenario 4: Want to test new code on 10% of real users first
✅ Traffic splitting deployment policy (canary).

### Scenario 5: Deploy must never reduce capacity, but in-place is acceptable
✅ Rolling with additional batch.

### Scenario 6: Safest production deploy, cost not a concern
✅ Immutable — fresh instances, swap only when healthy.

### Scenario 7: You deleted your Beanstalk environment and lost your database
✅ The DB was created *inside* the environment. Fix: always provision RDS separately for production and connect via env vars.

### Scenario 8: Background job processing without a public endpoint
✅ Worker environment tier reading from SQS.

### Scenario 9: Your app needs to read from S3
✅ Add the permission to the environment's instance profile — same as raw EC2.

### Scenario 10: A deploy broke production and you need to recover instantly
✅ If blue/green: swap the URL back to Blue. If immutable: traffic never moved to bad instances. If all-at-once: redeploy the previous version (slower).

---

## 14. Self-Check

- [ ] Where does Beanstalk sit on the control-vs-convenience spectrum?
- [ ] What's the difference between an application, an application version, and an environment?
- [ ] What does the single instance preset lack that the HA preset provides?
- [ ] What AWS resources does the HA preset create?
- [ ] Name the five deployment policies and the trade-off each makes.
- [ ] Which deployment policies update existing instances vs launch new ones?
- [ ] What's the difference between rolling and rolling with additional batch?
- [ ] What is blue/green deployment, and how does Beanstalk implement it?
- [ ] How is blue/green different from immutable and traffic splitting?
- [ ] Why should you create production RDS outside the Beanstalk environment?
- [ ] What's the difference between a web server tier and a worker tier?
- [ ] How does a Beanstalk instance get permissions to call other AWS services?

---

## 15. Interview Questions

### Conceptual (warm-up)

1. **What is Elastic Beanstalk, and what problem does it solve?**
2. **Where does Beanstalk fit between EC2 and serverless?**
3. **Is Beanstalk free? What do you pay for?**
4. **Explain the hierarchy: application, version, environment.**
5. **What resources does Beanstalk provision on your behalf?**
6. **What platforms/languages does Beanstalk support?**
7. **Do you lose control over the underlying AWS resources with Beanstalk?**
   *(No — they're standard resources you can inspect and modify.)*

### Presets and Configuration

8. **What is a configuration preset in Beanstalk?**
9. **What's the difference between the single instance and high availability presets?**
10. **What does the high availability preset set up, and why is it "highly available"?**
11. **When would you choose single instance over high availability?**
12. **What is `.ebextensions`, and what's it used for?**
13. **Why should you NOT create your production database inside the Beanstalk environment?**
   *(Deleting the environment deletes the DB — create RDS separately.)*

### Deployment Policies

14. **Name and explain the Beanstalk deployment policies.**
15. **Which deployment policy has downtime, and why might you still use it?**
   *(All at once — fastest/cheapest, fine for dev.)*
16. **What's the difference between rolling and rolling with additional batch?**
   *(The additional batch maintains full capacity during the deploy.)*
17. **What is an immutable deployment, and why is it considered safe?**
   *(New instances; old ones untouched; trivial rollback.)*
18. **What is traffic splitting, and when would you use it?**
   *(Canary — test new version on a % of real traffic.)*
19. **Which deployment policies deploy in-place vs to new instances?**
   *(In-place: all-at-once, rolling, rolling+batch. New instances: immutable, traffic splitting.)*
20. **A deployment fails midway through a rolling deploy. What state are you in?**
   *(Mixed versions live — some instances on old, some on new.)*
21. **Which policy gives the cleanest rollback, and why?**
   *(Immutable or blue/green — the old version is never modified.)*

### Blue/Green Deployment

22. **What is blue/green deployment?**
23. **Does Beanstalk have a built-in blue/green policy? How do you actually do it?**
   *(No built-in policy — use two environments + Swap Environment URLs.)*
24. **How does the URL swap work, and why is it nearly instant?**
   *(It swaps CNAMEs at the DNS level.)*
25. **What are the main benefits of blue/green?**
   *(Zero downtime, instant rollback, full pre-prod testing, no mixed versions.)*
26. **What's the hardest part of blue/green in practice?**
   *(Database migrations — schema must stay backward-compatible while both versions are live.)*
27. **How is blue/green different from an immutable deployment?**
   *(Blue/green = two environments + DNS swap; immutable = new ASG within one environment.)*
28. **How is blue/green different from traffic splitting?**
   *(Blue/green is a 100% cutover; traffic splitting is a gradual canary by percentage.)*
29. **What's the cost trade-off of blue/green?**
   *(You run two full environments during the cutover.)*

### Environment Tiers and Integration

30. **What's the difference between a web server tier and a worker tier?**
31. **How does a worker environment get its work?**
   *(Reads messages from an SQS queue via a daemon.)*
32. **How does a Beanstalk instance get permissions to access S3 or DynamoDB?**
   *(Via the instance profile — same as raw EC2, no hardcoded keys.)*
33. **What are the two IAM roles involved in a Beanstalk environment?**
   *(Service role for Beanstalk to manage resources; instance profile for the app.)*
34. **How does Beanstalk use CloudWatch?**
   *(Health metrics, alarms, and Auto Scaling triggers.)*

### Scenario-Based (Senior level)

35. **Design a deployment strategy for a production app that needs zero downtime and instant rollback.**
   *(Blue/green with backward-compatible DB changes, or immutable.)*
36. **You need to validate a risky new release on a small fraction of real users before full rollout. How?**
   *(Traffic splitting / canary.)*
37. **Your team accidentally deleted a Beanstalk environment and lost the production database. How do you prevent this in future?**
   *(Provision RDS outside the environment; connect via env vars.)*
38. **You're migrating a monolith to Beanstalk. How would you structure applications, versions, and environments across dev/staging/prod?**
39. **How would you handle secrets and configuration in a Beanstalk app?**
   *(Environment variables backed by SSM Parameter Store / Secrets Manager.)*
40. **When would you choose Beanstalk over ECS/EKS, and when would you not?**
   *(Beanstalk for simple web apps and quick PaaS; ECS/EKS for complex container orchestration and microservices at scale.)*
41. **How would you achieve a fully automated CI/CD pipeline deploying to Beanstalk?**
   *(CodePipeline/CodeBuild or GitHub Actions → build artifact → deploy with chosen policy; blue/green via environment swap.)*

### Tricky Conceptual

42. **Can you run multiple environments under one application? Why would you?**
   *(Yes — dev/staging/prod, each on a different version.)*
43. **During a rolling deploy, can two users hit different versions of your app at the same time?**
   *(Yes — mixed versions are live during the rollout.)*
44. **Is "blue/green" a deployment policy in Beanstalk?**
   *(No — it's a pattern implemented via two environments and a URL swap, not one of the five policies.)*
45. **If Beanstalk just orchestrates EC2/ELB/ASG, why use it instead of building those yourself?**
   *(Speed, less operational overhead, sensible defaults — while keeping full access to the underlying resources.)*

---

**Next up:** `07-ecs-eks.md` — containers on AWS (ECS, Fargate, EKS), the natural step up from Beanstalk when you outgrow PaaS and need real container orchestration. Ties directly to your Kubernetes background.
