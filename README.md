[README.md](https://github.com/user-attachments/files/26388586/README.md)
# DevOps Project 01 — AWS 3-Tier Architecture

> Production-style infrastructure on AWS, designed around real enterprise patterns.
> Built to understand the *why* behind secure multi-tier networking, not just follow a tutorial.

![AWS](https://img.shields.io/badge/AWS-EC2_%7C_VPC_%7C_NLB_%7C_ASG_%7C_RDS-232F3E?style=flat-square&logo=amazonaws&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-Java-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![NGINX](https://img.shields.io/badge/NGINX-Reverse_Proxy-009639?style=flat-square&logo=nginx&logoColor=white)
![Maven](https://img.shields.io/badge/Maven-Build-C71A36?style=flat-square&logo=apachemaven&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-Quality_Gate-4E9BCD?style=flat-square&logo=sonarqube&logoColor=white)
![JFrog](https://img.shields.io/badge/JFrog-Artifactory-41BF47?style=flat-square&logo=jfrog&logoColor=white)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Security Design](#security-design)
4. [CI/CD Pipeline](#cicd-pipeline)
5. [Tech Stack](#tech-stack)
6. [Setup Guide](#setup-guide)
7. [Known Limitations](#known-limitations)
8. [Roadmap](#roadmap)
9. [Key Learnings](#key-learnings)

---

## Overview

This project implements a production-style three-tier architecture on AWS. The focus is on secure network design, horizontal scalability, and a complete CI/CD pipeline — the same patterns used in real enterprise environments.

**What this project covers:**

- Secure VPC with public and private subnet separation
- Network Load Balancer (NLB) as the sole internet-facing entry point
- Auto Scaling Groups for both the proxy (NGINX) and application (Spring Boot) tiers
- Backend database on RDS MySQL, accessible only from the application tier
- Integrated CI/CD pipeline: Bitbucket → Maven → SonarQube → JFrog → App Servers

---

## Architecture

```
<img width="415" height="582" alt="Screenshot 2026-03-22 at 11 48 55 PM" src="https://github.com/user-attachments/assets/efc61590-1b07-4d4e-98d2-ba65cddeffc3" />

```

> **Note:** Add an architecture screenshot here once available.
> Place the image at `architecture/architecture-diagram.png` and reference it as:
> `![Architecture](architecture/architecture-diagram.png)`

### Traffic flow

1. Users reach the **External NLB** — the only internet-facing component
2. NLB routes to **NGINX** instances in the private proxy tier
3. NGINX forwards requests to **Spring Boot** app servers in the application tier
4. App servers read/write to **RDS MySQL** in an isolated private subnet
5. Outbound traffic from private instances routes through the **NAT Instance**

---

## Security Design

**Core principle:** no application component is directly internet-accessible.

### Network isolation

| Component | Subnet | Public IP | Internet route |
|---|---|---|---|
| External NLB | Public | Yes | Direct via IGW |
| Bastion host | Public | Yes | Direct via IGW |
| NGINX instances | Private | No | Via NAT Instance only |
| App servers | Private | No | Via NAT Instance only |
| CI/CD tools | Private | No | Via NAT Instance only |
| RDS MySQL | Private | No | None |

### Security group rules

| Source | Destination | Port | Justification |
|---|---|---|---|
| Internet | External NLB | 80, 443 | Public HTTPS entry point only |
| NLB (nlb-sg) | NGINX (nginx-sg) | 80, 443 | NLB-to-proxy traffic only |
| NGINX (nginx-sg) | App servers (app-sg) | 8080 | Proxy-to-application tier only |
| App servers (app-sg) | RDS MySQL (rds-sg) | 3306 | DB access from app tier only |
| Bastion (bastion-sg) | All EC2 instances | 22 | SSH exclusively via Bastion |

**Key design decisions:**

- Security groups reference each other by SG ID — no broad CIDR ranges. Access control survives subnet address changes.
- RDS has zero inbound rules except from `app-sg`. There is no admin shortcut — DB changes go through the app tier or an SSH tunnel via Bastion.
- Private subnet route tables contain a route to the NAT Instance, not to the IGW. Instances can initiate outbound connections (for updates, JFrog pulls) but cannot receive inbound ones.

---

## CI/CD Pipeline

```
Bitbucket  →  Maven  →  SonarQube  →  JFrog  →  App Servers
  (SCM)       (build)   (quality)     (store)     (deploy)
```

### Steps

| Step | Tool | Action |
|---|---|---|
| 1 | Bitbucket | Developer pushes code; build process is triggered |
| 2 | Maven | Source compiled; WAR artifact packaged |
| 3 | SonarQube | Static analysis run; pipeline blocked on critical findings |
| 4 | JFrog Artifactory | Versioned WAR stored; single source of truth for deployments |
| 5 | App servers | WAR pulled from JFrog; application service restarted |

> **Current state:** Step 5 is manual. Deployment automation is on the [Roadmap](#roadmap).

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Cloud provider | AWS | EC2, VPC, NLB, ASG, RDS |
| Networking | VPC, IGW, NAT Instance, Subnets, Security Groups | Private-first design |
| Load balancer | Network Load Balancer (NLB) | Layer 4, TCP pass-through |
| Proxy | NGINX | Reverse proxy to app tier |
| Application | Spring Boot (Java) | WAR deployment on EC2 |
| Database | RDS MySQL | `db.t3.micro`, Single AZ |
| Build | Apache Maven | WAR packaging |
| Code quality | SonarQube | Static analysis, quality gate |
| Artifact storage | JFrog Artifactory | Versioned artifact repository |
| Source control | Bitbucket | Feature branch workflow |

---

## Setup Guide

### 1. VPC and networking

```
VPC CIDR:        10.0.0.0/16
Public subnets:  10.0.1.0/24, 10.0.2.0/24  — NLB, Bastion, NAT
Private subnets: 10.0.3.0/24, 10.0.4.0/24  — NGINX, App, CI/CD
DB subnets:      10.0.5.0/24, 10.0.6.0/24  — RDS
```

- Attach Internet Gateway to VPC
- Add `0.0.0.0/0 → IGW` route to public subnet route table
- Launch NAT Instance in a public subnet; add `0.0.0.0/0 → NAT` to private subnet route tables

### 2. Security groups

Create the five security groups listed in [Security Design](#security-design). Reference SG IDs in rules — not CIDR ranges.

### 3. EC2 instances

| Instance | Subnet | Purpose |
|---|---|---|
| Bastion | Public | SSH access gateway |
| Maven server | Private | Build environment |
| SonarQube server | Private | Quality analysis |
| JFrog server | Private | Artifact repository |
| NGINX (ASG) | Private | Reverse proxy |
| App server (ASG) | Private | Spring Boot application |

### 4. Load balancer

- Create External NLB in public subnets
- Target group: NGINX instances on port 80/443
- Listener: port 80 → NGINX target group

### 5. NGINX configuration

```nginx
upstream app_servers {
    server <app-server-private-ip>:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Full config: `scripts/nginx.conf`

### 6. RDS

- Engine: MySQL 8.x
- Instance class: `db.t3.micro`
- Subnet group: DB private subnets
- Security group: `rds-sg` — inbound port 3306 from `app-sg` only
- Public accessibility: **No**

### 7. CI/CD setup

```bash
# Build WAR
mvn clean package -DskipTests

# SonarQube scan
mvn sonar:sonar -Dsonar.host.url=http://<sonar-server>:9000

# Upload to JFrog
curl -u user:password -T target/app.war \
  http://<jfrog-server>/artifactory/libs-release/app-1.0.war

# Deploy (via Bastion jump)
ssh -J ec2-user@<bastion-ip> ec2-user@<app-server-ip>
curl -O http://<jfrog-server>/artifactory/libs-release/app-1.0.war
sudo systemctl restart app-service
```

---

## Known Limitations

These are understood gaps — recognising them is part of the learning.

| Limitation | Impact | Resolution |
|---|---|---|
| Manual deployment | Slow releases; human error risk | Jenkins / GitHub Actions pipeline |
| Credentials baked into WAR | Not environment-portable | AWS SSM Parameter Store / Secrets Manager |
| No centralised logging | No visibility into errors at scale | CloudWatch Logs per tier |
| No monitoring or alerting | Failures discovered by users first | CloudWatch dashboards and alarms |
| Single-AZ RDS | Database is a single point of failure | Multi-AZ RDS with automatic failover |

---

## Roadmap

- [ ] Automated deployment via Jenkins or GitHub Actions
- [ ] Credentials moved to AWS SSM Parameter Store / Secrets Manager
- [ ] Zero-downtime rolling deployments via ASG instance refresh
- [ ] Centralised logging with CloudWatch Logs
- [ ] CloudWatch dashboards and alarms across all tiers
- [ ] Multi-AZ RDS for database high availability
- [ ] HTTPS with TLS termination at NLB using ACM

---

## Key Learnings

**Private subnet networking**
Understanding the routing difference between public subnets (IGW route) and private subnets (NAT route) — and why that distinction is what actually enforces network isolation, not just the subnet label.

**NLB vs ALB**
NLB passes TCP through at Layer 4 — NGINX sees the raw connection and handles all Layer 7 concerns itself. ALB would handle HTTP routing and headers at the AWS level. Choosing between them is a trade-off between control and managed capability, not just performance.

**Security group chaining**
Referencing SGs by ID instead of CIDR ranges means access control is tied to the resource's identity, not its address. This is the correct enterprise pattern — it survives subnet changes and is auditable.

**Multi-instance state**
Running more than one app server behind a load balancer immediately surfaces real problems: session state, configuration consistency, and log aggregation. These are things single-server tutorials skip entirely.

**What manual deployment teaches**
Doing the deploy manually — SSH into Bastion, jump to app server, pull WAR, restart service — makes it very clear what automation would eliminate and exactly where failures occur. The value of a CI/CD pipeline is most visible when you've done it by hand first.

---

## Author

**Simon Paul S** — DevOps Engineer · Practice Project 01

---
