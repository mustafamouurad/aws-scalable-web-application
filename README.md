

**AWS Solutions Architect – Associate | Graduation Project **

A production-grade, highly available web application deployed on AWS using EC2 instances inside a properly architected VPC, with public and private subnets spread across two Availability Zones.

## Table of Contents

- [Solution Overview](#solution-overview)
- [Architecture Diagram](#architecture-diagram)
- [AWS Services Used](#aws-services-used)
- [Deployment Steps](#deployment-steps)
  * [1. Networking (VPC)](#1-networking-vpc)
  * [2. Security](#2-security)
  * [3. Database (RDS Multi-AZ)](#3-database-rds-multi-az)
  * [4. Compute (EC2 + Auto Scaling)](#4-compute-ec2--auto-scaling)
  * [5. Load Balancing (ALB + WAF)](#5-load-balancing-alb--waf)
  * [6. Content Delivery (CloudFront)](#6-content-delivery-cloudfront)
  * [7. DNS (Route 53)](#7-dns-route-53)
  * [8. Secure Access (Systems Manager)](#8-secure-access-systems-manager)
  * [9. Monitoring (CloudWatch + SNS)](#9-monitoring-cloudwatch--sns)
- [Implementation Screenshots](#Implementation-Screenshots)
- [Testing & Verification](#testing--verification)
- [Cleanup](#cleanup)
- [Author](#author)

## Solution Overview

This solution deploys a scalable, fault-tolerant web application on AWS. All compute resources run in **private subnets** across two Availability Zones, with public-facing traffic routed through an **Application Load Balancer (ALB)** protected by **AWS WAF**. A **CloudFront** distribution sits in front of the ALB to reduce latency, and a **Multi-AZ RDS** instance provides the database backend with automated failover. **Auto Scaling** adjusts EC2 capacity based on CPU utilization, and **CloudWatch + SNS** provide monitoring and alerting. Secure operational access to instances is handled through **Systems Manager Session Manager**, with no SSH keys or bastion hosts required.

## Architecture Diagram

<img width="1536" height="1024" alt="aws project" src="https://github.com/user-attachments/assets/255ec69e-f713-4ac8-ba6c-3de4181a2a5f" />


<img width="2779" height="2381" alt="architecture-diagram" src="https://github.com/user-attachments/assets/f8998550-b959-4372-a75e-64a0f8149842" />




The architecture consists of:
- A VPC (`10.0.0.0/16`) spanning two Availability Zones, each with a public and a private subnet
- An Internet Gateway and two NAT Gateways (one per AZ) for outbound internet access from private subnets
- An Application Load Balancer in the public subnets, forwarding traffic to EC2 instances in the private subnets
- An Auto Scaling Group maintaining 2–4 EC2 instances based on CPU utilization
- A Multi-AZ RDS (MySQL) instance in the private subnets
- CloudFront in front of the ALB for latency reduction
- Route 53 for DNS resolution and health checking
- WAF attached to the ALB for OWASP Top 10 protection

## AWS Services Used

| Service | Purpose |
|---|---|
| **VPC** | Public/private subnets across 2 AZs, NAT Gateways, Security Groups, NACLs |
| **EC2 + Auto Scaling** | Launch Template, target-tracking scaling policy (CPU-based) |
| **Application Load Balancer + WAF** | Layer 7 routing, OWASP Top 10 managed rule protection |
| **CloudFront** | Edge caching and latency reduction in front of the ALB |
| **RDS (Multi-AZ)** | MySQL with automated failover to a standby instance |
| **Route 53** | Alias record to the ALB, endpoint health checks |
| **Systems Manager** | Session Manager for secure, keyless instance access |
| **CloudWatch + SNS** | Alarms on CPU, unhealthy targets, and database load; email notifications |

## Deployment Steps

> All resources were deployed manually via the AWS Management Console in the `us-west-1` region.

### 1. Networking (VPC)
- Created a VPC (`web-app-vpc`, `10.0.0.0/16`) with 4 subnets: 2 public, 2 private, one pair per AZ
- Attached an Internet Gateway to the VPC
- Created 2 NAT Gateways (one per public subnet) for AZ-independent outbound access
- Created 3 route tables: one shared public route table, and one private route table per AZ, each pointing to its own AZ's NAT Gateway
  
  

### 2. Security
- Created 3 Security Groups: `alb-sg` (80/443 from internet), `ec2-sg` (80 from `alb-sg` only), `rds-sg` (3306 from `ec2-sg` only)
- Created 2 Network ACLs: `public-nacl` and `private-nacl`, with appropriate inbound/outbound rules including ephemeral ports

### 3. Database (RDS Multi-AZ)
- Created a DB Subnet Group spanning both private subnets
- Launched a Multi-AZ MySQL RDS instance (`db.t3.micro`) with automated backups and a standby replica in the second AZ

### 4. Compute (EC2 + Auto Scaling)
- Created an IAM role (`ec2-ssm-role`) with the `AmazonSSMManagedInstanceCore` policy attached
- Created a Launch Template with a user data script that installs Apache and serves a page showing the instance ID
- Created an Auto Scaling Group (2 desired / 2 min / 4 max) spanning both private subnets, with a target-tracking scaling policy (CPU utilization target: 50%)

### 5. Load Balancing (ALB + WAF)
- Created a Target Group (`web-app-tg`, HTTP:80, health check path `/`)
- Created an internet-facing Application Load Balancer in the public subnets, listening on HTTP:80, forwarding to the target group
- Created a WAF Web ACL with AWS Managed Rule Groups (Core Rule Set, SQL Database, Known Bad Inputs) attached to the ALB for OWASP Top 10 coverage

### 6. Content Delivery (CloudFront)
- Created a CloudFront distribution with the ALB as its origin (HTTP only)
- Configured "Redirect HTTP to HTTPS" as the viewer protocol policy

### 7. DNS (Route 53)
- Created a public Hosted Zone
- Created an Alias (A) record pointing to the ALB
- Created a Health Check monitoring the ALB endpoint over HTTP on port 80

### 8. Secure Access (Systems Manager)
- Verified Session Manager connectivity to EC2 instances directly from the console with no SSH key or bastion host required

### 9. Monitoring (CloudWatch + SNS)
- Created an SNS topic (`web-app-alerts`) with an email subscription
- Created 3 CloudWatch Alarms, all notifying the SNS topic:
  - `web-app-high-cpu-alarm` — ASG average CPU > 70%
  - `web-app-rds-high-cpu-alarm` — RDS CPU > 80%
  - `web-app-unhealthy-targets` — ALB target group has any unhealthy targets

## Implementation Screenshots

### 1. Networking (VPC)
<img width="1920" height="826" alt="Nat GW" src="https://github.com/user-attachments/assets/1fe41e57-48c8-473a-b011-714ba6df178b" />
<img width="1920" height="809" alt="IGW" src="https://github.com/user-attachments/assets/c0fb0dff-8442-4557-8087-a9bde2bf933b" />
<img width="1920" height="807" alt="RouteTables" src="https://github.com/user-attachments/assets/da57517f-b2b6-4d9e-bd52-c3cabef92f55" />
<img width="1920" height="811" alt="subnets" src="https://github.com/user-attachments/assets/02be62ac-7cc0-4d70-bde9-f8e455937cdb" />
<img width="1920" height="812" alt="vpc" src="https://github.com/user-attachments/assets/db2b50d4-3cb2-419d-ab6e-903fba4cc30d" />


### 2. Security
<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/39b5bba5-8af1-4671-a96c-7d382503d2c4" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/e03107a2-bdb2-4a5f-b241-5b4cf7c6a52b" />



### 3. Database (RDS Multi-AZ)
<img width="1920" height="1080" alt="623660237-cbd62e42-62ce-42d1-9eaf-46c2b1f450b5" src="https://github.com/user-attachments/assets/78f5ddc2-fc56-456c-82f4-0df282c86633" />


### 4. Compute (EC2 + Auto Scaling)
<img width="1920" height="1080" alt="7" src="https://github.com/user-attachments/assets/4a8dee76-d2f2-4420-8894-85f857234132" />

<img width="1920" height="1080" alt="6" src="https://github.com/user-attachments/assets/cf9af760-9795-4e8d-b9e7-677539ef3982" />

<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/1a5d4a9b-2d45-4a90-88af-6a794d1bb4ab" />

<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/ed176267-9c2f-4185-bf8c-c5d2982f000e" />

<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/9b01b365-17f2-48e8-96ad-006e469cc9f3" />

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/fa78452f-9995-449f-b83a-2205c93f1982" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/f3551b78-84d5-468e-9764-b73d355f30e6" />




### 5. Load Balancing (ALB + WAF)
<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/d65d141e-9ec8-4e82-8f80-5d4e4a8efe18" />

<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/824bbc9f-b313-4094-965e-ec4e9abeb824" />

<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/7b189c04-8a29-4fe6-9ecf-0260fb5508f2" />

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/d0391ee3-4603-422a-b256-96287ce39032" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/f8707bd7-3324-4552-b8a1-904714af63e4" />



### 6. Content Delivery (CloudFront)

<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/5b7851ed-22da-4099-b89e-58163a3a6921" />

<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/66be0cd2-8053-4767-a4d5-911c76bae064" />

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/0b071b0b-072e-4661-b69c-6f3a50f38024" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/bdba19b3-8407-4ef7-97e6-c83f9f360796" />


### 7. DNS (Route 53)
<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/a7361220-7b58-4e67-bf7a-1b9671abcfa2" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/99f2a06a-2dd5-42d1-b32e-6b97f9925252" />


### 8. Secure Access (Systems Manager)
<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/9eb67b3a-e05a-4add-8c24-92da720430f1" />

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/5838128d-919b-4560-9039-7100d0890de2" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/cef299fb-c2f9-4619-9b38-a7f52ba9d53e" />



### 9. Monitoring (CloudWatch + SNS)
<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/f8cad4ee-f396-4a40-8bea-511bca763343" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/ae695536-f334-4556-bc29-74a825676bd3" />



## Testing & Verification

- Confirmed the ALB distributes traffic across both Availability Zones by reloading the application URL and observing the displayed EC2 instance ID change between requests
- Verified Session Manager access to an EC2 instance without SSH keys or a bastion host
- Confirmed the Route 53 health check reports the ALB endpoint as healthy
- Confirmed SNS email subscription is active (status: Confirmed)

**Live URL (via CloudFront):** `http://d1a2b3c4d5e6f7.cloudfront.net`
**Direct ALB URL (bypassing CDN):** `http://web-app-alb-611387839.us-west-1.elb.amazonaws.com`


## Cleanup

To avoid ongoing charges after evaluation, the following resources should be deleted in this order: CloudFront distribution → WAF Web ACL → ALB → Auto Scaling Group → Launch Template → RDS instance (disable deletion protection first) → NAT Gateways → Elastic IPs → Route 53 Hosted Zone → VPC.

## Author

**Moustafa Mahmoud**
AWS Solutions Architect – Associate, Graduation Project
