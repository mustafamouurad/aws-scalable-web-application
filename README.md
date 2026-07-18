# Scalable Web Application with ALB and Auto Scaling

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
- [Testing & Verification](#testing--verification)
- [Cost Considerations](#cost-considerations)
- [Cleanup](#cleanup)
- [Author](#author)

## Solution Overview

This solution deploys a scalable, fault-tolerant web application on AWS. All compute resources run in **private subnets** across two Availability Zones, with public-facing traffic routed through an **Application Load Balancer (ALB)** protected by **AWS WAF**. A **CloudFront** distribution sits in front of the ALB to reduce latency, and a **Multi-AZ RDS** instance provides the database backend with automated failover. **Auto Scaling** adjusts EC2 capacity based on CPU utilization, and **CloudWatch + SNS** provide monitoring and alerting. Secure operational access to instances is handled through **Systems Manager Session Manager**, with no SSH keys or bastion hosts required.

## Architecture Diagram

<img width="1536" height="1024" alt="aws project" src="https://github.com/user-attachments/assets/255ec69e-f713-4ac8-ba6c-3de4181a2a5f" />



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

## Testing & Verification

- Confirmed the ALB distributes traffic across both Availability Zones by reloading the application URL and observing the displayed EC2 instance ID change between requests
- Verified Session Manager access to an EC2 instance without SSH keys or a bastion host
- Confirmed the Route 53 health check reports the ALB endpoint as healthy
- Confirmed SNS email subscription is active (status: Confirmed)

**Live URL:** `http://<cloudfront-distribution-domain>.cloudfront.net`

## Cost Considerations

The most significant recurring costs in this architecture are the **2 NAT Gateways** and the **Multi-AZ RDS instance** (~$37/month). WAF adds a small monthly cost (~$8/month for the Web ACL + 3 managed rule groups). CloudFront and Route 53 Hosted Zone usage are minimal for demo-level traffic.

## Cleanup

To avoid ongoing charges after evaluation, the following resources should be deleted in this order: CloudFront distribution → WAF Web ACL → ALB → Auto Scaling Group → Launch Template → RDS instance (disable deletion protection first) → NAT Gateways → Elastic IPs → Route 53 Hosted Zone → VPC.

## Author

**Moustafa Mahmoud**
AWS Solutions Architect – Associate, Graduation Project
