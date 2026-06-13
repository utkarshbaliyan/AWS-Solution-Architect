# AWS CloudWatch & X-Ray — Monitoring and Observability

> If EC2 is the workhorse and IAM the gatekeeper, CloudWatch is the **nervous system** of AWS — it sees what's happening everywhere. X-Ray adds the ability to trace a single request as it travels through your whole architecture. Together they answer the three core observability questions: *Is it healthy? What broke? Where exactly did it break?*

---

## Table of Contents

1. [Observability — The Big Picture](#1-observability--the-big-picture)
2. [CloudWatch Metrics](#2-cloudwatch-metrics)
3. [CloudWatch Alarms](#3-cloudwatch-alarms)
4. [CloudWatch Logs](#4-cloudwatch-logs)
5. [Log Groups and Log Streams](#5-log-groups-and-log-streams)
6. [The CloudWatch Agent](#6-the-cloudwatch-agent)
7. [CloudWatch Events vs EventBridge](#7-cloudwatch-events-vs-eventbridge)
8. [CloudWatch Dashboards](#8-cloudwatch-dashboards)
9. [AWS X-Ray](#9-aws-x-ray)
10. [How It All Fits Together](#10-how-it-all-fits-together)
11. [Real-World Usage by Service](#11-real-world-usage-by-service)
12. [Best Practices](#12-best-practices)
13. [Common Scenarios](#13-common-scenarios)
14. [Self-Check](#14-self-check)
15. [Interview Questions](#15-interview-questions)

---

## 1. Observability — The Big Picture

Observability rests on three pillars. Knowing which AWS tool covers which pillar makes everything else click.

| Pillar | Question it answers | AWS tool |
|---|---|---|
| **Metrics** | Is the system healthy? (numbers over time) | CloudWatch Metrics |
| **Logs** | What exactly happened? (text records) | CloudWatch Logs |
| **Traces** | Where did the request slow down or fail? (request journey) | AWS X-Ray |

```
┌─────────────────────────────────────────────────────────┐
│                      Your Application                      │
└───────────┬───────────────┬───────────────┬──────────────┘
            │               │               │
        Metrics           Logs           Traces
            │               │               │
            ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  CloudWatch  │ │  CloudWatch  │ │   AWS X-Ray  │
    │   Metrics    │ │     Logs     │ │              │
    └──────┬───────┘ └──────────────┘ └──────────────┘
           │
           ▼
     ┌──────────┐         ┌──────────────┐
     │  Alarms  │ ──────► │ EventBridge  │ ──► Actions
     └──────────┘         └──────────────┘
```

> 💡 **Mental model:** Metrics tell you *something is wrong*. Logs tell you *what* is wrong. Traces tell you *where* it's wrong. You need all three.

---

## 2. CloudWatch Metrics

A **metric** is a time-ordered set of data points — a number measured over time. CPU utilization, request count, queue depth, error rate: all metrics.

### Key concepts

| Term | Meaning |
|---|---|
| **Namespace** | A container for metrics (e.g., `AWS/EC2`, `AWS/S3`, or your custom `MyApp/Orders`) |
| **Metric** | The thing being measured (e.g., `CPUUtilization`) |
| **Dimension** | A name/value pair that identifies the metric (e.g., `InstanceId=i-1234`) |
| **Timestamp** | When the data point was recorded |
| **Statistic** | Aggregation over a period (Average, Sum, Min, Max, p99, etc.) |
| **Period** | The granularity of aggregation (e.g., 1 min, 5 min) |

### Standard vs Custom metrics

- **Standard metrics** — published automatically by AWS services (EC2 CPU, S3 bucket size, RDS connections, etc.)
- **Custom metrics** — you push your own with `PutMetricData` (e.g., "active users", "items in cart", memory usage)

### Resolution

| Type | Granularity |
|---|---|
| **Standard resolution** | 1-minute granularity |
| **High resolution** | down to 1-second granularity (custom metrics) |

### The critical EC2 gotcha (from the EC2 chapter)

CloudWatch does **NOT** capture memory usage or disk space by default for EC2 — those are *inside* the OS, and AWS only sees the hypervisor level. CPU, network, and disk I/O are free; **memory and disk usage require the CloudWatch Agent.**

> 🎯 **Exam favorite.** "Why can't I see memory in CloudWatch?" → because it requires the CloudWatch Agent.

### Metric Math and Anomaly Detection

- **Metric Math** — combine metrics with formulas (e.g., error rate = errors ÷ total requests)
- **Anomaly Detection** — ML learns a metric's normal pattern and flags deviations, used as a smarter alarm threshold

---

## 3. CloudWatch Alarms

An **alarm** watches a single metric (or a metric-math expression) and triggers an action when it crosses a threshold.

### Alarm states

| State | Meaning |
|---|---|
| **OK** | Metric is within the threshold |
| **ALARM** | Threshold breached |
| **INSUFFICIENT_DATA** | Not enough data to decide (e.g., instance just started) |

### Anatomy of an alarm

```
Metric:     CPUUtilization
Condition:  > 80%
Period:     5 minutes
Datapoints: 3 out of 3 (must breach 3 consecutive periods)
Action:     Send SNS notification + trigger Auto Scaling
```

### Alarm actions (what an alarm can DO)

| Action | Example |
|---|---|
| **SNS notification** | Email/SMS/push the on-call engineer |
| **Auto Scaling** | Add/remove EC2 instances |
| **EC2 action** | Stop, terminate, reboot, or recover an instance |
| **EventBridge** | Route to any downstream automation |
| **Systems Manager** | Create an OpsItem / incident |

### Composite Alarms

Combine multiple alarms with AND/OR logic to reduce noise.
*Example:* Only page someone if `HighCPU` **AND** `HighLatency` are both in ALARM — avoids false alerts from a single noisy metric.

> 💡 **EC2 auto-recovery:** An alarm on the `StatusCheckFailed_System` metric with a `recover` action automatically migrates a failed instance to healthy hardware while keeping the same IP, EBS, etc. A favorite real-world and exam pattern.

---

## 4. CloudWatch Logs

CloudWatch Logs centralizes log data from applications, OS, and AWS services into one searchable place.

### What sends logs to CloudWatch Logs?

- **Lambda** — automatically (every function writes to a log group)
- **EC2 / on-prem servers** — via the CloudWatch Agent
- **ECS / EKS containers** — via the `awslogs` / Fluent Bit drivers
- **API Gateway, Route 53, VPC Flow Logs, CloudTrail** — directly configured
- **RDS** — can export DB logs

### What you can do with logs

| Feature | Purpose |
|---|---|
| **Logs Insights** | Query logs with a purpose-built query language |
| **Metric Filters** | Turn log patterns into CloudWatch metrics (e.g., count "ERROR" lines → metric → alarm) |
| **Subscription Filters** | Stream logs in real time to Lambda, Kinesis, or OpenSearch |
| **Export to S3** | Archive logs cheaply for long-term retention |
| **Retention policies** | Auto-delete logs after N days (default: never expire — costs money!) |

### Logs Insights example query

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

### Metric Filter — a powerful pattern

You can't alarm on a log directly. The trick:

```
Log group → Metric Filter (count lines matching "ERROR")
          → CloudWatch Metric → Alarm → SNS → on-call
```

> 🎯 **Common exam/interview question:** "How do you get alerted when your app logs an error?" → Metric filter on the log group → metric → alarm.

> ⚠️ **Cost gotcha:** Logs never expire by default. Always set a retention policy or you'll pay to store logs forever.

---

## 5. Log Groups and Log Streams

These two are constantly confused. Get the hierarchy straight.

```
Log Group:  /aws/lambda/my-order-function   (the application/service)
   │
   ├── Log Stream: 2026/06/13/[$LATEST]abc123   (one instance/execution source)
   ├── Log Stream: 2026/06/13/[$LATEST]def456
   └── Log Stream: 2026/06/13/[$LATEST]ghi789
```

### Log Group

- A **container** for log streams that share the same retention, monitoring, and access settings.
- Usually maps to **one application or service** (e.g., one Lambda function, one app on EC2).
- Where you set **retention**, **metric filters**, and **permissions**.

### Log Stream

- A **sequence of log events from a single source** — one EC2 instance, one container, one Lambda execution environment.
- Many streams live inside one group.

### Analogy

> A **log group** is a folder; **log streams** are the individual files inside it. The folder is "my web app"; each file is "logs from one specific server."

| | Log Group | Log Stream |
|---|---|---|
| Represents | An application/service | A single source/instance |
| Settings live here | Retention, filters, access | — (inherits from group) |
| Cardinality | Few | Many per group |

---

## 6. The CloudWatch Agent

A piece of software you install on EC2 instances (or on-prem servers) to collect what CloudWatch can't see on its own.

### What it collects

- **Memory usage** (not available by default!)
- **Disk space usage** (not available by default!)
- **Swap usage**, detailed CPU/disk/network stats
- **Application and system logs** → pushed to CloudWatch Logs

### CloudWatch Agent vs default monitoring

| | Default (no agent) | With CloudWatch Agent |
|---|---|---|
| CPU | ✅ | ✅ |
| Network I/O | ✅ | ✅ |
| Disk I/O | ✅ | ✅ |
| **Memory** | ❌ | ✅ |
| **Disk space used** | ❌ | ✅ |
| **Custom app logs** | ❌ | ✅ |

### How it's deployed

1. Install the agent (via SSM, user data, or AMI bake-in)
2. Give the EC2 an **IAM role** with `CloudWatchAgentServerPolicy`
3. Provide a config file (what metrics/logs to collect)
4. Agent pushes data to CloudWatch on a schedule

> 💡 The older **CloudWatch Logs Agent** is deprecated — the **unified CloudWatch Agent** handles both metrics and logs. Use the unified one.

---

## 7. CloudWatch Events vs EventBridge

**EventBridge is the evolution of CloudWatch Events** — same underlying service, expanded features. AWS now recommends EventBridge for all new work, but you'll see both names.

### The core idea

A **serverless event bus**: AWS services and your apps emit *events*; you write *rules* that match events and route them to *targets*.

```
Event source ──► Event Bus ──► Rule (pattern match) ──► Target(s)
(EC2 state       (default      ("EC2 changed to        (Lambda, SNS,
 change, S3       or custom)    stopped")               SQS, Step Functions…)
 upload, cron)
```

### Two ways rules fire

| Type | Trigger | Example |
|---|---|---|
| **Event pattern** | An event matches a pattern | "When any EC2 instance enters `stopped` state" |
| **Schedule (cron/rate)** | Time-based | "Every day at 2 AM" — run a cleanup Lambda |

### What EventBridge adds over CloudWatch Events

- **Custom event buses** and **partner event sources** (Datadog, Zendesk, Shopify, etc.)
- **Schema Registry** — discover and codify event structures
- **Event archive & replay** — store events and re-run them later
- **More targets** and richer filtering

### Targets you can route events to

Lambda, SNS, SQS, Step Functions, Kinesis, ECS tasks, CodePipeline, Systems Manager, another event bus, and many more.

> 🎯 **Exam mental model:** EventBridge = "*when X happens anywhere, do Y automatically*." It's the glue for event-driven, serverless automation.

### Events vs Alarms — don't confuse them

| | CloudWatch Alarm | EventBridge |
|---|---|---|
| Watches | A **metric** crossing a threshold | An **event** occurring (state change, API call, schedule) |
| Example | "CPU > 80% for 5 min" | "An S3 object was uploaded" |
| Best for | Metric-driven reactions | Event-driven automation |

---

## 8. CloudWatch Dashboards

Customizable visual pages that pull metrics (and logs) from across services and regions into one view.

- **Cross-region and cross-account** views possible
- Widgets: line/stacked graphs, numbers, gauges, text, log tables, alarm status
- Useful for NOC screens, team health overviews, executive summaries

> 💡 Dashboards are about *human consumption* — they don't trigger actions. Alarms and EventBridge do the automation.

---

## 9. AWS X-Ray

X-Ray is **distributed tracing** — it follows a single request as it travels through every component of your application and shows you exactly where time is spent and where errors occur.

### The problem X-Ray solves

In a microservices/serverless world, one user request might touch: API Gateway → Lambda → DynamoDB → another Lambda → SQS → an EC2 service. If it's slow, *which hop* is the culprit? Metrics and logs alone won't easily tell you. **X-Ray traces the whole journey.**

### Core concepts

| Term | Meaning |
|---|---|
| **Trace** | The complete end-to-end journey of one request |
| **Segment** | The work done by one service/component (e.g., one Lambda) |
| **Subsegment** | Finer detail within a segment (e.g., a single DynamoDB call) |
| **Service Map** | A visual graph of all components and how requests flow between them |
| **Annotations** | Indexed key-value data for filtering traces (e.g., `userId`) |
| **Metadata** | Non-indexed extra data attached to a trace |
| **Sampling** | Only trace a percentage of requests to control cost/overhead |

### The Service Map

X-Ray's signature feature — an auto-generated diagram showing your architecture with latency and error rates on each node and edge:

```
[Client] ──► [API Gateway] ──► [Lambda] ──► [DynamoDB]
                                   │
                                   └──► [SNS]
   (each node shows avg latency, request count, error %)
```

A red/orange node instantly shows you where failures or slowness concentrate.

### How X-Ray gets data

- **Instrument your code** with the X-Ray SDK (adds tracing to your app)
- **The X-Ray daemon** collects trace data and sends it to the service (on EC2/ECS)
- **Lambda, API Gateway, App Mesh** have built-in integration (just enable it)

### What X-Ray reveals

- **Latency bottlenecks** — which service is slow
- **Errors and faults** — where requests fail (4xx vs 5xx)
- **Dependencies** — what calls what
- **Cold starts** (for Lambda)

> 🎯 **X-Ray vs CloudWatch:** CloudWatch aggregates (averages, totals across many requests). X-Ray follows *individual requests*. Use CloudWatch to know "latency went up"; use X-Ray to know "it's the DynamoDB call in the checkout Lambda."

---

## 10. How It All Fits Together

A complete observability flow for a typical app:

```
                          ┌──────────────────────────────┐
                          │        Your Application        │
                          └───┬──────────┬─────────────┬──┘
                              │          │             │
                   Metrics ───┘     Logs─┘      Traces─┘
                              │          │             │
                              ▼          ▼             ▼
                     ┌─────────────┐ ┌────────┐ ┌──────────┐
                     │   Metrics   │ │  Logs  │ │  X-Ray   │
                     └──────┬──────┘ └───┬────┘ └──────────┘
                            │            │
                    ┌───────▼──────┐     │ (metric filter)
                    │    Alarm     │◄────┘
                    └───────┬──────┘
                            │ (ALARM state)
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
         ┌────────┐  ┌─────────────┐ ┌──────────┐
         │  SNS   │  │ Auto Scaling│ │EventBridge│
         │ (page  │  │ (add EC2)   │ │ (automate)│
         │ oncall)│  └─────────────┘ └─────┬────┘
         └────────┘                        ▼
                                     ┌──────────┐
                                     │  Lambda  │
                                     │ (remediate)│
                                     └──────────┘
```

**Real example — auto-remediation:** App error rate spikes → metric filter on logs creates a metric → alarm fires → SNS pages on-call AND EventBridge triggers a Lambda that rolls back the deployment. No human needed for the first response.

---

## 11. Real-World Usage by Service

This is the part interviewers and the exam care about most — *how observability connects to each service.*

### EC2

- **Metrics:** CPU, network, disk I/O (free); memory + disk space (needs Agent)
- **Logs:** App/system logs via CloudWatch Agent
- **Alarms:** Auto-recovery on system status check failure; trigger Auto Scaling on CPU
- **Real use:** Alarm on CPU > 70% → ASG adds instances; alarm on `StatusCheckFailed_System` → auto-recover instance

### Lambda

- **Metrics (automatic):** Invocations, Duration, Errors, Throttles, ConcurrentExecutions
- **Logs (automatic):** Every function writes to `/aws/lambda/<function-name>`
- **X-Ray:** One toggle to trace function execution and downstream calls (great for spotting cold starts and slow DynamoDB calls)
- **Real use:** Alarm on `Errors` or `Throttles`; Logs Insights to debug; X-Ray to find which downstream call is slow

### RDS

- **Metrics:** CPU, connections, free storage, read/write IOPS, replica lag
- **Enhanced Monitoring:** OS-level metrics at up to 1-second granularity
- **Performance Insights:** Visualizes DB load by query/wait
- **Real use:** Alarm on `FreeStorageSpace` low or `DatabaseConnections` high → page DBA before outage

### S3

- **Metrics:** Bucket size, object count (daily, free); request metrics (per-request, paid)
- **Events:** S3 Event Notifications → EventBridge/Lambda/SQS on object upload
- **Real use:** Object uploaded → EventBridge → Lambda processes the file (classic event-driven pattern)

### ECS / EKS (containers)

- **Metrics:** Container Insights — CPU, memory, network per task/pod/service
- **Logs:** `awslogs` driver / Fluent Bit → CloudWatch Logs
- **X-Ray:** Trace requests across microservices
- **Real use:** Container Insights dashboard for cluster health; alarm on task memory; X-Ray service map across microservices

### API Gateway

- **Metrics:** Count, Latency, 4XXError, 5XXError, IntegrationLatency
- **Logs:** Access logs + execution logs to CloudWatch Logs
- **X-Ray:** End-to-end tracing from the API entry point
- **Real use:** Alarm on 5XX spike; X-Ray traces a slow API call through to the backend

### VPC (Networking)

- **VPC Flow Logs** → CloudWatch Logs: every accepted/rejected connection
- **Real use:** Logs Insights query on flow logs to investigate "who's talking to this instance?" or detect suspicious traffic

### DynamoDB

- **Metrics:** ConsumedRead/WriteCapacity, ThrottledRequests, latency
- **Real use:** Alarm on throttling → trigger capacity increase or alert; X-Ray shows DynamoDB call latency within a trace

### CloudTrail (governance, pairs with CloudWatch)

- CloudTrail logs *who did what* (API calls); ship those logs to CloudWatch Logs
- **Real use:** Metric filter on CloudTrail logs for "root login" or "security group changed" → alarm → security team paged

> 🎯 **Pattern to remember:** *Metrics for health, Logs for forensics, Events for automation, X-Ray for tracing, Alarms for reaction.* Almost every service plugs into all five.

---

## 12. Best Practices

1. **Set log retention** on every log group — defaults to "never expire" and quietly costs money
2. **Install the CloudWatch Agent** wherever you need memory/disk metrics
3. **Use IAM roles** (not keys) to grant the Agent/X-Ray daemon permissions
4. **Use metric filters** to turn important log patterns into alarms
5. **Use composite alarms** to cut alert noise and avoid pager fatigue
6. **Use EventBridge** for event-driven automation, not polling
7. **Enable X-Ray** on serverless/microservice apps from day one — retrofitting is painful
8. **Use sampling in X-Ray** to control cost on high-traffic apps
9. **Build dashboards** per team/service for at-a-glance health
10. **Centralize logs** cross-account into a dedicated logging/security account
11. **Alarm on what matters** (customer-facing symptoms) more than raw infra metrics
12. **Export old logs to S3** for cheap long-term retention/compliance

---

## 13. Common Scenarios

### Scenario 1: You can't see EC2 memory usage
✅ Install the CloudWatch Agent — memory and disk space aren't reported by default.

### Scenario 2: Alert me when my app logs an "ERROR"
✅ Create a metric filter on the log group matching "ERROR" → metric → alarm → SNS.

### Scenario 3: Run a cleanup job every night at 2 AM
✅ EventBridge scheduled rule (cron) → Lambda.

### Scenario 4: Process every file uploaded to S3
✅ S3 event notification → EventBridge/Lambda. Event-driven, no polling.

### Scenario 5: A microservices request is slow but you don't know which service
✅ Enable X-Ray; the service map shows the slow hop instantly.

### Scenario 6: Auto-scale EC2 when load is high
✅ CloudWatch alarm on CPUUtilization → Auto Scaling policy.

### Scenario 7: Automatically recover a failed instance
✅ Alarm on `StatusCheckFailed_System` with EC2 `recover` action.

### Scenario 8: Detect when someone changes a security group
✅ CloudTrail → CloudWatch Logs → metric filter for the API call → alarm → security team.

### Scenario 9: Investigate suspicious network traffic to an instance
✅ Enable VPC Flow Logs → CloudWatch Logs → query with Logs Insights.

### Scenario 10: Stream logs to a third-party tool in real time
✅ Subscription filter on the log group → Kinesis/Lambda → external system.

---

## 14. Self-Check

- [ ] What are the three pillars of observability and which AWS tool covers each?
- [ ] What's the difference between a metric, a dimension, and a namespace?
- [ ] Why can't CloudWatch see EC2 memory by default, and how do you fix it?
- [ ] What are the three alarm states?
- [ ] What's the difference between a log group and a log stream?
- [ ] How do you trigger an alarm based on a pattern in your logs?
- [ ] What's the difference between a CloudWatch Alarm and EventBridge?
- [ ] What's the relationship between CloudWatch Events and EventBridge?
- [ ] What does the X-Ray service map show you?
- [ ] What's the difference between a trace, a segment, and a subsegment?
- [ ] When would you use X-Ray instead of CloudWatch metrics?
- [ ] Why must you set a log retention policy?

---

## 15. Interview Questions

### Conceptual (warm-up)

1. **What is CloudWatch, and what is it used for?**
2. **What are the three pillars of observability, and which AWS services address them?**
3. **What is a CloudWatch metric? Explain namespace, dimension, and statistic.**
4. **What's the difference between standard and custom metrics?**
5. **Why doesn't CloudWatch show EC2 memory usage by default, and how do you get it?**
6. **What are the three states of a CloudWatch alarm?**
7. **What actions can a CloudWatch alarm trigger?**
8. **What is the difference between a log group and a log stream?**
9. **What is the CloudWatch Agent, and what does it add over default monitoring?**
10. **What is the default log retention in CloudWatch Logs, and why does that matter?**

### Logs and Metrics

11. **How would you get alerted when your application writes an error to its logs?**
   *(Metric filter → metric → alarm → SNS.)*
12. **What is CloudWatch Logs Insights?**
13. **What is a metric filter, and how is it different from a subscription filter?**
14. **How do you stream CloudWatch Logs to a third-party system in real time?**
   *(Subscription filter → Kinesis/Lambda.)*
15. **What is Metric Math, and when would you use it?**
16. **What's the difference between standard and high-resolution metrics?**
17. **How would you reduce CloudWatch Logs storage costs?**
   *(Retention policies + export to S3 for archives.)*

### Alarms and Automation

18. **What's the difference between a CloudWatch Alarm and EventBridge?**
19. **What is a composite alarm, and what problem does it solve?**
20. **How does EC2 auto-recovery work?**
   *(Alarm on StatusCheckFailed_System with a recover action.)*
21. **How would you auto-scale an EC2 fleet based on load?**
   *(Alarm on CPU → Auto Scaling policy.)*

### EventBridge

22. **What is EventBridge, and how does it relate to CloudWatch Events?**
23. **What are the two ways an EventBridge rule can be triggered?**
   *(Event pattern match, or schedule/cron.)*
24. **What targets can EventBridge route events to?**
25. **How would you run a scheduled task every night without managing a server?**
   *(EventBridge scheduled rule → Lambda.)*
26. **You want to process every object uploaded to an S3 bucket. How?**
   *(S3 event notification → EventBridge/Lambda.)*
27. **What does EventBridge add over the old CloudWatch Events?**
   *(Custom/partner buses, schema registry, archive & replay.)*

### X-Ray

28. **What is AWS X-Ray, and what problem does it solve?**
29. **Explain the difference between a trace, a segment, and a subsegment.**
30. **What is the X-Ray service map, and why is it useful?**
31. **When would you use X-Ray instead of CloudWatch metrics or logs?**
   *(To pinpoint where in a multi-service request the latency/error occurs.)*
32. **How does X-Ray get trace data from your application?**
   *(SDK instrumentation + X-Ray daemon; built-in for Lambda/API Gateway.)*
33. **What are annotations vs metadata in X-Ray?**
   *(Annotations are indexed and filterable; metadata is not.)*
34. **What is sampling in X-Ray, and why does it matter?**
   *(Trace a subset of requests to control cost/overhead.)*

### Real-World / Scenario (Senior level)

35. **Design an observability strategy for a serverless app (API Gateway + Lambda + DynamoDB).**
   *(Auto metrics + logs per service, X-Ray for tracing, alarms on errors/throttles, dashboard.)*
36. **A microservices request is slow. Walk me through how you'd diagnose it.**
   *(CloudWatch shows latency rose → X-Ray service map finds the slow hop → logs of that service for detail.)*
37. **How would you build automated remediation for a recurring production issue?**
   *(Alarm/metric filter → EventBridge → Lambda runbook; SNS to notify.)*
38. **How would you detect and alert on a root user login or a security group change?**
   *(CloudTrail → CloudWatch Logs → metric filter → alarm → SNS.)*
39. **How would you centralize logs and metrics across 20 AWS accounts?**
   *(Cross-account observability, dedicated logging account, cross-account dashboards, subscription filters.)*
40. **Your Lambda is intermittently slow. How do you find out why?**
   *(X-Ray to spot cold starts / slow downstream calls; CloudWatch Duration & Init metrics; Logs Insights.)*
41. **Differentiate CloudWatch, CloudTrail, and AWS Config — when do you use each?**
   *(CloudWatch = performance/health; CloudTrail = who did what API call; Config = resource configuration history/compliance.)*
42. **How would you avoid alert fatigue while still catching real incidents?**
   *(Composite alarms, alarm on customer-facing symptoms, anomaly detection, sensible thresholds.)*
43. **An EC2 instance is healthy in EC2 but the app is down. How does observability help you find the cause?**
   *(App logs via Agent, custom metrics, X-Ray traces, status checks all narrow it down.)*

### Tricky Conceptual

44. **Can you set a CloudWatch alarm directly on a log? Why or why not?**
   *(No — you alarm on a metric. Convert the log pattern to a metric via a metric filter first.)*
45. **What's the difference between an EventBridge rule and a CloudWatch alarm action?**
46. **Is CloudWatch regional or global?**
   *(Regional — metrics live in the region they're created; dashboards can aggregate cross-region.)*
47. **Why might enhanced monitoring on RDS show different CPU than CloudWatch standard metrics?**
   *(Enhanced monitoring reads OS-level data at higher resolution; standard reads from the hypervisor.)*

---

**Next up:** `06-sns-sqs.md` — SNS, SQS, and decoupled/event-driven architectures, which pair naturally with the EventBridge and alarm-notification patterns from this chapter.
