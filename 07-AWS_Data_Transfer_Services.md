# AWS Data Transfer Services — Transfer Family, DataSync & Storage Gateway

> **Study notes by a DevOps engineer who has actually broken prod with these.** Covers AWS Transfer Family, AWS DataSync, and AWS Storage Gateway — when to reach for each, how they actually behave, and the exam traps that catch people. Written for the SAA-C03 exam and real-world architecture interviews.

---

## Table of Contents

1. [The Big Picture — Why These Three Exist](#1-the-big-picture)
2. [AWS Transfer Family](#2-aws-transfer-family)
3. [AWS DataSync](#3-aws-datasync)
4. [AWS Storage Gateway](#4-aws-storage-gateway)
5. [Head-to-Head Decision Matrix](#5-decision-matrix)
6. [Real-World Architecture Patterns](#6-real-world-patterns)
7. [Exam Callouts & Gotchas](#7-exam-callouts)
8. [Self-Check Questions](#8-self-check-questions)
9. [Certification Questions (SAA-C03 style)](#9-certification-questions)
10. [Interview Questions — Tiered](#10-interview-questions)

---

## 1. The Big Picture

The single most useful mental model: **classify by the question "who initiates, and over what protocol?"**

```
                        AWS DATA TRANSFER LANDSCAPE
   ┌──────────────────────────────────────────────────────────────┐
   │                                                                │
   │   EXTERNAL PARTNER pushes/pulls files over standard protocol   │
   │   (SFTP/FTPS/FTP/AS2)              ──►   TRANSFER FAMILY        │
   │                                                                │
   │   BULK one-time or scheduled MIGRATION of data                 │
   │   (on-prem ↔ AWS, or AWS ↔ AWS)   ──►   DATASYNC               │
   │                                                                │
   │   ONGOING hybrid access — on-prem apps need low-latency        │
   │   local access but data lives in AWS  ──►  STORAGE GATEWAY     │
   │                                                                │
   └──────────────────────────────────────────────────────────────┘
```

The trap candidates fall into: all three "move data," so the exam tests whether you pick by **access pattern**, not by "it moves bytes." Memorize the one-liners:

- **Transfer Family** = give partners an SFTP/FTPS/FTP/AS2 endpoint backed by S3/EFS. *Protocol gateway.*
- **DataSync** = fast, managed bulk copy with verification. *Migration & replication engine.*
- **Storage Gateway** = on-prem appliance presenting AWS storage as local NFS/SMB/iSCSI/VTL. *Hybrid bridge.*

---

## 2. AWS Transfer Family

### 2.1 What it actually is

A **fully managed service that exposes file-transfer protocol endpoints** (SFTP, FTPS, FTP, and AS2) sitting in front of **Amazon S3 or Amazon EFS**. Your partners keep using their existing SFTP clients; AWS keeps the protocol servers patched, scaled, and highly available. You stop running a fleet of EC2-based SFTP boxes.

> **War story:** Half the "data transfer" work in enterprises is some 15-year-old bank or logistics partner that *only* speaks SFTP and will never touch an API. Transfer Family exists precisely so you don't have to babysit an EC2 SFTP server with a cron'd `s3 sync` and a PEM key in someone's home directory.

### 2.2 Supported protocols

| Protocol | What it is | Typical use |
|----------|-----------|-------------|
| **SFTP** | SSH File Transfer Protocol (port 22) | Most common; secure by default |
| **FTPS** | FTP over TLS (port 21 + data range) | Legacy partners needing encryption |
| **FTP**  | Plain FTP, **unencrypted** | **VPC-internal only**, never public |
| **AS2**  | Applicability Statement 2 — signed/encrypted EDI message transport over HTTP | B2B EDI (retail, healthcare, supply chain) |

> **Exam callout:** Plain **FTP is only supported for access within a VPC** — AWS will not let you expose it to the internet. If a question says "expose FTP to external partners over the internet," that's a distractor.

### 2.3 Architecture

```
   PARTNER                         AWS TRANSFER FAMILY
 ┌─────────┐    SFTP/FTPS/AS2    ┌──────────────────────┐      ┌──────────┐
 │ SFTP    │ ──────────────────►│  Managed endpoint     │─────►│   S3     │
 │ client  │                    │  (public / VPC)       │      │  bucket  │
 └─────────┘                    │                       │  or  └──────────┘
                                │  ┌─────────────────┐  │      ┌──────────┐
   Auth via:                    │  │ Identity provider│  │─────►│   EFS    │
   - Service-managed users      │  │ (SM / custom /   │  │      └──────────┘
   - AWS Directory Service      │  │  Directory Svc)  │  │
   - Custom IdP (Lambda/API GW) │  └─────────────────┘  │
                                └──────────────────────┘
```

**Endpoint types:**
- **Public** — internet-facing, AWS-managed IPs (can change; don't hardcode).
- **VPC (internal)** — private IPs only, reachable inside the VPC/via VPN/DX.
- **VPC (internet-facing)** — private endpoint in your VPC but with Elastic IPs attached, so you control the static IPs partners whitelist.

> **Real-world tip:** Partners almost always demand a **static IP to whitelist in their firewall**. Use the **VPC endpoint with attached Elastic IPs** so the IP never changes. Public endpoints have AWS-managed IPs that you should not pin.

### 2.4 Identity / authentication options

1. **Service-managed** — store users and their SSH public keys in Transfer Family itself. Simple, fine for a handful of partners.
2. **AWS Directory Service** — authenticate against Microsoft AD.
3. **Custom identity provider** — a Lambda + API Gateway (or just Lambda) that you write, validating against your own user store (e.g., Secrets Manager, DynamoDB, an external IdP). This is the powerful option for SaaS multi-tenant setups.

Each user maps to an **IAM role** (the access) plus a **scope-down policy / logical home directory** that locks them to a prefix so partner A can't see partner B's files.

### 2.5 Key features worth knowing

- **Managed workflows** — trigger post-upload steps (copy, tag, decrypt with PGP, run a Lambda, delete) automatically when a file lands. Great for "decrypt → validate → move to processed/ prefix" pipelines.
- **Logical directories** — present a clean virtual directory tree to the user that maps to arbitrary S3 prefixes (hide your real bucket structure).
- **PGP decryption** built into workflows.
- Billing = **per-protocol-enabled-endpoint per hour + data uploaded/downloaded per GB.** The hourly charge runs whether or not anyone connects — this surprises people.

> **Cost gotcha:** You pay an hourly rate **per enabled protocol**. Enabling SFTP + FTPS + FTP on one endpoint = 3× the hourly charge. Only enable what you need.

### 2.6 When to use / when NOT to

✅ Use when: external parties transfer files over **standard protocols**, you want a managed endpoint into S3/EFS, you need B2B EDI (AS2).

❌ Don't use when: it's an internal bulk migration (→ DataSync), you control both ends and can use the AWS CLI/SDK/API directly (just use S3 API — cheaper), or you need ongoing low-latency local file access on-prem (→ Storage Gateway).

---

## 3. AWS DataSync

### 3.1 What it actually is

A **managed data-transfer service for moving large amounts of data fast**, with built-in **encryption, scheduling, and end-to-end integrity verification**. It handles the parallelization, retries, and checksums you'd otherwise script by hand. Think of it as `rsync` on steroids, fully managed, running at up to ~10 Gbps per agent.

> **War story:** The day before a data-center decommission, someone always discovers 40 TB of NFS shares nobody migrated. DataSync is what saves that weekend — point an agent at the NFS export, point it at S3/EFS/FSx, schedule it, and it verifies every file landed intact. No more hand-rolled `rsync` loops dying at 2 a.m.

### 3.2 Source ↔ destination support

DataSync moves between **on-prem storage and AWS**, and **AWS-to-AWS**.

```
   SOURCES                         DESTINATIONS
   ┌──────────────┐               ┌──────────────┐
   │ NFS share    │               │ Amazon S3    │ (all storage classes)
   │ SMB share    │   DataSync    │ Amazon EFS   │
   │ HDFS         │ ────────────► │ FSx (Windows,│
   │ Object store │   (verified,  │  Lustre,     │
   │  (S3-compat) │   encrypted,  │  ONTAP, OpenZFS)
   │ S3 / EFS /   │   scheduled)  │              │
   │  FSx (AWS)   │               └──────────────┘
   └──────────────┘
```

- **On-prem → AWS:** requires a **DataSync agent** (a VM you deploy on VMware/Hyper-V/KVM/EC2) that reads the source and ships data to AWS.
- **AWS → AWS** (e.g., S3 in region A → S3 in region B, or S3 → EFS): **no agent needed**, it's fully managed in-cloud.

### 3.3 The agent

For on-prem or other-cloud sources, you deploy a lightweight **agent VM** near the source. It connects to the DataSync service endpoint, reads the source, compresses + encrypts in transit (TLS), and transfers. One agent can drive up to ~10 Gbps; deploy multiple for more throughput.

### 3.4 Key features

- **Incremental transfers** — after the first run, only changed data moves (compares metadata/checksums).
- **Data integrity verification** — checksums every file end-to-end so you *know* the copy matches.
- **Scheduling** — hourly/daily/weekly; great for ongoing replication and DR.
- **Filters/includes-excludes** — copy only `*.csv`, skip `/tmp`, etc.
- **Bandwidth throttling** — cap throughput so you don't saturate the office link during business hours.
- **Preserves metadata** — POSIX permissions, ownership, timestamps (important for lift-and-shift).
- Pricing = **per-GB transferred** (flat data-processing fee). Simple and predictable.

### 3.5 DataSync vs Transfer Family — the classic confusion

| | **DataSync** | **Transfer Family** |
|---|---|---|
| Purpose | **Bulk migrate / replicate** data | **Expose protocol endpoint** for partners |
| Who initiates | **You** (scheduled/triggered jobs) | **External users/partners** (interactive) |
| Protocols | NFS/SMB/HDFS/object (internal) | SFTP/FTPS/FTP/AS2 (partner-facing) |
| Verification | Built-in checksums | No (it's a transfer endpoint) |
| Best for | DC migration, DR replication | Partner SFTP drop-box into S3 |

> **Exam callout:** "Migrate 200 TB of on-prem NFS to EFS, verified, on a schedule" → **DataSync**. "Let an external vendor upload daily files via SFTP to S3" → **Transfer Family**. Don't mix them.

### 3.6 DataSync vs Snowball

- **DataSync** = over the **network**. Good when you have bandwidth and time.
- **Snowball / Snow Family** = **physical appliance** shipped to you. Use when the data is too big for your link (e.g., petabytes, or a slow site where network transfer would take *months*).
- Common combo: **Snowball for the initial bulk seed, DataSync for the ongoing incremental/changed data.**

---

## 4. AWS Storage Gateway

### 4.1 What it actually is

A **hybrid cloud storage service** — a software appliance (VM on-prem, or hardware appliance) that gives your **on-prem applications local-feeling access to virtually unlimited AWS storage**, with **local caching** for low latency. The data lives in AWS (S3, Glacier, EBS snapshots, etc.); the gateway caches the hot subset locally.

> **War story:** A media company had editors on-site who needed fast access to files but a storage budget that couldn't hold the full archive on-prem. File Gateway: editors see a normal SMB share, recently-used files are served from the local cache at LAN speed, everything ultimately lives in S3 with lifecycle policies pushing cold assets to Glacier. The cache is the whole trick.

### 4.2 The three (well, four) gateway types

```
   STORAGE GATEWAY TYPES
   ┌────────────────────────────────────────────────────────────┐
   │                                                             │
   │  FILE GATEWAY    on-prem mounts NFS/SMB ──► files in S3     │
   │   ├─ S3 File Gateway   → Amazon S3                          │
   │   └─ FSx File Gateway  → Amazon FSx for Windows             │
   │                                                             │
   │  VOLUME GATEWAY  on-prem mounts iSCSI block ──► EBS snaps   │
   │   ├─ Cached volumes    (primary data in S3, cache local)   │
   │   └─ Stored volumes    (primary data local, async backup   │
   │                         to S3 as EBS snapshots)             │
   │                                                             │
   │  TAPE GATEWAY    on-prem backup app sees a Virtual Tape    │
   │   (VTL)          Library ──► tapes in S3 / Glacier         │
   │                                                             │
   └────────────────────────────────────────────────────────────┘
```

#### A. File Gateway (S3 File Gateway)
- On-prem servers mount an **NFS or SMB** share.
- Files become **objects in S3** (1 file = 1 object), with a local cache for hot data.
- Use for: extending file shares to the cloud, backups, tiering archives to S3/Glacier.

#### B. FSx File Gateway
- Like File Gateway but the backend is **Amazon FSx for Windows File Server** — low-latency local access to Windows-native shares (SMB, AD integration, NTFS ACLs).

#### C. Volume Gateway (block storage over iSCSI)
- **Cached volumes:** primary data stored in **S3**, frequently-accessed data cached locally. Local cache small, cloud is source of truth.
- **Stored volumes:** primary data stored **locally** (entire dataset on-prem), asynchronously backed up to S3 as **EBS snapshots**. Low-latency to all data + cloud DR.

> **Exam callout — the #1 Volume Gateway trap:**
> - **Cached = primary data in AWS** (cloud), cache on-prem. (Keyword: "limited on-prem storage," "minimize on-prem footprint.")
> - **Stored = primary data on-prem** (entire dataset local), backups to AWS. (Keyword: "low-latency access to *entire* dataset," "all data local.")

#### D. Tape Gateway (VTL)
- Presents a **Virtual Tape Library** to existing backup software (Veeam, Veritas NetBackup, Commvault, etc.) over iSCSI.
- Your backup app thinks it's writing to physical tapes; they're really stored in **S3, then archived to S3 Glacier / Glacier Deep Archive**.
- Use for: **replacing a physical tape backup infrastructure** without changing your backup software.

> **Exam callout:** "Eliminate physical tape libraries / move tape backups to cloud, keep existing backup software" → **Tape Gateway**, every time.

### 4.3 Caching is the core idea

Every gateway type keeps a **local cache** of the hot data so on-prem apps get **low-latency access**, while the **durable copy lives in AWS** (effectively unlimited capacity, durable, lifecycle-managed). That's the whole hybrid value proposition: *local speed, cloud scale.*

### 4.4 When to use / when NOT to

✅ Use when: on-prem apps need **ongoing, low-latency local access** but you want the durable/scalable copy in AWS; you want to retire tape; you need a cloud-backed file share or iSCSI volumes.

❌ Don't use when: it's a **one-time migration** (→ DataSync, faster and purpose-built); you only need a partner-facing protocol endpoint (→ Transfer Family); the workload is fully in-cloud (just use S3/EFS/FSx directly).

> **Exam callout:** Storage Gateway is **ongoing hybrid access**, not migration. If the question says "one-time migration of 50 TB," the answer is **DataSync** even though Storage Gateway *could* technically move it.

---

## 5. Decision Matrix

| Need | Service | Why |
|------|---------|-----|
| External partner uploads via **SFTP/FTPS** into S3 | **Transfer Family** | Managed protocol endpoint |
| **B2B EDI** message exchange | **Transfer Family (AS2)** | AS2 is the EDI transport |
| **One-time / scheduled bulk migration** of NFS/SMB to AWS, verified | **DataSync** | Fast, checksummed, incremental |
| **Ongoing DR replication** AWS↔AWS or on-prem↔AWS | **DataSync** | Scheduling + incremental |
| On-prem app needs **local NFS/SMB share** backed by S3 | **Storage Gateway (File)** | Local cache + S3 durability |
| On-prem app needs **iSCSI block volumes**, data lives in cloud | **Storage Gateway (Volume, Cached)** | Block + cloud primary |
| Keep **entire dataset on-prem**, back up to AWS | **Storage Gateway (Volume, Stored)** | Local primary, S3 backups |
| **Replace physical tape** backup | **Storage Gateway (Tape/VTL)** | VTL into S3/Glacier |
| Data too big for the network link (PB-scale, slow site) | **Snow Family** | Physical shipping |

**The 5-second exam heuristic:**
- "Partner" + "SFTP/FTPS/FTP/AS2" → **Transfer Family**
- "Migrate" / "replicate" / "verified bulk copy" → **DataSync**
- "Hybrid" / "local cache" / "on-prem app needs it locally" / "replace tape" → **Storage Gateway**

---

## 6. Real-World Patterns

### Pattern 1 — Partner data ingestion pipeline
```
Partner ──SFTP──► Transfer Family ──► S3 (raw/) 
                       │
                       └─ Managed Workflow: PGP decrypt → validate → 
                          move to processed/ → trigger Lambda → load to Redshift
```
A logistics partner drops encrypted EDI/CSV daily; the workflow decrypts, validates, files it, and kicks off analytics. Zero servers to manage.

### Pattern 2 — Data center exit
```
On-prem NFS/SMB (200 TB) ──DataSync agent──► S3 + EFS (verified, scheduled)
       │ (initial seed if link is slow)
       └──Snowball──► S3, then DataSync handles the delta
```

### Pattern 3 — Hybrid file share + archive tiering
```
On-prem editors ──SMB──► S3 File Gateway ──► S3 ──Lifecycle──► Glacier Deep Archive
                          (local cache = LAN-speed hot files)
```

### Pattern 4 — Tape modernization
```
Backup software (Veeam) ──iSCSI VTL──► Tape Gateway ──► S3 ──► Glacier Deep Archive
(no change to backup software; physical tape library retired)
```

---

## 7. Exam Callouts & Gotchas

- 🔴 **Plain FTP (Transfer Family) = VPC-internal only.** Never internet-facing.
- 🔴 **Volume Gateway: Cached = primary in cloud; Stored = primary on-prem.** Most-missed distinction in the whole topic.
- 🔴 **Storage Gateway is hybrid/ongoing, not migration.** One-time bulk move → DataSync.
- 🔴 **Tape Gateway** is the answer whenever "physical tape" + "keep existing backup software" appears.
- 🔴 **DataSync AWS→AWS needs no agent;** on-prem source needs an agent VM.
- 🔴 **Transfer Family billing is per-enabled-protocol-hour** + data — runs even with zero connections.
- 🔴 **Use VPC endpoint + Elastic IP** in Transfer Family when a partner needs a static whitelist IP.
- 🔴 **DataSync verifies integrity (checksums); Transfer Family does not.**
- 🔴 **Snowball vs DataSync** = physical vs network. Too big for the link → Snowball.
- 🔴 Transfer Family backs onto **S3 or EFS**; DataSync lands in **S3/EFS/FSx**; Storage Gateway backs onto **S3/Glacier/EBS snapshots**.

---

## 8. Self-Check Questions

<details>
<summary><strong>Q1.</strong> A partner can only send files via SFTP and needs them in an S3 bucket. Which service?</summary>

**AWS Transfer Family** (SFTP endpoint backed by S3). DataSync is for bulk migration you initiate; this is a partner-initiated protocol transfer.
</details>

<details>
<summary><strong>Q2.</strong> You must migrate 300 TB of on-prem NFS to EFS with integrity verification on a weekly schedule. Which service?</summary>

**AWS DataSync** — deploy an agent at the NFS source, schedule the task, and it provides built-in checksum verification and incremental transfers.
</details>

<details>
<summary><strong>Q3.</strong> An on-prem app needs low-latency access to its *entire* dataset locally, but you want durable backups in AWS. Cached or Stored Volume Gateway?</summary>

**Stored Volumes** — primary data stays on-prem (entire dataset local), asynchronously backed up to S3 as EBS snapshots. Cached would keep primary data in the cloud.
</details>

<details>
<summary><strong>Q4.</strong> The company wants to eliminate its physical tape library but keep using Veritas NetBackup. Which service/type?</summary>

**Storage Gateway — Tape Gateway (VTL).** It presents a virtual tape library to the existing backup software while storing tapes in S3/Glacier.
</details>

<details>
<summary><strong>Q5.</strong> Why is plain FTP a poor answer for an internet-facing partner endpoint?</summary>

FTP is **unencrypted**, and AWS Transfer Family **only supports FTP within a VPC**, not internet-facing. Use SFTP or FTPS for external partners.
</details>

<details>
<summary><strong>Q6.</strong> A partner insists on whitelisting a single static IP for your SFTP endpoint. How do you configure Transfer Family?</summary>

Use a **VPC endpoint (internet-facing) with attached Elastic IP(s)** so the IP is static and partner-controllable. Public endpoints use AWS-managed IPs that can change.
</details>

<details>
<summary><strong>Q7.</strong> You need to copy S3 data from us-east-1 to eu-west-1 on a schedule, verified. Do you need a DataSync agent?</summary>

**No.** AWS-to-AWS DataSync transfers are fully managed and **require no agent**. Agents are only for on-prem (or non-AWS) sources.
</details>

---

## 9. Certification Questions (SAA-C03 style)

**Q1.** A company runs on-prem applications that write to an iSCSI block volume. They want to minimize on-prem storage while keeping low-latency access to frequently used data, with the primary copy in AWS. Which solution?

- A. Storage Gateway — Stored Volumes
- B. Storage Gateway — Cached Volumes
- C. AWS DataSync to EBS
- D. Transfer Family with EFS

<details><summary>Answer</summary>

**B — Cached Volumes.** Primary data lives in S3 (minimizes on-prem footprint), with a local cache for low-latency access to hot data. Stored Volumes (A) keep the entire dataset on-prem.
</details>

---

**Q2.** A retail company exchanges EDI documents with suppliers and needs a managed, secure B2B transport that supports signing and encryption. Which AWS service/feature?

- A. DataSync
- B. Transfer Family with SFTP
- C. Transfer Family with AS2
- D. Storage Gateway Tape Gateway

<details><summary>Answer</summary>

**C — Transfer Family with AS2.** AS2 is the standard protocol for B2B EDI message exchange with signing/encryption. SFTP (B) transfers files but isn't the EDI transport.
</details>

---

**Q3.** A company must migrate 500 TB from an on-prem SMB share to Amazon FSx for Windows. The site has a 1 Gbps link. They want verified, incremental, scheduled transfers. Which service?

- A. Transfer Family
- B. Storage Gateway File Gateway
- C. AWS DataSync
- D. S3 Transfer Acceleration

<details><summary>Answer</summary>

**C — AWS DataSync.** It supports SMB→FSx, with built-in verification, incremental transfers, and scheduling. (If the link were too slow for the volume, you'd seed with Snowball first.)
</details>

---

**Q4.** An organization wants to retire its physical tape backup infrastructure but cannot change its existing backup application. What should they deploy?

- A. AWS Backup
- B. Storage Gateway — Tape Gateway
- C. DataSync to Glacier
- D. Storage Gateway — File Gateway

<details><summary>Answer</summary>

**B — Tape Gateway.** It presents a Virtual Tape Library to the existing backup software, storing tapes in S3 and archiving to Glacier — no change to the backup app.
</details>

---

**Q5.** External vendors must upload daily files using SFTP into an S3 bucket, each vendor isolated to its own prefix, with automatic PGP decryption on arrival. Which combination?

- A. EC2 SFTP server + cron + S3 sync
- B. Transfer Family (SFTP) with scope-down policies + managed workflows
- C. DataSync with filters
- D. Storage Gateway File Gateway with SMB

<details><summary>Answer</summary>

**B — Transfer Family with scope-down policies and managed workflows.** Scope-down policies/logical home dirs isolate each vendor to a prefix; a managed workflow performs PGP decryption automatically. (A) is the unmanaged anti-pattern this service replaces.
</details>

---

**Q6.** A media company needs on-prem editors to access an SMB file share at LAN speed, but the full archive is too large for on-prem storage and should live durably in S3 with Glacier tiering. Which solution?

- A. DataSync to S3
- B. Storage Gateway — S3 File Gateway
- C. Transfer Family with EFS
- D. FSx for Lustre

<details><summary>Answer</summary>

**B — S3 File Gateway.** Local SMB share with a cache for LAN-speed hot files; files stored as S3 objects with lifecycle policies tiering cold data to Glacier.
</details>

---

**Q7.** Which statement about DataSync vs Transfer Family is correct?

- A. Both expose SFTP endpoints to external users.
- B. DataSync is initiated by your scheduled tasks; Transfer Family is initiated by external users over standard protocols.
- C. Transfer Family verifies data integrity with checksums; DataSync does not.
- D. DataSync requires an agent for all transfers.

<details><summary>Answer</summary>

**B.** DataSync is for you-initiated bulk/scheduled migration with verification; Transfer Family is a partner-facing protocol endpoint. (C is reversed — DataSync verifies; D is false — AWS→AWS needs no agent.)
</details>

---

**Q8.** A company has a slow 50 Mbps link and must move 2 PB to S3 within two weeks. What's the best approach?

- A. DataSync over the existing link
- B. AWS Snowball (Snow Family), then DataSync for the delta
- C. Transfer Family with FTPS
- D. Storage Gateway Cached Volumes

<details><summary>Answer</summary>

**B.** 2 PB over 50 Mbps would take far longer than two weeks, so physically ship the bulk via **Snowball**, then use **DataSync** to sync any changed/new data afterward.
</details>

---

## 10. Interview Questions — Tiered

### 🟢 Junior / Foundational

1. **What are the three main AWS data-transfer/hybrid services and the one-line purpose of each?**
   *Transfer Family = managed SFTP/FTPS/FTP/AS2 endpoint into S3/EFS. DataSync = managed bulk/scheduled data migration & replication with verification. Storage Gateway = on-prem appliance giving local access to AWS-backed storage.*

2. **Which protocols does Transfer Family support, and which one is restricted to VPC-only?**
   *SFTP, FTPS, FTP, AS2. Plain **FTP** is VPC-internal only (unencrypted, never internet-facing).*

3. **Name the Storage Gateway types.**
   *File Gateway (S3 and FSx flavors), Volume Gateway (Cached / Stored), Tape Gateway (VTL).*

4. **Does DataSync always need an agent?**
   *No — only for on-prem/non-AWS sources. AWS-to-AWS transfers are agentless.*

### 🟡 Mid-Level / Practical

5. **A partner needs a static IP to whitelist for SFTP. How do you architect the Transfer Family endpoint?**
   *VPC endpoint (internet-facing) with attached Elastic IPs, so the IP is static. Avoid public endpoints (AWS-managed, changeable IPs).*

6. **Explain the difference between Cached and Stored Volume Gateway with a real use case for each.**
   *Cached: primary data in S3, local cache — use when on-prem storage is limited but you want cloud as source of truth. Stored: entire dataset on-prem, async EBS-snapshot backups to S3 — use when you need low-latency to all data plus cloud DR.*

7. **You're migrating 200 TB of NFS to EFS and the business wants proof every file arrived intact. Walk me through your approach.**
   *Deploy a DataSync agent near the NFS export, create source/destination locations, build a task with verification enabled and include/exclude filters, schedule it, throttle bandwidth during business hours, run an initial full transfer then incrementals. DataSync's end-to-end checksums give the integrity proof.*

8. **How would you build a secure partner file-ingestion pipeline that decrypts PGP files on arrival and isolates each partner?**
   *Transfer Family SFTP endpoint, per-user IAM roles + scope-down policies / logical home directories for isolation, custom IdP via Lambda if you have many partners, and a managed workflow to PGP-decrypt → validate → move to a processed prefix → trigger downstream Lambda.*

### 🔴 Senior / Architecture & Trade-offs

9. **When would you choose Snowball over DataSync, and how do they combine?**
   *Snowball when data volume vs. available bandwidth makes network transfer infeasible (PB-scale or very slow links). Combine: Snowball seeds the bulk, DataSync handles ongoing incremental deltas — minimizing cutover downtime during a DC migration.*

10. **Critique using an EC2-based SFTP server vs Transfer Family. What are the operational trade-offs?**
    *EC2: you own patching, HA, scaling, key rotation, monitoring, and the S3 sync glue — more control but high operational burden and risk. Transfer Family: managed availability/patching/scaling, native S3/EFS integration, workflows, IdP flexibility — less control over the host but far lower ops cost. Choose EC2 only for unusual protocol/custom-daemon needs.*

11. **Design a hybrid storage strategy for a company keeping on-prem editing workflows but wanting cloud durability and cost tiering. Which services and why?**
    *S3 File Gateway for the local SMB/NFS share with caching (LAN-speed hot files), S3 as the durable store, lifecycle policies tiering cold data to Glacier/Deep Archive. Optionally DataSync for a one-time bulk seed of the existing archive into S3 before cutover. Tape Gateway if they also want to retire physical tape backups.*

12. **A team proposes Storage Gateway for a one-time 50 TB migration. Why push back, and what do you recommend instead?**
    *Storage Gateway is built for ongoing hybrid access with caching, not bulk one-time migration — it's slower and operationally heavier for this. Recommend DataSync: purpose-built, parallelized, verified, incremental, and decommissionable after the move. Reserve Storage Gateway for genuine ongoing hybrid needs.*

13. **How does each service handle encryption in transit and at rest?**
    *In transit: Transfer Family uses SFTP/FTPS/AS2 (TLS/SSH); DataSync uses TLS; Storage Gateway uses TLS to AWS. At rest: all land in AWS storage (S3/EFS/FSx/Glacier/EBS) which supports SSE/KMS encryption. Transfer Family also supports PGP decryption of payloads via workflows.*

---

> **Final mental model to lock in before the exam:**
> **Transfer Family** = *door* (partners come in over a protocol).
> **DataSync** = *moving truck* (you haul bulk data, verified).
> **Storage Gateway** = *bridge* (on-prem keeps working, but storage lives in AWS).
