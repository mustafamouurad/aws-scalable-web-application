<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/60d27739-ca86-4111-a836-b970190940f8" /># Scalable Web Application with ALB and Auto Scaling

**AWS Solutions Architect – Associate | Graduation Project 1**

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
<img width="1920" height="1080" alt="Screenshot (788)" src="https://github.com/user-attachments/assets/a7cb085e-9ac7-4607-ab4c-e794c00404c8" />
<img width="1920" height="1080" alt="Screenshot (787)" src="https://github.com/user-attachments/assets/595403a4-ab33-4c46-9421-ed68e47b3ea4" />
<img width="1920" height="1080" alt="Screenshot (786)" src="https://github.com/user-attachments/assets/e6c4e1fa-f54a-4265-946e-3f32fc5dc438" />
<img width="1920" height="1080" alt="Screenshot (785)" src="https://github.com/user-attachments/assets/e47eee9d-d0c9-4e52-9bef-b4572c54eb24" />
<img width="1920" height="1080" alt="Screenshot (784)" src="https://github.com/user-attachments/assets/8f14504b-f2ad-427c-9f94-eee5366a067d" />


### 2. Security
<img width="1920" height="1080" alt="Screenshot (792)" src="https://github.com/user-attachments/assets/c6b9fb43-0290-4d1b-93ba-c64b09aea75e" />
<img width="1920" height="1080" alt="Screenshot (791)" src="https://github.com/user-attachments/assets/6414a8ff-f5b7-47ba-9f0f-ffbfe1a47c71" />


### 3. Database (RDS Multi-AZ)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cbd62e42-62ce-42d1-9eaf-46c2b1f450b5" />


### 4. Compute (EC2 + Auto Scaling)
<img width="1920" height="1080" alt="Screenshot (803)" src="https://github.com/user-attachments/assets/26470481-8f9d-4d2a-8a1a-42fa95debae4" />
<img width="1920" height="1080" alt="Screenshot (802)" src="https://github.com/user-attachments/assets/d255fa23-bea1-48f4-a250-5194bd7a7f82" />
<img width="1920" height="1080" alt="Screenshot (801)" src="https://github.com/user-attachments/assets/da8c02ad-ed0a-4129-b1f2-bad2ab2d05f8" />
<img width="1920" height="1080" alt="Screenshot (800)" src="https://github.com/user-attachments/assets/c81a0c8d-ce3f-4564-ab99-d9c008dc3f43" />
<img width="1920" height="1080" alt="Screenshot (799)" src="https://github.com/user-attachments/assets/51a42263-eed3-443e-b043-ec9e852b685d" />
<img width="1920" height="1080" alt="Screenshot (798)" src="https://github.com/user-attachments/assets/44bab338-5dac-491d-b40c-a26c1208a91f" />
<img width="1920" height="1080" alt="Screenshot (797)" src="https://github.com/user-attachments/assets/908790ab-a9a5-4e3f-a275-295ff64c8956" />



### 5. Load Balancing (ALB + WAF)
<img width="1920" height="1080" alt="Screenshot (810)" src="https://github.com/user-attachments/assets/7f2175f3-eef6-4270-bb17-bfbc0690d40c" />
<img width="1920" height="1080" alt="Screenshot (809)" src="https://github.com/user-attachments/assets/ac502084-edf1-496e-91e2-2d0080c5ca26" />
<img width="1920" height="1080" alt="Screenshot (808)" src="https://github.com/user-attachments/assets/4c1e978a-4ab2-4cd6-bf1c-87e0e0731771" />
<img width="1920" height="1080" alt="Screenshot (807)" src="https://github.com/user-attachments/assets/be64f6b1-00f5-40c3-8304-7f56229d62bb" />
<img width="1920" height="1080" alt="Screenshot (806)" src="https://github.com/user-attachments/assets/a8b9cabf-9fb7-4dd9-9870-faf327a9a449" />


### 6. Content Delivery (CloudFront)

<img width="1920" height="1080" alt="Screenshot (817)" src="https://github.com/user-attachments/assets/76364683-7390-409e-b938-35706769356f" />
<img width="1920" height="1080" alt="Screenshot (816)" src="https://github.com/user-attachments/assets/c72c31f1-a7d3-4315-ac4a-8f6cd9418f53" />
<img width="1920" height="1080" alt="Screenshot (814)" src="https://github.com/user-attachments/assets/a3b02186-0a0f-44e4-8940-bc65dd637d36" />
<img width="1920" height="1080" alt="Screenshot (813)" src="https://github.com/user-attachments/assets/916d94f0-0b5a-4973-aabb-5eb25b9c865b" />

### 7. DNS (Route 53)
<img width="1920" height="1080" alt="Screenshot (819)" src="https://github.com/user-attachments/assets/42ef774e-cbb5-4395-bb3b-5f3a38b19d48" />
<img width="1920" height="1080" alt="Screenshot (818)" src="https://github.com/user-attachments/assets/2c145986-0382-4fe7-b3c0-cdbc8c9e6f4d" />

### 8. Secure Access (Systems Manager)
<img width="1920" height="1080" alt="Screenshot (824)" src="https://github.com/user-attachments/assets/bb3a7de8-03d1-4b2e-886d-a79cfc7b425a" />
<img width="1920" height="1080" alt="Screenshot (823)" src="https://github.com/user-attachments/assets/d99a58f2-8920-4522-a8b2-fa5fdee70986" />
<img width="1920" height="1080" alt="Screenshot (822)" src="https://github.com/user-attachments/assets/3a7b154a-2f1f-4cab-9981-927427e916cd" />


### 9. Monitoring (CloudWatch + SNS)
<img width="1920" height="1080" alt="Screenshot (821)" src="https://github.com/user-attachments/assets/511bf6a8-6c90-46ef-a0f2-3d8663ec4175" />
<img width="1920" height="1080" alt="Screenshot (820)" src="https://github.com/user-attachments/assets/d233e3a4-4b59-420b-890b-6bb87aad6d48" />


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
