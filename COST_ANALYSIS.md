# KnowledgeCity Platform - Cost Analysis & Optimization

## Executive Summary

**Estimated Monthly Cost (Per Region)**: $10,125  
**Total (2 Regions)**: $20,250/month  
**Annual Cost**: $243,000  

**With Optimizations**: ~$145,000/year (**40% savings**)

---

## Detailed Cost Breakdown (Per Region)

### 1. Compute Resources

#### EKS Control Plane
```
Service: Amazon EKS
Cost: $0.10/hour × 730 hours = $73/month
```

#### EKS Worker Nodes (General Purpose)
```
Instance Type: t3.xlarge (4 vCPU, 16 GB RAM)
Configuration:
  - Base: 6 nodes (2 per AZ)
  - Average: 12 nodes (auto-scaling)
  - Peak: 30 nodes

Pricing:
  - On-Demand: $0.1664/hour
  - 1-Year Reserved: $0.0998/hour (40% savings)
  - 3-Year Reserved: $0.0664/hour (60% savings)

Calculation (with Savings Plans):
  - 12 nodes × $0.0998/hour × 730 hours = $873/month
  - Peak scaling (18 additional nodes × 2 hours/day):
    18 × $0.1664 × 60 = $180/month

Total: $1,053/month
```

#### EKS Worker Nodes (Compute Optimized - Video Processing)
```
Instance Type: c6i.2xlarge (8 vCPU, 16 GB RAM)
Configuration:
  - Base: 0 nodes
  - Average: 3 nodes
  - 70% Spot Instances, 30% On-Demand

Pricing:
  - On-Demand: $0.34/hour
  - Spot: $0.102/hour (70% discount average)

Calculation:
  - Spot: 2.1 nodes × $0.102 × 730 = $156/month
  - On-Demand: 0.9 nodes × $0.34 × 730 = $223/month

Total: $379/month
```

#### Fargate (Video Processing Burst)
```
Configuration:
  - 0.5 vCPU × 1 GB RAM per task
  - 1,000 tasks/day × 5 minutes average
  - 5,000 minutes/day = ~83 hours/day

Pricing:
  - $0.04048/hour per task

Calculation:
  - 83 hours × 30 days × $0.04048 = $101/month

Total: $101/month
```

**Compute Subtotal**: $1,606/month

---

### 2. Database Services

#### RDS PostgreSQL (Primary Database)
```
Instance Type: db.r6g.2xlarge (8 vCPU, 64 GB RAM)
Configuration:
  - Multi-AZ deployment
  - 500 GB storage (gp3)
  - 3,000 IOPS
  - 35-day backup retention

Pricing:
  - On-Demand: $1.008/hour × 2 (Multi-AZ) = $2.016/hour
  - 1-Year Reserved: $0.635/hour × 2 = $1.27/hour (37% savings)
  
Calculation:
  - Instance: $1.27 × 730 = $927/month
  - Storage: 500 GB × $0.138 = $69/month
  - Backup Storage: 500 GB × $0.095 = $48/month
  - IOPS: 3,000 × $0.006 = $18/month

Total: $1,062/month
```

#### RDS Proxy
```
Pricing: $0.015/hour per vCPU
Configuration: 8 vCPUs

Calculation:
  - 8 × $0.015 × 730 = $88/month

Total: $88/month
```

#### ElastiCache Redis (Multi-AZ)
```
Instance Type: cache.r6g.large (2 vCPU, 13.07 GB RAM)
Configuration:
  - 3 nodes (1 per AZ)
  - Cluster mode enabled
  
Pricing:
  - On-Demand: $0.226/hour
  - 1-Year Reserved: $0.158/hour (30% savings)

Calculation:
  - 3 nodes × $0.158 × 730 = $346/month

Total: $346/month
```

#### ClickHouse on EKS (Self-Managed)
```
Instance Type: r6i.2xlarge (8 vCPU, 64 GB RAM) equivalent
Configuration: 3 pods across 3 AZs

Storage:
  - 500 GB gp3 EBS per pod = 1,500 GB total
  - Pricing: $0.096/GB-month

Calculation:
  - Compute: Included in EKS worker nodes
  - Storage: 1,500 GB × $0.096 = $144/month

Total: $144/month
```

**Database Subtotal**: $1,640/month

---

### 3. Storage Services

#### S3 Storage
```
Regional Bucket (User Data):
  - 5 TB user-generated content
  - Intelligent-Tiering storage class
  
Pricing:
  - Frequent Access: 1 TB × $0.023 = $23
  - Infrequent Access: 3 TB × $0.0125 = $38
  - Archive: 1 TB × $0.004 = $4
  - Monitoring: 5,000 objects × $0.0025/1000 = $13

Total Regional: $78/month

Global Content Bucket (Courses):
  - 10 TB video content
  - Standard storage with lifecycle
  
Pricing:
  - First 50 TB: 10,000 GB × $0.023 = $230

Total Global: $230/month
```

#### EBS Volumes
```
gp3 volumes for Kubernetes:
  - Application pods: 200 GB × 20 pods = 4,000 GB
  - Database pods: 500 GB × 3 pods = 1,500 GB
  - Total: 5,500 GB
  
Pricing: $0.096/GB-month

Calculation:
  - 5,500 × $0.096 = $528/month

Total: $528/month
```

#### EBS Snapshots
```
Daily snapshots: 2 TB incremental
Pricing: $0.05/GB-month

Calculation:
  - 2,000 GB × $0.05 = $100/month

Total: $100/month
```

**Storage Subtotal**: $936/month

---

### 4. Network & Content Delivery

#### Application Load Balancer
```
Configuration:
  - 1 ALB per region
  - 100 GB processed/hour
  
Pricing:
  - ALB: $0.0225/hour × 730 = $16.43
  - LCU: 100 LCUs × $0.008 × 730 = $584

Total: $600/month
```

#### NAT Gateways
```
Configuration: 3 NAT Gateways (1 per AZ)

Pricing:
  - $0.045/hour × 3 × 730 = $99
  - Data Processing: 1 TB × 3 × $0.045 = $135

Total: $234/month
```

#### VPC Endpoints
```
Configuration:
  - 3 Interface Endpoints (ECR API, ECR DKR, SecretsManager)
  - 1 Gateway Endpoint (S3 - Free)

Pricing:
  - $0.01/hour × 3 × 730 = $22
  - Data Processing: 500 GB × $0.01 = $5

Total: $27/month
```

#### CloudFront (Global CDN)
```
Configuration:
  - 50 TB data transfer/month
  - 100M requests/month
  - Origin Shield enabled (2 regions)

Pricing:
  - First 10 TB: 10,000 GB × $0.085 = $850
  - Next 40 TB: 40,000 GB × $0.080 = $3,200
  - Requests: 100M × $0.0075/10,000 = $75
  - Origin Shield: 2 × $0.0100/hour × 730 = $15

Total: $4,140/month (shared across regions)
Per Region: $2,070/month
```

#### Data Transfer Out
```
Inter-AZ transfer: 500 GB × $0.01 = $5/month
Cross-region S3 replication: 1 TB × $0.02 = $20/month

Total: $25/month
```

**Network Subtotal**: $2,956/month

---

### 5. Security & Compliance

#### AWS Secrets Manager
```
Secrets: 20 secrets
API calls: 10,000/month

Pricing:
  - Secrets: 20 × $0.40 = $8
  - API calls: 10,000 × $0.05/10,000 = $0.50

Total: $8.50/month
```

#### AWS KMS
```
Customer-managed keys: 5
Requests: 100,000/month

Pricing:
  - Keys: 5 × $1 = $5
  - Requests: 100,000 × $0.03/10,000 = $0.30

Total: $5.30/month
```

#### AWS Certificate Manager
```
Public certificates: Free
Private certificates: 0

Total: $0/month
```

#### CloudTrail
```
One trail per region (free for management events)
Data events: 1M events

Pricing:
  - Data events: $0.10/100,000 events
  - 1,000,000 × $0.10/100,000 = $1

Total: $1/month
```

#### GuardDuty
```
CloudTrail analysis: 1M events
VPC Flow Logs: 500 GB
DNS logs: 100 GB

Pricing:
  - CloudTrail: First 500K free, then 500K × $4.00/million = $2
  - VPC Flow: 500 GB × $0.50 = $250
  - DNS: 100 GB × $0.30 = $30

Total: $282/month
```

#### AWS WAF (at ALB)
```
Web ACL: 1
Rules: 10
Requests: 100M/month

Pricing:
  - Web ACL: $5
  - Rules: 10 × $1 = $10
  - Requests: 100M × $0.60/million = $60

Total: $75/month
```

**Security Subtotal**: $372/month

---

### 6. Monitoring & Operations

#### CloudWatch Logs
```
Ingestion: 100 GB/month
Storage: 500 GB

Pricing:
  - Ingestion: 100 × $0.50 = $50
  - Storage: 500 × $0.03 = $15

Total: $65/month
```

#### CloudWatch Metrics
```
Custom metrics: 1,000
API requests: 100,000

Pricing:
  - Metrics: 1,000 × $0.30 = $300
  - API: 100,000 × $0.01/1,000 = $1

Total: $301/month
```

#### Prometheus/Grafana/Loki (Self-Hosted on EKS)
```
Storage: 200 GB EBS (included in EBS costs)
Compute: Included in EKS worker nodes

Total: $0 additional
```

#### Backup & Disaster Recovery
```
RDS automated backups: Included (35 days)
Manual snapshots: 500 GB × $0.095 = $48
S3 versioning overhead: 1 TB × $0.023 = $23

Total: $71/month
```

**Monitoring Subtotal**: $437/month

---

### 7. Additional Services

#### Route 53
```
Hosted zones: 2
Queries: 100M/month
Health checks: 4

Pricing:
  - Hosted zones: 2 × $0.50 = $1
  - Queries: 100M × $0.40/million = $40
  - Health checks: 4 × $0.50 × 730 hours / 730 = $2

Total: $43/month (shared across regions)
Per Region: $21.50/month
```

#### Systems Manager (Parameter Store)
```
Standard parameters: Free
Advanced parameters: 10

Pricing:
  - Advanced: 10 × $0.05 = $0.50

Total: $0.50/month
```

#### Elastic Container Registry (ECR)
```
Storage: 50 GB
Data transfer: 100 GB

Pricing:
  - Storage: 50 × $0.10 = $5
  - Data transfer: Free (within same region)

Total: $5/month
```

#### SQS
```
Requests: 100M/month

Pricing:
  - Standard queue: 100M × $0.40/million = $40
  - First 1M requests free

Total: $40/month
```

#### SNS
```
Requests: 10M/month
Email notifications: 1,000

Pricing:
  - Requests: 10M × $0.50/million = $5
  - Email: 1,000 × $0 (first 1,000 free) = $0

Total: $5/month
```

**Additional Services Subtotal**: $72/month

---

## Total Cost Summary (Per Region)

| Category | Monthly Cost |
|----------|--------------|
| Compute | $1,606 |
| Database | $1,640 |
| Storage | $936 |
| Network & CDN | $2,956 |
| Security | $372 |
| Monitoring | $437 |
| Additional Services | $72 |
| **Total per Region** | **$8,019** |
| **CloudFront (Shared)** | **$2,070** |
| **Route 53 (Shared)** | **$22** |
| **Grand Total per Region** | **$10,111** |

### Multi-Region Total
```
US Region: $10,111
Saudi Region: $10,111
Total: $20,222/month

Annual: $242,664
```

---

## Cost Optimization Strategies

### 1. Reserved Instances & Savings Plans (40-60% savings)

```yaml
Strategy:
  EKS Nodes:
    - 3-Year Compute Savings Plan: 60% discount
    - Coverage: 80% of base capacity
    - Savings: $1,053 → $421 = $632/month saved
  
  RDS:
    - 3-Year Reserved Instances: 60% discount
    - Savings: $1,062 → $425 = $637/month saved
  
  ElastiCache:
    - 1-Year Reserved Instances: 30% discount
    - Savings: $346 → $242 = $104/month saved

Total Savings: $1,373/month per region = $2,746/month
Annual Savings: $32,952
```

### 2. Spot Instances for Video Processing (70-90% savings)

```yaml
Current Cost: $379/month
With 70% Spot: $156/month (Spot) + $223/month (On-Demand)
With 90% Spot: $92/month (Spot) + $67/month (On-Demand) = $159/month

Savings: $379 - $159 = $220/month per region = $440/month
Annual Savings: $5,280
```

### 3. S3 Intelligent-Tiering (30-50% savings)

```yaml
Current S3 Cost (Regional): $78/month

With Intelligent-Tiering automatic optimization:
  - Frequent Access (20%): 1 TB × $0.023 = $23
  - Infrequent Access (50%): 2.5 TB × $0.0125 = $31
  - Archive (30%): 1.5 TB × $0.004 = $6
  Total: $60/month

Savings: $78 - $60 = $18/month per region = $36/month
Annual Savings: $432
```

### 4. CloudFront Optimization (20-30% savings)

```yaml
Current Cost: $4,140/month

Optimization Strategies:
  1. Increase cache hit rate to 95% (currently 85%)
     - Reduces origin requests by 50%
     - Savings on data transfer: $400/month
  
  2. Compress assets (gzip/brotli)
     - Reduces data transfer by 60%
     - Savings: $1,900/month
  
  3. CloudFront committed usage discount
     - 10% additional discount
     - Savings: $184/month

Optimized Cost: $4,140 - $2,484 = $1,656/month
Savings: $2,484/month
Annual Savings: $29,808
```

### 5. Right-Sizing (15-20% savings)

```yaml
Monthly Review Process:
  - Analyze CloudWatch metrics
  - Identify underutilized resources
  - Downsize instances

Example:
  EKS Nodes: t3.xlarge → t3.large (50% cost reduction for underutilized nodes)
  Savings: $200/month per region = $400/month
  
  RDS: db.r6g.2xlarge → db.r6g.xlarge (if CPU < 50%)
  Savings: $500/month per region = $1,000/month

Total Savings: $1,400/month
Annual Savings: $16,800
```

### 6. Data Transfer Optimization

```yaml
Strategies:
  1. VPC Endpoints (avoid NAT Gateway charges)
     Current NAT cost: $234/month
     With VPC Endpoints: $27/month
     Savings: $207/month per region = $414/month
  
  2. CloudFront Regional Edge Caches
     Reduces origin requests by 80%
     Savings on ALB data transfer: $150/month
  
  3. Direct Connect (if data transfer > 10 TB/month)
     Cost: $0.02/GB (vs $0.09/GB internet)
     Potential savings: 78%

Total Savings: $564/month per region = $1,128/month
Annual Savings: $13,536
```

### 7. Lifecycle Policies (10-15% savings)

```yaml
S3 Lifecycle Rules:
  - Videos not accessed in 90 days → Glacier Instant Retrieval
    Storage cost: $0.004/GB (vs $0.023/GB)
    Estimated: 3 TB affected
    Savings: 3,000 GB × ($0.023 - $0.004) = $57/month
  
  - Videos not accessed in 365 days → Glacier Deep Archive
    Storage cost: $0.00099/GB
    Estimated: 2 TB affected
    Savings: 2,000 GB × ($0.023 - $0.00099) = $44/month

Total Savings: $101/month per region = $202/month
Annual Savings: $2,424
```

### 8. Database Optimization

```yaml
Strategies:
  1. Connection Pooling (RDS Proxy)
     Reduces connection overhead
     Enables smaller instance size
     Savings: $200/month per region
  
  2. Read Replicas (horizontal scaling)
     Cheaper than vertical scaling
     Better performance distribution
  
  3. Query Optimization
     - Indexing strategies
     - Caching frequently accessed data
     - Reduces IOPS requirements
     Savings: $100/month per region

Total Savings: $300/month per region = $600/month
Annual Savings: $7,200
```

---

## Optimized Cost Summary

| Optimization Strategy | Monthly Savings | Annual Savings |
|----------------------|-----------------|----------------|
| Reserved Instances & Savings Plans | $2,746 | $32,952 |
| Spot Instances | $440 | $5,280 |
| S3 Intelligent-Tiering | $36 | $432 |
| CloudFront Optimization | $2,484 | $29,808 |
| Right-Sizing | $1,400 | $16,800 |
| Data Transfer Optimization | $1,128 | $13,536 |
| Lifecycle Policies | $202 | $2,424 |
| Database Optimization | $600 | $7,200 |
| **Total Savings** | **$9,036** | **$108,432** |

### Final Cost Projection

```
Original Cost: $20,222/month ($242,664/year)
Optimized Cost: $11,186/month ($134,232/year)

Total Savings: 45% reduction
```

---

## Cost Allocation by Department

```yaml
Engineering (Infrastructure): 60%
  - $6,712/month
  - Compute, Database, Storage

Product (Features & Content): 25%
  - $2,797/month
  - CDN, Video Processing

Security & Compliance: 10%
  - $1,119/month
  - WAF, GuardDuty, Encryption

Operations & Monitoring: 5%
  - $559/month
  - Monitoring, Logging, Alerting
```

---

## Cost per User Metrics

```yaml
Assumptions:
  - 100,000 active users/month
  - 500,000 video views/month
  - 10 TB data transfer/month

Calculations:
  Infrastructure Cost: $11,186/month
  Cost per User: $11,186 / 100,000 = $0.11/user/month
  Cost per Video View: $11,186 / 500,000 = $0.022/view

Benchmark:
  Industry Average: $0.50 - $1.50/user/month
  KnowledgeCity: $0.11/user/month ✅ (78% below industry average)
```

---

## Scaling Cost Projections

### Growth Scenario 1: 10x Users (1M users)

```yaml
Changed Components:
  - EKS Nodes: 12 → 60 nodes
  - RDS: db.r6g.2xlarge → db.r6g.8xlarge
  - ElastiCache: 3 → 9 nodes
  - CloudFront: 50 TB → 200 TB

Estimated Cost: $45,000/month
Cost per User: $0.045/user/month (scale efficiency)
```

### Growth Scenario 2: New Region (Europe)

```yaml
Additional Cost per Region: $11,186/month
Europe Region: $11,186/month

Total (3 regions): $33,558/month
```

### Growth Scenario 3: Premium Features

```yaml
Additional Services:
  - AI-powered recommendations: $500/month
  - Real-time collaboration: $800/month
  - Advanced analytics: $400/month

Total Additional: $1,700/month
New Total: $12,886/month
```

---

## Budget Monitoring & Alerts

### AWS Budget Configuration

```yaml
Monthly Budget: $12,000 per region
Alerts:
  - 80% threshold: Email to FinOps team
  - 100% threshold: Email to CTO + FinOps
  - 120% threshold: PagerDuty alert + Auto-scale restriction

Cost Anomaly Detection:
  - Alert if daily cost exceeds 150% of 7-day average
  - Automated investigation playbook
```

### Cost Optimization Dashboard (Grafana)

```yaml
Metrics to Track:
  - Daily spend by service
  - Cost per user trend
  - Savings plan utilization (target: >85%)
  - Spot instance coverage (target: >70%)
  - S3 storage tier distribution
  - CloudFront cache hit rate (target: >95%)
  - Reserved instance expiration dates
  - Right-sizing recommendations count
```

---

## ROI Analysis

### Investment
```
Infrastructure Setup: $50,000 (one-time)
Monthly Operating Cost: $11,186
Annual Operating Cost: $134,232
```

### Revenue Projection (Example)
```
Users: 100,000
Subscription: $29/month
Monthly Revenue: $2,900,000
Annual Revenue: $34,800,000

Infrastructure as % of Revenue: 0.39% ✅
```

### Break-Even Analysis
```
If charging $1/user/month:
Required users to break even: 11,186 users

Current users: 100,000
Profit margin on infrastructure: 89% ✅
```

---

## Conclusion

The proposed architecture delivers:
- **45% cost savings** through optimization strategies
- **$0.11 cost per user** (78% below industry average)
- **Scalable** infrastructure (10x growth = only 4x cost increase)
- **Predictable** costs through reserved instances and savings plans
- **Optimized** for performance and cost balance

**Recommendation**: Implement all optimization strategies from Day 1 for maximum savings.

---

**Document Version**: 1.0  
**Last Updated**: November 12, 2025  
**Next Review**: Monthly (ongoing optimization)

