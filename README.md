# Project 1: Scalable Web Application with ALB and Auto Scaling

> **Architecture:** EC2-Based | **Cloud:** AWS | **Pattern:** Multi-AZ, High Availability

---

## Table of Contents

- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Architecture Deep Dive](#architecture-deep-dive)
  - [Network Layer — VPC](#1-network-layer--vpc)
  - [Edge Layer — CloudFront & WAF](#2-edge-layer--cloudfront--waf)
  - [Compute Layer — EC2 & ASG](#3-compute-layer--ec2--asg)
  - [Database Layer — RDS Multi-AZ](#4-database-layer--rds-multi-az)
  - [Security](#5-security)
  - [Observability](#6-observability)
- [AWS Services Used](#aws-services-used)
- [Key Design Decisions](#key-design-decisions)
- [Learning Outcomes](#learning-outcomes)
- [Prerequisites](#prerequisites)
- [Deployment Guide](#deployment-guide)
- [Cost Estimate](#cost-estimate)

---

## Overview

This project deploys a **production-grade, highly available web application** on AWS using EC2 instances inside a properly architected VPC. The architecture spans **two Availability Zones (AZ-1 and AZ-2)** to eliminate single points of failure, with all compute resources isolated in private subnets for maximum security.

**Key capabilities:**
- ⚡ **Auto Scaling** — scales EC2 instances automatically based on CPU load (target tracking at 60%)
- 🌍 **Global CDN** — CloudFront caches static assets at edge locations to reduce latency
- 🛡️ **WAF Protection** — OWASP Top 10 rules block common web attacks before they reach compute
- 🗄️ **Database HA** — RDS Multi-AZ with synchronous replication and automated failover
- 🔒 **Zero-trust Access** — Session Manager replaces SSH/Bastion for instance access (no open ports)
- 📊 **Full Observability** — CloudWatch dashboards, alarms, and SNS notifications

---

## Architecture Diagram

![Architecture Diagram](diagrams/architecture-diagram.png)

> *Project 1: Scalable Web Application with ALB and Auto Scaling — EC2-Based Architecture*

---

## Architecture Deep Dive

### 1. Network Layer — VPC

The VPC is the foundation of the architecture, organized into **three tiers** across two AZs:

```
VPC (e.g. 10.0.0.0/16)
│
├── AZ-1
│   ├── Public Subnet   (10.0.1.0/24)  → ALB, Bastion Host, NAT GW
│   ├── Private Subnet  (10.0.2.0/24)  → EC2 / ASG (App Tier)
│   └── Private Subnet  (10.0.3.0/24)  → RDS Primary (Data Tier)
│
└── AZ-2
    ├── Public Subnet   (10.0.4.0/24)  → ALB, NAT GW
    ├── Private Subnet  (10.0.5.0/24)  → EC2 / ASG (App Tier)
    └── Private Subnet  (10.0.6.0/24)  → RDS Standby (Data Tier)
```

**Route Tables:**

| Subnet | Destination | Target |
|--------|-------------|--------|
| Public | `0.0.0.0/0` | Internet Gateway |
| Private (App) | `0.0.0.0/0` | NAT Gateway (same AZ) |
| Private (DB) | local only | — |

> ⚠️ Each AZ has its own **NAT Gateway** — if AZ-1 NAT fails, AZ-2 EC2 instances continue to reach the internet via AZ-2 NAT.

---

### 2. Edge Layer — CloudFront & WAF

**Traffic flow:**

```
Users → Route 53 → CloudFront (+ WAF) → Internet Gateway → ALB → EC2
                         ↓
                    S3 (static assets: images, CSS, JS)
```

- **Route 53** — Alias record pointing to ALB with health checks. Automatic DNS failover if ALB becomes unhealthy.
- **CloudFront** — Caches static content at AWS edge locations globally. Reduces latency and offloads traffic from EC2.
- **WAF** — Attached to CloudFront. Enforces OWASP Top 10 rules (SQL injection, XSS, rate limiting) before requests reach the VPC.
- **ALB** — Layer 7 load balancer distributing traffic across EC2 instances in both AZs. Performs health checks every 30 seconds.

---

### 3. Compute Layer — EC2 & ASG

**Web Tier (Public Subnet):**
- EC2 instances serve as the web-facing layer (Nginx/Apache)
- Managed by an **Auto Scaling Group** with a Launch Template
- Scaling policy: **Target Tracking at 60% CPU** (min: 2, desired: 2, max: 6)

**App Tier (Private Subnet):**
- EC2 instances handle application logic (Node.js / Python / Java)
- No direct internet exposure — traffic only from ALB
- **Session Manager** enables secure CLI access without SSH keys or open ports

**Auto Scaling Config:**
```
Min capacity  : 2  (always on across both AZs)
Desired       : 2
Max capacity  : 6
Scale-out     : CPU > 60% for 2 consecutive periods
Scale-in      : CPU < 40% for 10 consecutive periods
```

---

### 4. Database Layer — RDS Multi-AZ

```
App EC2 (Port 3306/5432)
     ↓
RDS Primary (AZ-1)  ──sync replication──►  RDS Standby (AZ-2)
                                                   ↑
                                        Auto failover (~60–120 sec)
```

- **Engine:** MySQL or PostgreSQL
- **Multi-AZ:** Synchronous replication — zero data loss on failover
- **Automated Backups:** Daily snapshots + transaction logs (point-in-time restore)
- **No public access** — DB subnet has no route to the internet

---

### 5. Security

| Layer | Mechanism | Details |
|-------|-----------|---------|
| Edge | WAF | OWASP Top 10, rate limiting |
| Network | Security Groups | Stateful rules — only allow necessary ports |
| Network | NACLs | Stateless subnet-level rules |
| Network | Private Subnets | EC2 and RDS not reachable from internet |
| Access | Session Manager | No SSH, no Bastion, no open port 22 |
| Audit | CloudTrail | Every API call logged to S3 |
| Data | RDS Encryption | Encrypted at rest (AES-256) and in transit (TLS) |

**Security Group Rules (summary):**

```
ALB SG         → accepts 80/443 from 0.0.0.0/0
EC2 SG         → accepts traffic from ALB SG only
RDS SG         → accepts 3306/5432 from EC2 SG only
```

---

### 6. Observability

```
EC2 / ALB / RDS → CloudWatch Metrics & Logs
                         ↓
                  CloudWatch Alarms
                         ↓
                      SNS Topic → Email Notifications
```

**Recommended Alarms:**

| Alarm | Threshold | Action |
|-------|-----------|--------|
| CPU Utilization | > 80% | SNS Email |
| ALB 5xx Errors | > 10/min | SNS Email |
| RDS Storage | < 20% free | SNS Email |
| ASG Scale-out | triggered | SNS Email |

**CloudTrail** → logs all API calls → stores in S3 (Audit Logs bucket)

---

## AWS Services Used

| Service | Purpose |
|---------|---------|
| **VPC** | Network isolation, subnets, route tables, NACLs |
| **EC2** | Web and application compute |
| **Auto Scaling Group** | Automatic scaling based on demand |
| **ALB** | Layer 7 load balancing, health checks |
| **CloudFront** | CDN, static asset caching |
| **WAF** | OWASP Top 10 protection, rate limiting |
| **RDS Multi-AZ** | Managed relational database with HA |
| **Route 53** | DNS routing, health checks |
| **NAT Gateway** | Outbound internet for private subnets |
| **Internet Gateway** | Inbound/outbound internet for public subnets |
| **Systems Manager** | Session Manager — bastion-free EC2 access |
| **CloudWatch** | Metrics, logs, dashboards, alarms |
| **SNS** | Alarm notifications via email |
| **CloudTrail** | API audit logging |
| **S3** | Static assets (CloudFront origin) + CloudTrail logs |
| **IAM** | Roles and policies for EC2, SSM, RDS access |

---

## Key Design Decisions

### ✅ Why NAT Gateway per AZ (not one shared)?
A single NAT Gateway is a single point of failure. If its AZ goes down, all private EC2 instances across all AZs lose internet access. One NAT Gateway per AZ ensures fault isolation.

### ✅ Why Session Manager instead of Bastion Host?
| Bastion Host | Session Manager |
|---|---|
| Extra EC2 cost | No extra infrastructure |
| Port 22 must be open | Zero open ports |
| Key management overhead | IAM-based access |
| Audit requires extra setup | All sessions logged automatically |

### ✅ Why WAF on CloudFront (not ALB)?
Attaching WAF to CloudFront means malicious traffic is blocked at the **edge**, before it enters the AWS network — saving bandwidth and compute costs.

### ✅ Why Multi-AZ RDS over Read Replica?
Multi-AZ provides **automatic failover** (HA), while Read Replicas provide **read scalability**. For production HA, Multi-AZ is the correct choice.

---

## Learning Outcomes

By completing this project, you will be able to:

- ✅ Design VPCs with correct subnet, route table, and NAT Gateway configurations
- ✅ Build highly available architectures across multiple Availability Zones
- ✅ Configure ALB listener rules and target group health checks
- ✅ Implement Auto Scaling with target tracking and step scaling policies
- ✅ Secure applications with WAF, Security Groups, and private subnets
- ✅ Use Systems Manager Session Manager as a bastion-free access alternative
- ✅ Set up CloudWatch dashboards and alarms with SNS notifications
- ✅ Configure RDS Multi-AZ with automated backups and failover

---

## Prerequisites

- AWS Account with appropriate IAM permissions
- AWS CLI configured (`aws configure`)
- Basic knowledge of networking (CIDR, subnets, routing)
- Familiarity with EC2, VPC, and IAM concepts

---

## Deployment Guide

### Step 1 — VPC & Networking
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create subnets (repeat for each AZ)
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a

# Create and attach Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id <vpc-id> --internet-gateway-id <igw-id>

# Create NAT Gateways (one per AZ — requires Elastic IP)
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id <public-subnet-id> --allocation-id <eip-id>
```

### Step 2 — Security Groups
```bash
# ALB Security Group
aws ec2 create-security-group --group-name alb-sg --description "ALB SG" --vpc-id <vpc-id>
aws ec2 authorize-security-group-ingress --group-id <alb-sg-id> --protocol tcp --port 443 --cidr 0.0.0.0/0

# EC2 Security Group (allow from ALB only)
aws ec2 create-security-group --group-name ec2-sg --description "EC2 SG" --vpc-id <vpc-id>
aws ec2 authorize-security-group-ingress --group-id <ec2-sg-id> --protocol tcp --port 80 --source-group <alb-sg-id>
```

### Step 3 — Launch Template & ASG
```bash
# Create Launch Template
aws ec2 create-launch-template \
  --launch-template-name web-lt \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.medium",
    "IamInstanceProfile": {"Name": "SSMInstanceProfile"},
    "SecurityGroupIds": ["<ec2-sg-id>"]
  }'

# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt \
  --min-size 2 --desired-capacity 2 --max-size 6 \
  --vpc-zone-identifier "<private-subnet-az1>,<private-subnet-az2>"
```

### Step 4 — RDS Multi-AZ
```bash
aws rds create-db-instance \
  --db-instance-identifier prod-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --master-username admin \
  --master-user-password <password> \
  --multi-az \
  --db-subnet-group-name <db-subnet-group> \
  --no-publicly-accessible
```

### Step 5 — ALB
```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name web-alb \
  --subnets <public-subnet-az1> <public-subnet-az2> \
  --security-groups <alb-sg-id>

# Create Target Group
aws elbv2 create-target-group \
  --name web-tg \
  --protocol HTTP --port 80 \
  --vpc-id <vpc-id> \
  --health-check-path /health
```

### Step 6 — CloudFront + WAF
```bash
# Create WAF Web ACL (OWASP managed rules)
aws wafv2 create-web-acl \
  --name prod-waf \
  --scope CLOUDFRONT \
  --default-action Allow={} \
  --rules file://waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=prod-waf
```

---

## Cost Estimate

> Approximate monthly cost for a **dev/test** deployment in `us-east-1`:

| Service | Est. Monthly Cost |
|---------|------------------|
| EC2 (2x t3.medium) | ~$60 |
| NAT Gateway (2x) | ~$65 |
| ALB | ~$20 |
| RDS Multi-AZ (db.t3.medium) | ~$100 |
| CloudFront (10GB transfer) | ~$1 |
| WAF | ~$10 |
| CloudWatch | ~$10 |
| **Total** | **~$266/month** |

> 💡 Use **AWS Free Tier** where available and **stop/terminate resources** when not in use to minimize costs.

---

## License

This project is for educational purposes as part of the AWS Manara Program.
