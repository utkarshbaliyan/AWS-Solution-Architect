# AWS Solutions Architect Associate — Prerequisites

> Foundational concepts you should be comfortable with **before** diving into AWS services. These notes cover the "why" behind cloud computing — once these click, services like EC2, VPC, S3, and IAM become much easier to reason about.

---

## Table of Contents

1. [Client-Server Architecture](#1-client-server-architecture)
2. [How the Internet Works](#2-how-the-internet-works)
3. [IP Addresses, DNS, and Domains](#3-ip-addresses-dns-and-domains)
4. [The OSI Model (Simplified)](#4-the-osi-model-simplified)
5. [Core Networking Protocols](#5-core-networking-protocols)
6. [Ports and Sockets](#6-ports-and-sockets)
7. [Cloud Computing Fundamentals](#7-cloud-computing-fundamentals)
8. [Virtualization](#8-virtualization)
9. [Storage Concepts](#9-storage-concepts)
10. [Security Basics](#10-security-basics)
11. [What's Next — IAM](#11-whats-next--iam)

---

## 1. Client-Server Architecture

The internet — and 99% of what AWS hosts — runs on this model.

**Client** → the consumer of a service. Your browser, your phone app, a Python script calling an API. It *requests* something.

**Server** → the provider of a service. A program running somewhere, listening for requests and sending back responses. It *responds*.

```
   ┌──────────┐    1. Request (HTTP GET /home)    ┌──────────┐
   │  CLIENT  │ ────────────────────────────────► │  SERVER  │
   │ (browser)│                                   │ (nginx)  │
   │          │ ◄──────────────────────────────── │          │
   └──────────┘    2. Response (HTML page)        └──────────┘
```

### Key properties

- **Servers are passive** — they wait. They don't talk first.
- **Clients initiate** — every conversation starts with a client request.
- **Stateless by default** — HTTP doesn't "remember" you between requests. That's why we have cookies, sessions, and tokens.
- **One server → many clients** — a single web server might serve thousands of clients simultaneously.

### Variations you'll encounter in AWS

| Pattern | Example |
|---|---|
| **Client → Server** | Your browser → EC2 instance running a web app |
| **Client → Load Balancer → Servers** | User → ALB → multiple EC2 instances |
| **Server → Server** | EC2 app → RDS database (the app is a client to the DB) |
| **Peer-to-peer** | Less common in AWS, but exists (e.g., gossip protocols in clusters) |

> 💡 **Mental model:** When you launch an EC2 instance running a web server, you've just created a "server" in the client-server sense. When that same EC2 calls an S3 bucket, it becomes a "client" to S3.

---

## 2. How the Internet Works

The internet is just a giant network of networks. When you type `google.com` in your browser, here's roughly what happens:

```
┌──────┐   ┌────────────┐   ┌──────────┐   ┌─────────────┐   ┌────────┐
│ You  │──►│ Your Wi-Fi │──►│ Your ISP │──►│ Backbone    │──►│ Google │
│      │   │ Router     │   │ (Jio/Air)│   │ (undersea   │   │ Server │
│      │◄──│            │◄──│          │◄──│ cables etc.)│◄──│        │
└──────┘   └────────────┘   └──────────┘   └─────────────┘   └────────┘
```

### The journey of a request

1. **You type a URL** — `https://example.com`
2. **DNS lookup** — your machine asks: "what's the IP for `example.com`?" → gets back something like `93.184.216.34`
3. **TCP handshake** — your computer establishes a reliable connection to that IP
4. **TLS handshake** — if HTTPS, encryption keys are exchanged
5. **HTTP request** — your browser sends `GET /` over the connection
6. **Server responds** — with HTML, status code, headers, etc.
7. **Browser renders** — parses HTML, fetches CSS/JS/images (more requests), shows you the page

### Why this matters for AWS

Every AWS service is reached over this same internet (or AWS's private backbone). When you set up a VPC, security groups, or Route 53, you're configuring **which part of this pipeline you control** and **who's allowed to talk to whom**.

---

## 3. IP Addresses, DNS, and Domains

### IP Addresses

An IP address is a unique identifier for a device on a network. Two versions matter:

- **IPv4** — `192.168.1.1` — four numbers (0–255), separated by dots. ~4.3 billion total. We've run out.
- **IPv6** — `2001:0db8:85a3::8a2e:0370:7334` — way more addresses. The future.

#### Public vs Private IPs

| Type | Range (IPv4) | Where it's used |
|---|---|---|
| **Public** | Everything else | Reachable on the open internet (e.g., your EC2's public IP) |
| **Private** | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Inside a private network — your home Wi-Fi, AWS VPC |

> 💡 In AWS, your VPC uses **private IPs** internally. Resources get a **public IP** only when they need to be reachable from the internet.

#### CIDR notation

`10.0.0.0/16` means "the first 16 bits are the network, the rest are hosts." This `/16` block has 65,536 IPs. You'll see CIDR everywhere in AWS networking.

| CIDR | Number of IPs |
|---|---|
| `/32` | 1 (single host) |
| `/24` | 256 |
| `/16` | 65,536 |
| `/8`  | 16,777,216 |

### DNS — Domain Name System

DNS is the internet's phonebook. It translates human-readable names → IP addresses.

```
example.com  ──DNS lookup──►  93.184.216.34
```

#### DNS record types you must know

| Record | What it does |
|---|---|
| **A** | Maps a name to an IPv4 address |
| **AAAA** | Maps a name to an IPv6 address |
| **CNAME** | Alias — points one name to another name |
| **MX** | Mail server for the domain |
| **TXT** | Arbitrary text — used for domain verification, SPF, etc. |
| **NS** | Specifies which name servers are authoritative for the domain |

AWS's DNS service is **Route 53**. Remember this name — it shows up everywhere.

---

## 4. The OSI Model (Simplified)

A 7-layer model that describes how data moves across a network. You don't need to memorize every layer for SAA, but knowing the big ones helps when debugging or designing.

| Layer | Name | What lives here | Example |
|---|---|---|---|
| 7 | Application | The actual app protocols | HTTP, HTTPS, FTP, DNS, SSH |
| 6 | Presentation | Encoding, encryption | TLS/SSL |
| 5 | Session | Session management | — |
| 4 | Transport | Reliable delivery | **TCP, UDP** |
| 3 | Network | Routing between networks | **IP**, ICMP |
| 2 | Data Link | Local network frames | Ethernet, Wi-Fi |
| 1 | Physical | The wire/radio itself | Cables, fiber, radio waves |

> 🎯 **For AWS, focus on Layers 3, 4, and 7.** Most networking decisions (security groups, load balancers, VPC routing) live in these layers.

- **Layer 4 Load Balancer (NLB)** → routes based on IP + port. Fast, dumb.
- **Layer 7 Load Balancer (ALB)** → routes based on HTTP content (URLs, headers). Smart, slightly slower.

---

## 5. Core Networking Protocols

### TCP (Transmission Control Protocol)

- **Connection-oriented** — establishes a connection before sending data (3-way handshake)
- **Reliable** — guarantees delivery, in order, no duplicates
- **Slower** but accurate
- **Used by:** HTTP, HTTPS, SSH, FTP, SMTP

```
Client                Server
  │   ──── SYN ────►   │
  │   ◄── SYN/ACK ──   │
  │   ──── ACK ────►   │
  │ (connection open)  │
```

### UDP (User Datagram Protocol)

- **Connectionless** — just sends packets, no handshake
- **Unreliable** — no guarantees, packets can be lost or arrive out of order
- **Fast, low overhead**
- **Used by:** DNS queries, video streaming, gaming, VoIP

### HTTP / HTTPS

- **HTTP** — HyperText Transfer Protocol. Plain text, runs on TCP port 80.
- **HTTPS** — HTTP over TLS. Encrypted, runs on TCP port 443.
- **Stateless** — each request is independent.

#### HTTP methods

| Method | Purpose |
|---|---|
| GET | Retrieve data |
| POST | Create data |
| PUT | Replace data |
| PATCH | Partially update data |
| DELETE | Remove data |

#### Status codes (common ones)

| Code | Meaning |
|---|---|
| 200 | OK |
| 301 / 302 | Redirect |
| 400 | Bad Request |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (authenticated, no permission) — **comes up constantly in IAM** |
| 404 | Not Found |
| 500 | Internal Server Error |
| 502 / 503 / 504 | Upstream issues — often seen with load balancers |

### SSH (Secure Shell)

- Encrypted remote shell access. Runs on TCP port 22.
- You'll use this to connect to EC2 Linux instances via `.pem` key files.

### Other protocols to know

| Protocol | Port | Use |
|---|---|---|
| FTP | 21 | File transfer (insecure) |
| SFTP | 22 | Secure file transfer (over SSH) |
| SMTP | 25, 587 | Sending email |
| DNS | 53 | Name resolution |
| RDP | 3389 | Remote Desktop for Windows EC2 |
| MySQL | 3306 | MySQL DB connections |
| PostgreSQL | 5432 | Postgres DB connections |

---

## 6. Ports and Sockets

- A **port** is a number (0–65535) that identifies a specific service on a machine.
- A **socket** is an IP + port combination — e.g., `192.168.1.10:443`.

Multiple services can run on one machine because each listens on a different port. Your EC2 might have:
- nginx on port 80
- a Node.js app on port 3000
- SSH on port 22

### Port ranges

| Range | Name | Use |
|---|---|---|
| 0–1023 | Well-known | Standard services (HTTP=80, SSH=22, etc.) |
| 1024–49151 | Registered | Apps and frameworks (3000, 8080, 5432, etc.) |
| 49152–65535 | Ephemeral | Temporary, client-side connections |

> 💡 **Security Group rules in AWS** are basically "allow/deny traffic on these ports from these IPs." Understanding ports is non-negotiable.

---

## 7. Cloud Computing Fundamentals

### What is "the cloud"?

Renting computing resources from someone else's data center, on demand, paying only for what you use.

### The three service models

| Model | You manage | Provider manages | AWS example |
|---|---|---|---|
| **IaaS** (Infrastructure) | OS, runtime, app, data | Hardware, networking, virtualization | EC2, EBS, VPC |
| **PaaS** (Platform) | App, data | Everything below | Elastic Beanstalk, RDS |
| **SaaS** (Software) | Nothing — just use it | Everything | Gmail, Dropbox, AWS WorkMail |

### Deployment models

- **Public cloud** — AWS, Azure, GCP. Shared infrastructure.
- **Private cloud** — your own data center, but cloud-style.
- **Hybrid** — mix of both. Common in enterprises.

### The 6 advantages of cloud (AWS loves these — they're on the exam)

1. **Trade capital expense for variable expense** — no upfront server purchases
2. **Benefit from massive economies of scale** — AWS buys hardware at a scale you can't
3. **Stop guessing capacity** — scale up or down as needed
4. **Increase speed and agility** — provision resources in minutes
5. **Stop spending money on running data centers** — focus on your app, not racks
6. **Go global in minutes** — deploy to multiple regions instantly

### AWS global infrastructure terms

- **Region** — a geographic area (e.g., `us-east-1`, `ap-south-1` in Mumbai)
- **Availability Zone (AZ)** — one or more isolated data centers within a region
- **Edge Location** — smaller sites for content caching (CloudFront)

> 💡 **Multi-AZ = high availability. Multi-Region = disaster recovery.** This distinction matters for the exam.

---

## 8. Virtualization

The core trick that makes cloud computing possible.

A **hypervisor** runs on physical hardware and lets you run multiple **virtual machines (VMs)** on the same physical server, each isolated from the others.

```
┌─────────────────────────────────────────────────┐
│  VM 1 (Linux)  │  VM 2 (Windows)  │  VM 3 (...) │
├─────────────────────────────────────────────────┤
│              Hypervisor (e.g., KVM, Xen)        │
├─────────────────────────────────────────────────┤
│              Physical Hardware                  │
└─────────────────────────────────────────────────┘
```

EC2 instances are virtual machines. AWS's hypervisor (Nitro, these days) divides massive physical servers into the VMs you provision.

### Containers vs VMs

Since you're working with Docker — quick refresher:

| | VM | Container |
|---|---|---|
| Isolation | Full OS | Process-level |
| Boot time | Minutes | Seconds |
| Size | GBs | MBs |
| AWS service | EC2 | ECS, EKS, Fargate |

Containers share the host OS kernel; VMs each run their own kernel. This is why containers are lighter and faster.

---

## 9. Storage Concepts

Three fundamental storage types — AWS has a service for each.

### Block storage

- Raw disk-like storage. The OS manages a filesystem on top.
- Low-latency, high-performance.
- **AWS service:** EBS (Elastic Block Store) — attached to EC2 like a hard drive.

### File storage

- A shared filesystem accessible over a network. Multiple machines can mount it.
- **AWS service:** EFS (Elastic File System) for Linux, FSx for Windows.

### Object storage

- Stores files as objects with metadata, accessed via API (HTTP-based).
- Infinitely scalable, but slower for random access.
- **AWS service:** S3 (Simple Storage Service).

### Quick comparison

| Type | Access | Use case |
|---|---|---|
| Block (EBS) | Mounted to one EC2 | Boot volumes, databases |
| File (EFS) | Mounted to many EC2 | Shared application files |
| Object (S3) | HTTP API | Backups, static assets, data lakes |

---

## 10. Security Basics

### CIA Triad

The three goals of information security:

- **Confidentiality** — only authorized people can see data (encryption, access control)
- **Integrity** — data hasn't been tampered with (hashing, signatures)
- **Availability** — the service is up when needed (redundancy, DDoS protection)

### Authentication vs Authorization

These two get confused constantly — they're different things.

- **Authentication** — *Who are you?* (login, password, MFA)
- **Authorization** — *What are you allowed to do?* (permissions, policies)

> 💡 IAM in AWS handles **both**. A user logging in is authentication. The policy attached to that user is authorization.

### Encryption

- **In transit** — data is encrypted while moving across the network (TLS/HTTPS)
- **At rest** — data is encrypted while sitting on a disk (S3 server-side encryption, EBS encryption)

AWS uses **KMS (Key Management Service)** to manage encryption keys.

### Symmetric vs Asymmetric encryption

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared key | Public + Private key pair |
| Speed | Fast | Slow |
| Use case | Encrypting data (AES) | Identity / signing (SSH, TLS handshake) |

### Common security concepts you'll meet

- **Firewall** — filters traffic based on rules. In AWS: Security Groups, Network ACLs.
- **VPN** — encrypted tunnel between two networks. AWS: Site-to-Site VPN, Client VPN.
- **DDoS** — Distributed Denial of Service attack. AWS: Shield, WAF.
- **Principle of Least Privilege** — give every user/service the *minimum* permissions needed. This is the foundation of IAM.

---

## 11. What's Next — IAM

With these fundamentals in place, the first AWS service we'll dive into is **IAM (Identity and Access Management)**.

IAM is where you'll see almost every concept from this document show up at once:

- **Authentication and authorization** — IAM does both
- **Principle of least privilege** — IAM's whole purpose
- **Users, groups, roles, policies** — the building blocks
- **JSON policy documents** — declarative, like everything modern
- **HTTPS API calls** — every AWS action goes through an IAM check

> 🎯 **Why IAM first?** Because every other AWS service depends on it. You can't launch an EC2, create an S3 bucket, or call a Lambda without IAM somewhere in the picture.

---

## Quick Self-Check

Before moving on to IAM, make sure you can answer these without looking:

- [ ] What's the difference between a public and private IP?
- [ ] What does CIDR `/24` mean?
- [ ] What's the difference between TCP and UDP, and which does HTTP use?
- [ ] What port does HTTPS use?
- [ ] What's the difference between authentication and authorization?
- [ ] What's the difference between a Region and an Availability Zone?
- [ ] What's the difference between block, file, and object storage?
- [ ] What does "stateless" mean in the context of HTTP?

If any of these feel shaky, re-read that section before moving on.

---

**Next up:** `01-iam.md` — IAM users, groups, roles, policies, and the principle of least privilege in practice.
