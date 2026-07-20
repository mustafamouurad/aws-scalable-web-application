

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

<img width="1920" height="813" alt="2" src="https://github.com/user-attachments/assets/099c42d1-4bf5-4cad-a752-16571efe8a81" />

<img width="1920" height="816" alt="1" src="https://github.com/user-attachments/assets/530e91ac-0400-457e-a4c4-977a79bef5e0" />



### 3. Database (RDS Multi-AZ)

<img width="1920" height="818" alt="623660237-cbd62e42-62ce-42d1-9eaf-46c2b1f450b5" src="https://github.com/user-attachments/assets/15f27150-e359-4d7c-b8d7-594f8e393907" />


### 4. Compute (EC2 + Auto Scaling)

<img width="1920" height="821" alt="7" src="https://github.com/user-attachments/assets/10364422-f75d-4a07-9319-bfe00aea4c03" />

<img width="1920" height="831" alt="6" src="https://github.com/user-attachments/assets/dcf4f222-71fa-4723-be25-c2c17912d7a5" />

<img width="1920" height="816" alt="5" src="https://github.com/user-attachments/assets/4aa870f7-6db6-49b7-9f0d-334d6ef1deae" />

<img width="1920" height="828" alt="4" src="https://github.com/user-attachments/assets/246da2d2-2156-45ff-83fb-eda499be9475" />

<img width="1920" height="831" alt="3" src="https://github.com/user-attachments/assets/9e889fb5-21fb-4fed-bf1c-c71415a796f9" />

<img width="1920" height="827" alt="2" src="https://github.com/user-attachments/assets/d6cec6b7-ff59-4944-a92a-2955258d3ea7" />

<img width="1920" height="817" alt="1" src="https://github.com/user-attachments/assets/35c79e3d-88d6-41a1-9466-9fb16ab2f712" />




### 5. Load Balancing (ALB + WAF)

<img width="1920" height="825" alt="5" src="https://github.com/user-attachments/assets/25241e66-a4ac-4ac9-b4f8-734a0ad9e98e" />

<img width="1920" height="822" alt="4" src="https://github.com/user-attachments/assets/aa6a9d7c-7b58-4085-a5d5-b66719570260" />

<img width="1920" height="825" alt="3" src="https://github.com/user-attachments/assets/97d5d40e-e9f0-4e23-870d-3a5d2fb7986c" />

<img width="1920" height="824" alt="2" src="https://github.com/user-attachments/assets/e501359d-bf7f-4d51-ba40-38911f964a8a" />

<img width="1920" height="799" alt="1" src="https://github.com/user-attachments/assets/d380da70-e3e6-4b5d-941c-8633d3c9828b" />



### 6. Content Delivery (CloudFront)

<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/5b7851ed-22da-4099-b89e-58163a3a6921" />

<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/66be0cd2-8053-4767-a4d5-911c76bae064" />

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/0b071b0b-072e-4661-b69c-6f3a50f38024" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/bdba19b3-8407-4ef7-97e6-c83f9f360796" />


### 7. DNS (Route 53)

<img width="1920" height="828" alt="2" src="https://github.com/user-attachments/assets/661476b9-8986-45ba-aaf6-c1d2e52bef84" />

<img width="1920" height="804" alt="1" src="https://github.com/user-attachments/assets/25b72dcf-488d-4c5e-8da7-9dde6cfbea0e" />



### 8. Secure Access (Systems Manager)
<img width="1920" height="822" alt="3" src="https://github.com/user-attachments/assets/5bba3176-e6ca-432d-b49c-1a2f4db2d63f" />

<img width="1920" height="798" alt="2" src="https://github.com/user-attachments/assets/5482d7cd-8229-4f61-adaf-7ca3596ff4f2" />

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/e2200c5d-c63f-463b-bd66-eb814a4f7ec1" />




### 9. Monitoring (CloudWatch + SNS)

<img width="1920" height="776" alt="2" src="https://github.com/user-attachments/assets/6a82673a-f7eb-4890-881c-782687815439" />

<img width="1920" height="785" alt="1" src="https://github.com/user-attachments/assets/ec8ea51c-fa9c-4497-b2b7-70e5172c3020" />




## Testing & Verification

- Confirmed the ALB distributes traffic across both Availability Zones by reloading the application URL and observing the displayed EC2 instance ID change between requests
- Verified Session Manager access to an EC2 instance without SSH keys or a bastion host
- Confirmed the Route 53 health check reports the ALB endpoint as healthy
- Confirmed SNS email subscription is active (status: Confirmed)

**Live URL (via CloudFront):** `http://d1a2b3c4d5e6f7.cloudfront.net`
**Direct ALB URL (bypassing CDN):** `http://web-app-alb-611387839.us-west-1.elb.amazonaws.com`

> **Note:** Live URLs were active during development and testing (see screenshots above showing the CloudFront-served app with changing Instance IDs). All AWS resources were terminated after project completion to avoid ongoing charges — see [Cleanup](#cleanup) section.


## Cleanup

To avoid ongoing charges after evaluation, the following resources should be deleted in this order: CloudFront distribution → WAF Web ACL → ALB → Auto Scaling Group → Launch Template → RDS instance (disable deletion protection first) → NAT Gateways → Elastic IPs → Route 53 Hosted Zone → VPC.

## Author

**Moustafa Mahmoud**
AWS Solutions Architect – Associate, Graduation Project
