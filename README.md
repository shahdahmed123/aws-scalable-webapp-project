
# Project 1: Highly Available Scalable Web Application on AWS (EC2, ALB & Auto Scaling)

> **Architecture:** EC2-Based • **Availability:** Multi-AZ • **Pattern:** Three-Tier Architecture

## Table of Contents

- Overview
- Architecture Diagram
- Traffic Flow
- Design Principles
- Architecture Components
- Load Balancing
- Security Architecture
- Security Group Configuration
- Network Ports
- Monitoring & Logging
- High Availability
- Deployment Steps
- AWS Services Used
- Repository Structure
- Future Improvements

---

# Overview

This project demonstrates a highly available and scalable web application deployed on AWS using a three-tier architecture across two Availability Zones. The design focuses on scalability, security, high availability, and operational best practices.

---

# Architecture

![Architecture Diagram](diagrams/architecture-diagram.png)

---

# Traffic Flow

```text
Users
   │
Route 53
   │
CloudFront
   │
AWS WAF
   │
Internet Gateway
   │
Application Load Balancer
   │
Target Group (Web-TG)
   │
Web Tier Auto Scaling Group
   │
Application Tier EC2
   │
Amazon RDS Multi-AZ
```

---

# Design Principles

- High Availability
- Scalability
- Security by Design
- Least Privilege
- Cost Optimization
- Operational Excellence

---

# Architecture Components

## Edge Layer
- Amazon Route 53
- Amazon CloudFront
- AWS WAF

## Network Layer
- Amazon VPC
- Public & Private Subnets
- Internet Gateway
- NAT Gateway

## Web Tier
- Public Application Load Balancer
- Web Target Group (Web-TG)
- Web EC2 Auto Scaling Group
- Bastion Host

## Application Tier
- Amazon EC2
- AWS Systems Manager Session Manager

## Database Tier
- Amazon RDS Multi-AZ
- Synchronous Replication

---

# Load Balancing

The public Application Load Balancer receives HTTPS requests from CloudFront and forwards traffic to the **Web Target Group (Web-TG)**.

The Target Group is associated with the **Web Tier Auto Scaling Group**, allowing EC2 instances to be automatically registered, deregistered, and health checked.

| Component | Configuration |
|-----------|---------------|
| Load Balancer | Application Load Balancer |
| Listener | HTTPS (443) |
| Target Group | Web-TG |
| Health Check | HTTP (80) |
| Targets | Web Tier EC2 Instances |

---

# Security Architecture

- AWS WAF protects against common web attacks.
- CloudFront provides edge caching and AWS Shield Standard protection.
- Application and Database tiers are isolated in private subnets.
- IAM Roles eliminate long-term credentials.
- CloudTrail records AWS API activity.
- CloudWatch and SNS provide monitoring and alerting.

---

# Security Group Configuration

| Security Group | Attached Resources | Inbound Rules | Outbound Rules |
|----------------|--------------------|---------------|----------------|
| **ALB-Web-SG** | Public ALB + Web Tier EC2 | HTTP (80), HTTPS (443) from Internet, SSH (22) from Bastion/My IP | HTTP (80) to App-SG |
| **App-SG** | Application EC2 | HTTP (80) from ALB-Web-SG | MySQL (3306) to RDS-SG, HTTPS (443) |
| **RDS-SG** | Amazon RDS | MySQL (3306) from App-SG only | Default |

Security Boundary:

```text
Internet
   │
ALB-Web-SG
   │
Web-TG
   │
Web Auto Scaling Group
   │
App-SG
   │
3306
   │
RDS-SG
```

---

# Network Ports

| Port | Purpose |
|------|---------|
| 443 | Client → CloudFront → ALB |
| 80 | ALB → Web Tier |
| 80 | Web Tier → Application Tier |
| 3306 | Application Tier → Amazon RDS |
| 22 | Bastion SSH |

---

# Monitoring & Logging

- Amazon CloudWatch
- Amazon SNS
- AWS CloudTrail
- Amazon S3 Audit Logs

---

# High Availability

- Multi-AZ deployment
- Web Tier Auto Scaling Group
- Application Load Balancer
- Amazon RDS Multi-AZ
- Automatic health checks

---

# Deployment Steps

1. Create VPC, subnets and route tables.
2. Configure Internet Gateway and NAT Gateway.
3. Create Security Groups.
4. Deploy Bastion Host.
5. Create Web Target Group.
6. Deploy Application Load Balancer.
7. Deploy Web Tier Auto Scaling Group.
8. Deploy Application EC2.
9. Deploy Amazon RDS Multi-AZ.
10. Configure Route 53, CloudFront and AWS WAF.
11. Configure CloudWatch, SNS and CloudTrail.

---

# AWS Services Used

- Amazon EC2
- EC2 Auto Scaling
- Application Load Balancer
- Amazon VPC
- Amazon Route 53
- Amazon CloudFront
- AWS WAF
- Amazon RDS
- AWS Systems Manager
- Amazon CloudWatch
- Amazon SNS
- AWS CloudTrail
- Amazon S3

---

# Repository Structure

```text
.
├── README.md
└── diagrams/
    └── architecture-diagram.png
```

---

# Future Improvements

- Terraform
- CI/CD Pipeline
- ECS / EKS
- AWS Secrets Manager
- AWS Backup
- VPC Endpoints

---

# License

Educational project created for AWS learning and portfolio development.
