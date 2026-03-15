# High-Availability AWS Architecture & SRE Analysis 

**Topics:** AWS, Cloud Architecture, Site Reliability Engineering, High Availability, DevOps, Observability

![AWS](https://img.shields.io/badge/Cloud-AWS-orange)
![Architecture](https://img.shields.io/badge/Architecture-High%20Availability-blue)
![SRE](https://img.shields.io/badge/Focus-Site%20Reliability%20Engineering-green)
![Database](https://img.shields.io/badge/Database-Amazon%20RDS-lightgrey)
![Observability](https://img.shields.io/badge/Monitoring-CloudWatch-yellow)


## 📌 Executive Summary

This repository documents the design, implementation, and Site Reliability Engineering (SRE) validation of a highly available, cloud-native **Student-Records Web Application**.

The objective of this project was to evolve a **single-node (EC2) proof-of-concept** into a **resilient Multi-AZ AWS architecture**, establish **realistic Service Level Objectives (SLOs)**, and conduct destructive load testing to measure system behavior during traffic spikes.

Key engineering areas explored:

* Error Budgeting
* Infrastructure scaling using AWS IaaS
* Observability using CloudWatch
* Root Cause Analysis (RCA) of cascading failures
* Chaos-style load testing


## 🔧 Skills Demonstrated

* AWS Cloud Architecture
* High Availability Design
* Auto Scaling & Load Balancing
* Site Reliability Engineering (SRE) Principles
* Service Level Objectives (SLOs)
* Observability with CloudWatch
* Load Testing & Performance Analysis
* Failure Root Cause Analysis


## 🏗️ Architecture Evolution

### Phase 1 — Proof of Concept (Single EC2 instance)

The initial architecture was a single-AZ deployment designed to validate the application logic.

It consisted of:

* A single **EC2 instance**
* **Node.js web server**
* **Local MySQL database**
* Internet access through an **Internet Gateway**
* Deployment within a single **Availability Zone**

This architecture successfully validated application functionality but introduced several limitations:

* Single point of failure
* No fault tolerance
* No horizontal scalability
* No database isolation

#### Architecture Diagram

<img src="./AWS%20Diagrams/1.%20Initial%20Phase%20Diagram.png" alt="Initial Architecture" width="600">

---

### Phase 2 — High-Availability Production Architecture

To meet reliability targets, the infrastructure was redesigned into a **fault-tolerant Multi-AZ architecture**.

#### Compute Tier

* EC2 instances deployed across **multiple Availability Zones**
* Managed by an **Auto Scaling Group (ASG)**

#### Traffic Routing

* **Application Load Balancer (ALB)** distributes traffic
* Health checks automatically remove unhealthy instances

#### Data Tier

* **Amazon RDS (MySQL)** deployed inside **private subnets**
* Database credentials securely stored in **AWS Secrets Manager**

#### Architecture Diagram

<img src="./AWS%20Diagrams/2.%20Final%20Diagram.png" alt="Final Architecture" width="700">

This design provides:

* High availability
* Automatic scaling
* Secure database isolation
* Fault tolerance across availability zones


## 🎯 Service Level Objectives (SLOs)

To define acceptable reliability levels, SLOs were calculated using a **rolling monthly window**.

Total minutes per month: **43,800 minutes**

### Defined SLO Targets

| Metric       | Target    |
| ------------ | --------- |
| Availability | > 99.9%   |
| Error Rate   | < 1%      |
| p95 Latency  | < 1500 ms |

### Error Budget

For **99.9% availability**, the system is allowed:

**43.8 minutes of downtime per month**


## 💥 Chaos Engineering & Performance Testing

To validate the Auto Scaling Group and infrastructure resilience, the system was subjected to sustained load tests of:

**2,000 Requests Per Second (RPS)**

Metrics were collected using:

* Client-side request load via Cloud9 IDE
* AWS CloudWatch monitoring
* Application Load Balancer metrics


### Test 1 — Cascading Failure (1 Instance Baseline)

#### Scenario

* 2,000 RPS directed to
* **Single t3.micro EC2 instance**

#### Result

The extreme load caused immediate **100% CPU utilization** on the EC2 instance.

As the first instance failed ALB health checks and was terminated, a new auto scaling instance was still in a 'provisioning' or 'warming' state.

#### Impact

| Metric       | Result    |
| ------------ | --------- |
| Availability | 67.56%    |
| Error Rate   | 32.44%    |
| p95 Latency  | 10,002 ms |

The system exhausted the **monthly error budget in under 5 minutes.**

---

### Test 2 — Mitigation & Validation (3 Instances Baseline)

#### Remediation

The Auto Scaling Group's desired baseline capacity was increased to:

**3 EC2 instances**

This allowed the system to absorb the initial traffic spike without CPU bottlenecking.

#### Result

The distributed compute load enabled efficient processing of requests while maintaining healthy database connections.

#### Impact

| Metric       | Result  |
| ------------ | ------- |
| Availability | 99.983% |
| Error Rate   | 0.005%  |
| p95 Latency  | 10 ms   |

The architecture remained stable and responsive under load.


## 📊 SLO Compliance Summary

| Metric       | SLO Target | Test 1 (1 Instance) | Test 2 (3 Instances) | Status   |
| ------------ | ---------- | ---------------- | ---------------- | -------- |
| Availability | > 99.9%    | 67.56%           | 99.983%          | ✅ PASSED |
| Error Rate   | < 1.0%     | 32.44%           | 0.005%           | ✅ PASSED |
| p95 Latency  | < 1,500 ms | 10,002 ms        | 10 ms            | ✅ PASSED |


## 🔍 Observability

System monitoring was implemented using **AWS CloudWatch**.

Key metrics observed:

* Application Availability
* ALB Error Rate **(HTTPCode_ELB_5XX_Count / RequestCount)**
* Application latency

### Proxy Service Level Indicator

Due to limitations in native AWS SQL monitoring, the following proxy metric was used:

**HTTPCode_ELB_5XX_Count**

This metric provided an effective indicator of backend failures.


## ⚙️ Technology Stack

### Cloud Infrastructure

* AWS EC2
* AWS VPC 
* Application Load Balancer
* Auto Scaling Groups

### Database & Security

* Amazon RDS (MySQL)
* AWS Secrets Manager
* Security Groups

### Observability

* AWS CloudWatch
* Metric Math dashboards

### Testing

* AWS Cloud9 IDE - High-concurrency CLI load testing


## 🧾 Architecture Decisions

### Decision 1 — Use Application Load Balancer

**Reason**

Application Load Balancer provides:

* Layer 7 routing
* Advanced health checks
* Integration with Auto Scaling Groups

This allows unhealthy instances to be automatically removed during failure scenarios.


### Decision 2 — Deploy RDS in Private Subnets

**Reason**

Databases should not be publicly accessible.

Placing RDS in private subnets:

* Reduces attack surface
* Restricts access to application servers
* Aligns with AWS security best practices


### Decision 3 — Use AWS Secrets Manager for Credentials

**Reason**

Hardcoding database credentials creates security risks.

Using Secrets Manager provides:

* Secure credential storage
* Centralized secret management
* Future capability for credential rotation


## 🧠 Core Engineering Takeaways

### The Danger of the Thundering Herd

Insufficient baseline capacity during traffic spikes can trigger cascading failures faster than Auto Scaling can respond.

**Correct baseline sizing is critical**


### Proxy Metrics for Observability

When direct application metrics are unavailable, load balancer error metrics can effectively represent backend health.


### Availability vs Latency Tradeoff

During heavy load, the architecture prioritized:

**remaining available rather than aggressively dropping connections**

This is a common reliability tradeoff in distributed systems.

---

<a href="https://github.com/georgecyberli" class="button icon back">Back to My Portfolio Page</a>
