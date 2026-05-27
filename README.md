# Auto-Scaling Event-Driven Data Pipeline on AWS

Designed and implemented a fully elastic, event-driven word frequency processing pipeline on AWS, plus a companion AI web application cloud architecture (ArtAI). Built as part of the Large-Scale Data Engineering module (MSc Data Science, University of Bristol).

## Architecture Overview

```
User uploads .txt files
        │
        ▼
   ┌─────────┐      S3 Event        ┌──────────┐
   │  S3     │  ──────────────────►  │  SQS     │
   │ Bucket  │   Notification        │  Queue   │
   └─────────┘                       └────┬─────┘
                                          │
                              Poll messages│
                                          ▼
                                ┌──────────────────┐
                                │  EC2 Auto        │
                                │  Scaling Group   │
                                │  (1-5 instances) │
                                └────────┬─────────┘
                                         │
                                Write results
                                         │
                                         ▼
                                ┌──────────────────┐
                                │   DynamoDB       │
                                │   Results Table  │
                                └──────────────────┘
```

## Part 1: ArtAI Cloud Design

Designed a production-grade AWS architecture for an AI-powered image processing web application with the following components:

| Layer | AWS Services | Purpose |
|-------|-------------|---------|
| **Frontend & CDN** | S3, CloudFront, WAF | Global low-latency delivery, DDoS protection, SQL injection prevention |
| **Authentication** | Cognito | User registration, login, token management |
| **API Layer** | API Gateway | Secure entry point, token verification, traffic control |
| **Compute** | ECS Fargate, ALB | Serverless containers, health-checked load balancing |
| **AI Inference** | SageMaker | Real-time model inference via private VPC endpoint |
| **Storage** | S3 (SSE-KMS), Cross-Region Replication | Encrypted storage, versioning, disaster recovery |
| **Networking** | VPC, Private Subnets, NAT Gateway, VPC Endpoints | Zero Trust network model, no public IPs on backend |
| **Monitoring** | CloudWatch, SNS | Performance metrics, automatic alerts at 10K+ daily requests |
| **Backup** | AWS Backup, S3 Glacier lifecycle | Automated backup, compliance-ready disaster recovery |

## Part 2: Scaling the WordFreq Application

### Task A — Baseline Deployment

Established a fully functional event-driven processing pipeline:

- Provisioned EC2 instance (Ubuntu 24.04 LTS, t2.micro) with Go runtime and WordFreq binaries
- Created S3 buckets (upload + processing) and SQS queues (jobs + results)
- Configured S3 event notifications to auto-generate SQS job messages on file upload
- Validated complete pipeline: S3 → SQS → EC2 → DynamoDB
- Configured worker as systemd service for automatic boot startup
- Created reusable Golden AMI from configured instance

### Task B — Auto-Scaling Implementation

**Step 1: Golden AMI Creation**
- Converted fully configured dev instance into immutable, reusable AMI
- Eliminates configuration drift between workers
- Pre-installed: Go runtime, application binaries, systemd service, AWS service references

**Step 2: Launch Template**
- Created EC2 Launch Template from Golden AMI
- Attached EMR_EC2_DefaultRole IAM instance profile (least privilege access)
- Configured security group, key pair, and networking
- Saved as versioned template for immutable infrastructure

**Step 3: Auto Scaling Group (ASG)**
- Created wordfreq-asg with capacity: Min 1 / Desired 1 / Max 5
- Deployed across multiple Availability Zones for high availability
- No load balancer needed (workers pull directly from SQS)
- Self-healing: automatic replacement of failed instances

**Step 4: Dynamic Scaling Policies**

| Policy | Metric | Trigger | Action | Cooldown |
|--------|--------|---------|--------|----------|
| **Scale-Out** | SQS ApproximateNumberOfMessagesVisible | ≥10 messages for 1 min | Add 1 instance | 120s |
| **Scale-In** | SQS ApproximateNumberOfMessagesVisible | 0 messages for 5 min | Remove 1 instance | 120s |

Scaling based on SQS queue depth (not CPU) — appropriate for I/O-bound workloads.

### Task C — Load Testing

- Injected ~130 text documents into S3 processing bucket
- Observed SQS queue backlog spike to 89 messages
- Verified ASG automatic scale-out (multiple EC2 worker instances launched)
- Confirmed 100+ records written to DynamoDB by parallel workers
- Validated complete closed-loop: S3 upload → SQS backlog → CloudWatch alarm → ASG scale-out → EC2 processing → DynamoDB results
- No manual intervention required — fully self-managing compute fleet

### Task D — Optimised Architecture

Enhanced the baseline system for production readiness:

- **High Availability:** Multi-AZ ASG in private VPC subnets
- **Disaster Recovery:** S3 versioning + Glacier lifecycle policies, DynamoDB Point-in-Time Recovery (PITR)
- **Cost Optimisation:** ASG Min capacity set to 0 (zero cost when idle), Spot Instance support
- **Security:** Private subnets only, no public IPs, VPC Endpoints for S3/SQS/DynamoDB, IAM least privilege, CloudTrail audit logging
- **Monitoring:** CloudWatch metrics and scaling alarms

### Task E — Further Improvements Analysis

Evaluated alternative architectures for different scale requirements:

| Architecture | Best For | Key Advantages |
|-------------|----------|----------------|
| **Current (EC2 ASG)** | Moderate workloads | Full control, flexible instance types |
| **AWS Lambda** | Small-medium files, event-driven | Zero server management, pay-per-execution, built-in retry |
| **AWS EMR (Spark)** | Large-scale batch analytics | Distributed processing, in-memory computing, Spot cluster support |

## AWS Services Used

- **Compute:** EC2, Auto Scaling Groups, ECS Fargate, SageMaker
- **Storage:** S3 (SSE-KMS encryption), DynamoDB, S3 Glacier
- **Messaging:** SQS, SNS
- **Networking:** VPC, Private/Public Subnets, NAT Gateway, VPC Endpoints, ALB
- **Security:** IAM Roles, Cognito, WAF, CloudTrail, Security Groups
- **Monitoring:** CloudWatch (Alarms, Metrics), CloudWatch Logs
- **CDN:** CloudFront
- **API:** API Gateway
- **Backup:** AWS Backup, Cross-Region Replication, DynamoDB PITR

## Key Concepts Demonstrated

- Event-driven architecture (S3 notifications → SQS → worker fleet)
- Immutable infrastructure (Golden AMI + versioned Launch Templates)
- Queue-based auto-scaling (SQS depth, not CPU utilisation)
- Zero Trust networking (private subnets, VPC endpoints, no public IPs)
- Cost optimisation (Spot Instances, Min:0 idle scaling, Glacier lifecycle)
- Multi-AZ high availability and disaster recovery
- Infrastructure as configuration (IAM roles, security groups, scaling policies)

## Tech Stack

- **Cloud:** AWS (15+ services)
- **Application:** Go (WordFreq worker)
- **OS:** Ubuntu Server 24.04 LTS
- **Process Management:** systemd
- **CLI Tools:** AWS CLI, SSH

## Author

**Jitendra Suwalka**
MSc Data Science — University of Bristol
- [LinkedIn](https://www.linkedin.com/in/jitendra-suwalka-ds)
- [GitHub](https://github.com/jitendraSuwalka)
