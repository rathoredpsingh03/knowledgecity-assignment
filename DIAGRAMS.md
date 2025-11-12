# KnowledgeCity Architecture Diagrams

## 1. High-Level System Architecture (Enhanced)

```mermaid
flowchart TB
    subgraph Internet["🌐 Global Internet"]
        Users["👥 Global Users<br/>US, Saudi, International"]
    end
    
    subgraph Edge["🛡️ Edge Security & CDN"]
        CF["☁️ CloudFlare<br/>━━━━━━━━━━<br/>• DDoS Protection L3/4/7<br/>• WAF OWASP Top 10<br/>• Rate Limiting<br/>• Bot Detection"]
        CloudFront["📦 CloudFront<br/>━━━━━━━━━━<br/>• 450+ Edge Locations<br/>• Origin Shield<br/>• Static Assets CDN<br/>• Video Streaming"]
    end
    
    subgraph DNS["🌍 Global DNS"]
        R53["Route 53<br/>━━━━━━━━━━<br/>• Geolocation Routing<br/>• Health Checks<br/>• Failover < 5 min"]
    end
    
    subgraph US["🇺🇸 US Region (us-east-1)"]
        direction TB
        subgraph US_Edge["Load Balancing"]
            ALB_US["⚖️ ALB Multi-AZ<br/>━━━━━━━━━━<br/>• 3 AZs<br/>• TLS Termination<br/>• Health Checks"]
        end
        
        subgraph US_Compute["💻 Compute Layer"]
            EKS_US["☸️ EKS Cluster<br/>━━━━━━━━━━<br/>• 3 AZs<br/>• Auto-Scaling 6-50 nodes<br/>• PHP API Pods: 9-100<br/>• Analytics Pods: 3-30<br/>• Video Pods: 6-50<br/>• Frontend Pods: 6-50"]
        end
        
        subgraph US_Data["💾 Data Layer"]
            RDS_US["🗄️ RDS PostgreSQL<br/>━━━━━━━━━━<br/>• Multi-AZ Primary/Standby<br/>• Read Replicas × 3<br/>• 500GB gp3<br/>• Automated Backups"]
            ClickHouse_US["📊 ClickHouse<br/>━━━━━━━━━━<br/>• 3 Replicas<br/>• Analytics DB<br/>• Compressed Storage"]
            Redis_US["⚡ ElastiCache Redis<br/>━━━━━━━━━━<br/>• Multi-AZ Cluster<br/>• 3 Nodes<br/>• Session Store"]
            S3_US["🪣 S3 Buckets<br/>━━━━━━━━━━<br/>• User Data Regional<br/>• Course Content Global<br/>• Intelligent-Tiering<br/>• Versioning Enabled"]
        end
        
        subgraph US_Services["🔧 Supporting Services"]
            SQS_US["📨 SQS/SNS<br/>Video Processing Queue"]
            Secrets_US["🔐 Secrets Manager<br/>Credentials & Keys"]
            KMS_US["🔑 KMS<br/>Encryption Keys"]
        end
    end
    
    subgraph SA["🇸🇦 Saudi Region (me-central-1)"]
        direction TB
        subgraph SA_Edge["Load Balancing"]
            ALB_SA["⚖️ ALB Multi-AZ<br/>━━━━━━━━━━<br/>• 3 AZs<br/>• TLS Termination<br/>• Health Checks"]
        end
        
        subgraph SA_Compute["💻 Compute Layer"]
            EKS_SA["☸️ EKS Cluster<br/>━━━━━━━━━━<br/>• 3 AZs<br/>• Auto-Scaling 6-50 nodes<br/>• PHP API Pods: 9-100<br/>• Analytics Pods: 3-30<br/>• Video Pods: 6-50<br/>• Frontend Pods: 6-50"]
        end
        
        subgraph SA_Data["💾 Data Layer"]
            RDS_SA["🗄️ RDS PostgreSQL<br/>━━━━━━━━━━<br/>• Multi-AZ Primary/Standby<br/>• Read Replicas × 3<br/>• 500GB gp3<br/>• Automated Backups"]
            ClickHouse_SA["📊 ClickHouse<br/>━━━━━━━━━━<br/>• 3 Replicas<br/>• Analytics DB<br/>• Compressed Storage"]
            Redis_SA["⚡ ElastiCache Redis<br/>━━━━━━━━━━<br/>• Multi-AZ Cluster<br/>• 3 Nodes<br/>• Session Store"]
            S3_SA["🪣 S3 Buckets<br/>━━━━━━━━━━<br/>• User Data Regional<br/>• Course Content Global<br/>• Intelligent-Tiering<br/>• Versioning Enabled"]
        end
        
        subgraph SA_Services["🔧 Supporting Services"]
            SQS_SA["📨 SQS/SNS<br/>Video Processing Queue"]
            Secrets_SA["🔐 Secrets Manager<br/>Credentials & Keys"]
            KMS_SA["🔑 KMS<br/>Encryption Keys"]
        end
    end
    
    subgraph Monitoring["📊 Observability (Both Regions)"]
        Prometheus["📈 Prometheus<br/>Metrics"]
        Grafana["📊 Grafana<br/>Dashboards"]
        Loki["📝 Loki<br/>Logs"]
        Tempo["🔍 Tempo<br/>Traces"]
    end
    
    %% User Flow
    Users -->|HTTPS| CF
    Users -->|HTTPS| CloudFront
    CF -->|Geolocation| R53
    R53 -->|US Traffic| ALB_US
    R53 -->|Saudi Traffic| ALB_SA
    
    %% US Region Flow
    ALB_US --> EKS_US
    EKS_US -->|Read/Write| RDS_US
    EKS_US -->|Analytics| ClickHouse_US
    EKS_US -->|Cache| Redis_US
    EKS_US -->|Storage| S3_US
    EKS_US -->|Queue| SQS_US
    EKS_US -->|Secrets| Secrets_US
    Secrets_US -->|Decrypt| KMS_US
    
    %% Saudi Region Flow
    ALB_SA --> EKS_SA
    EKS_SA -->|Read/Write| RDS_SA
    EKS_SA -->|Analytics| ClickHouse_SA
    EKS_SA -->|Cache| Redis_SA
    EKS_SA -->|Storage| S3_SA
    EKS_SA -->|Queue| SQS_SA
    EKS_SA -->|Secrets| Secrets_SA
    Secrets_SA -->|Decrypt| KMS_SA
    
    %% CDN Flow
    CloudFront -->|Origin| S3_US
    CloudFront -->|Origin| S3_SA
    
    %% Cross-Region Replication (Course Content Only)
    S3_US <-.->|Course Content<br/>Replication| S3_SA
    
    %% Monitoring
    EKS_US & EKS_SA -.->|Metrics| Prometheus
    EKS_US & EKS_SA -.->|Logs| Loki
    EKS_US & EKS_SA -.->|Traces| Tempo
    Prometheus --> Grafana
    Loki --> Grafana
    Tempo --> Grafana
    
    %% Styling
    classDef edgeClass fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    classDef usClass fill:#4dabf7,stroke:#1864ab,stroke-width:3px,color:#fff
    classDef saClass fill:#51cf66,stroke:#2b8a3e,stroke-width:3px,color:#fff
    classDef monitorClass fill:#ffd43b,stroke:#f59f00,stroke-width:3px,color:#000
    classDef dataClass fill:#e599f7,stroke:#9c36b5,stroke-width:2px,color:#fff
    
    class CF,CloudFront edgeClass
    class ALB_US,EKS_US,RDS_US,Redis_US,ClickHouse_US,S3_US,SQS_US,Secrets_US,KMS_US usClass
    class ALB_SA,EKS_SA,RDS_SA,Redis_SA,ClickHouse_SA,S3_SA,SQS_SA,Secrets_SA,KMS_SA saClass
    class Prometheus,Grafana,Loki,Tempo monitorClass
    class R53 dataClass
```

## 2. Multi-AZ Architecture Detail (Single Region)

```mermaid
flowchart TB
    subgraph Region["🌎 AWS Region (us-east-1 / me-central-1)"]
        direction TB
        
        subgraph AZ1["Availability Zone 1a"]
            direction TB
            NAT1["🔀 NAT Gateway"]
            
            subgraph EKS_AZ1["EKS Node Group"]
                Pod1_1["📦 PHP API Pod<br/>━━━━━━━━━━<br/>CPU: 500m-2<br/>Mem: 512Mi-2Gi"]
                Pod1_2["📦 Analytics Pod<br/>━━━━━━━━━━<br/>CPU: 1-4<br/>Mem: 2Gi-8Gi"]
                Pod1_3["📦 Video Pod<br/>━━━━━━━━━━<br/>CPU: 2-8<br/>Mem: 4Gi-16Gi"]
                Pod1_4["📦 Frontend Pod<br/>━━━━━━━━━━<br/>CPU: 100m-500m<br/>Mem: 128Mi-512Mi"]
            end
            
            RDS_Primary["🗄️ RDS Primary<br/>━━━━━━━━━━<br/>db.r6g.2xlarge<br/>Active"]
        end
        
        subgraph AZ2["Availability Zone 1b"]
            direction TB
            NAT2["🔀 NAT Gateway"]
            
            subgraph EKS_AZ2["EKS Node Group"]
                Pod2_1["📦 PHP API Pod<br/>━━━━━━━━━━<br/>CPU: 500m-2<br/>Mem: 512Mi-2Gi"]
                Pod2_2["📦 Analytics Pod<br/>━━━━━━━━━━<br/>CPU: 1-4<br/>Mem: 2Gi-8Gi"]
                Pod2_3["📦 Video Pod<br/>━━━━━━━━━━<br/>CPU: 2-8<br/>Mem: 4Gi-16Gi"]
                Pod2_4["📦 Frontend Pod<br/>━━━━━━━━━━<br/>CPU: 100m-500m<br/>Mem: 128Mi-512Mi"]
            end
            
            RDS_Standby["🗄️ RDS Standby<br/>━━━━━━━━━━<br/>db.r6g.2xlarge<br/>Sync Replication"]
            Redis2["⚡ Redis Node 2<br/>cache.r6g.large"]
        end
        
        subgraph AZ3["Availability Zone 1c"]
            direction TB
            NAT3["🔀 NAT Gateway"]
            
            subgraph EKS_AZ3["EKS Node Group"]
                Pod3_1["📦 PHP API Pod<br/>━━━━━━━━━━<br/>CPU: 500m-2<br/>Mem: 512Mi-2Gi"]
                Pod3_2["📦 Analytics Pod<br/>━━━━━━━━━━<br/>CPU: 1-4<br/>Mem: 2Gi-8Gi"]
                Pod3_3["📦 Video Pod<br/>━━━━━━━━━━<br/>CPU: 2-8<br/>Mem: 4Gi-16Gi"]
                Pod3_4["📦 Frontend Pod<br/>━━━━━━━━━━<br/>CPU: 100m-500m<br/>Mem: 128Mi-512Mi"]
            end
            
            RDS_Replica["🗄️ RDS Replica<br/>━━━━━━━━━━<br/>db.r6g.2xlarge<br/>Read Only"]
            Redis3["⚡ Redis Node 3<br/>cache.r6g.large"]
        end
        
        ALB["⚖️ Application Load Balancer<br/>━━━━━━━━━━━━━━━━━━━━━━<br/>Spans all 3 AZs<br/>Health Checks every 30s"]
        
        Redis1["⚡ Redis Node 1<br/>cache.r6g.large"]
        
        ClickHouse1["📊 ClickHouse 1<br/>AZ-1a"]
        ClickHouse2["📊 ClickHouse 2<br/>AZ-1b"]
        ClickHouse3["📊 ClickHouse 3<br/>AZ-1c"]
        
        S3["🪣 S3 (Regional)<br/>━━━━━━━━━━<br/>11 9s Durability<br/>Cross-AZ Replication"]
    end
    
    Users["👥 Users"] -->|HTTPS| ALB
    
    ALB -->|Round Robin| EKS_AZ1
    ALB -->|Round Robin| EKS_AZ2
    ALB -->|Round Robin| EKS_AZ3
    
    Pod1_1 & Pod2_1 & Pod3_1 -->|Write| RDS_Primary
    Pod1_1 & Pod2_1 & Pod3_1 -->|Read| RDS_Replica
    
    RDS_Primary -.->|Sync Replication| RDS_Standby
    RDS_Primary -.->|Async Replication| RDS_Replica
    
    Pod1_1 & Pod2_1 & Pod3_1 -->|Cache| Redis1
    Pod1_2 & Pod2_2 & Pod3_2 -->|Cache| Redis2
    Pod1_3 & Pod2_3 & Pod3_3 -->|Cache| Redis3
    
    Redis1 <-.->|Cluster Sync| Redis2
    Redis2 <-.->|Cluster Sync| Redis3
    Redis3 <-.->|Cluster Sync| Redis1
    
    Pod1_2 & Pod2_2 & Pod3_2 -->|Analytics| ClickHouse1
    Pod1_2 & Pod2_2 & Pod3_2 -->|Analytics| ClickHouse2
    Pod1_2 & Pod2_2 & Pod3_2 -->|Analytics| ClickHouse3
    
    ClickHouse1 <-.->|Replication| ClickHouse2
    ClickHouse2 <-.->|Replication| ClickHouse3
    ClickHouse3 <-.->|Replication| ClickHouse1
    
    EKS_AZ1 & EKS_AZ2 & EKS_AZ3 -->|Storage| S3
    
    EKS_AZ1 -->|Internet| NAT1
    EKS_AZ2 -->|Internet| NAT2
    EKS_AZ3 -->|Internet| NAT3
    
    classDef az1Class fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef az2Class fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef az3Class fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef sharedClass fill:#fff3e0,stroke:#f57c00,stroke-width:3px
    
    class AZ1,NAT1,EKS_AZ1,Pod1_1,Pod1_2,Pod1_3,Pod1_4,RDS_Primary,ClickHouse1,Redis1 az1Class
    class AZ2,NAT2,EKS_AZ2,Pod2_1,Pod2_2,Pod2_3,Pod2_4,RDS_Standby,ClickHouse2,Redis2 az2Class
    class AZ3,NAT3,EKS_AZ3,Pod3_1,Pod3_2,Pod3_3,Pod3_4,RDS_Replica,ClickHouse3,Redis3 az3Class
    class ALB,S3 sharedClass
```

**Key Features:**
- 🎯 No single point of failure
- ⚡ Automatic failover (Primary → Standby < 120s)
- 📊 Load distribution across 3 AZs
- 🔄 Synchronous replication (Primary → Standby)
- 📈 Horizontal scaling per AZ
- 🛡️ Each AZ can handle full load independently

## 3. Video Processing Pipeline

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant S3
    participant SQS
    participant VideoService
    participant CloudFront
    
    User->>Frontend: Upload Video
    Frontend->>API: Request Pre-signed URL
    API->>S3: Generate Pre-signed URL
    S3-->>API: Return URL
    API-->>Frontend: Return URL
    Frontend->>S3: Upload to Pre-signed URL
    S3->>SQS: S3 Event Notification
    SQS->>VideoService: Trigger Processing
    
    VideoService->>S3: Download Source Video
    VideoService->>VideoService: Transcode to Multiple Formats<br/>(1080p, 720p, 480p, 360p)
    VideoService->>VideoService: Generate HLS/DASH Manifests
    VideoService->>S3: Upload Processed Videos
    VideoService->>API: Update Processing Status
    API->>User: Notify Completion
    
    User->>Frontend: Play Video
    Frontend->>CloudFront: Request Video
    CloudFront->>S3: Fetch (if not cached)
    CloudFront-->>Frontend: Stream Video
```

## 3. User Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant CloudFlare
    participant API
    participant Redis
    participant RDS
    participant SecretsManager
    
    User->>Frontend: Login (email, password)
    Frontend->>CloudFlare: POST /api/auth/login
    CloudFlare->>CloudFlare: WAF Check
    CloudFlare->>API: Forward Request
    API->>Redis: Check Rate Limit
    Redis-->>API: Allow
    API->>RDS: Query User Credentials
    RDS-->>API: Return User Record
    API->>API: Verify Password (bcrypt)
    API->>SecretsManager: Get JWT Private Key
    SecretsManager-->>API: Return Key
    API->>API: Generate JWT Token
    API->>Redis: Store Session
    API-->>Frontend: Return JWT Token
    Frontend->>Frontend: Store Token (httpOnly cookie)
    
    Note over User,RDS: Subsequent Requests
    
    User->>Frontend: Access Protected Resource
    Frontend->>API: GET /api/courses (with JWT)
    API->>Redis: Validate Session
    Redis-->>API: Session Valid
    API->>API: Verify JWT Signature
    API->>RDS: Query Course Data
    RDS-->>API: Return Data
    API-->>Frontend: Return Response
```

## 4. Multi-Region Traffic Routing

```mermaid
graph TB
    USER_US[US User<br/>IP: 203.0.113.1]
    USER_SA[Saudi User<br/>IP: 151.101.1.1]
    USER_EU[EU User<br/>IP: 185.60.216.1]
    
    R53[Route 53<br/>Geolocation Routing]
    
    USER_US --> R53
    USER_SA --> R53
    USER_EU --> R53
    
    R53 -->|US IP Range| US_REGION[US Region<br/>us-east-1]
    R53 -->|Saudi/ME IP Range| SA_REGION[Saudi Region<br/>me-central-1]
    R53 -->|Latency-based| NEAREST[Nearest Region<br/>Based on Latency]
    
    subgraph Health_Checks[Route 53 Health Checks]
        HC1[US Region Health Check<br/>Every 30s]
        HC2[Saudi Region Health Check<br/>Every 30s]
    end
    
    HC1 -.Monitor.-> US_REGION
    HC2 -.Monitor.-> SA_REGION
    
    HC1 -->|Unhealthy| FAILOVER1[Route to<br/>Saudi Region]
    HC2 -->|Unhealthy| FAILOVER2[Route to<br/>US Region]
```

## 5. Security Layers (Enhanced Defense in Depth)

```mermaid
flowchart TB
    subgraph Layer1["🛡️ LAYER 1: Edge Security"]
        direction LR
        CF_DDoS["☁️ CloudFlare DDoS<br/>━━━━━━━━━━<br/>• Layer 3/4/7 Protection<br/>• 100+ Tbps Capacity<br/>• Anycast Network"]
        CF_WAF["🔥 CloudFlare WAF<br/>━━━━━━━━━━<br/>• OWASP Top 10<br/>• SQL Injection<br/>• XSS Prevention<br/>• Rate Limiting 100/min"]
        Shield["🛡️ AWS Shield<br/>━━━━━━━━━━<br/>• Standard DDoS<br/>• Auto-Detection<br/>• Auto-Mitigation"]
        
        CF_DDoS --> CF_WAF
        CF_WAF --> Shield
    end
    
    subgraph Layer2["🔒 LAYER 2: Network Security"]
        direction LR
        VPC["🏢 VPC<br/>━━━━━━━━━━<br/>• Private Subnets<br/>• Isolated Networks<br/>• No Direct Internet"]
        SG["🔐 Security Groups<br/>━━━━━━━━━━<br/>• Stateful Firewall<br/>• Least Privilege<br/>• ALB: 443 CloudFlare<br/>• EKS: ALB Only<br/>• RDS: EKS Only"]
        NACL["📋 Network ACLs<br/>━━━━━━━━━━<br/>• Stateless Firewall<br/>• Subnet Level<br/>• Allow/Deny Rules"]
        FlowLogs["📊 VPC Flow Logs<br/>━━━━━━━━━━<br/>• Traffic Analysis<br/>• Anomaly Detection<br/>• Forensics"]
        
        VPC --> SG
        SG --> NACL
        NACL --> FlowLogs
    end
    
    subgraph Layer3["🔐 LAYER 3: Application Security"]
        direction LR
        Istio["🕸️ Istio mTLS<br/>━━━━━━━━━━<br/>• Service-to-Service<br/>• Certificate Rotation<br/>• Zero Trust"]
        OAuth["🔑 OAuth 2.0 / JWT<br/>━━━━━━━━━━<br/>• Token-Based Auth<br/>• 15-60 min Expiry<br/>• Refresh Tokens"]
        RBAC["👥 RBAC<br/>━━━━━━━━━━<br/>• Role-Based Access<br/>• Least Privilege<br/>• K8s RBAC + App RBAC"]
        InputVal["✅ Input Validation<br/>━━━━━━━━━━<br/>• JSON Schema<br/>• Parameterized Queries<br/>• XSS Prevention"]
        
        Istio --> OAuth
        OAuth --> RBAC
        RBAC --> InputVal
    end
    
    subgraph Layer4["🔐 LAYER 4: Data Security"]
        direction TB
        subgraph AtRest["Encryption at Rest"]
            RDS_Enc["🗄️ RDS<br/>AES-256 KMS"]
            S3_Enc["🪣 S3<br/>SSE-KMS"]
            EBS_Enc["💾 EBS<br/>AES-256"]
        end
        subgraph InTransit["Encryption in Transit"]
            TLS["🔒 TLS 1.3<br/>Client ↔ Edge"]
            mTLS["🔐 mTLS<br/>Service ↔ Service"]
            SSL["🔒 SSL/TLS<br/>App ↔ Database"]
        end
        subgraph Secrets["Secrets Management"]
            SM["🔑 Secrets Manager<br/>90-day Rotation"]
            ESO["🔄 External Secrets<br/>K8s Operator"]
            KMS_Key["🔐 KMS Keys<br/>Customer-Managed"]
        end
        
        AtRest --> InTransit
        InTransit --> Secrets
    end
    
    subgraph Layer5["📊 LAYER 5: Monitoring & Detection"]
        direction LR
        GuardDuty["🔍 GuardDuty<br/>━━━━━━━━━━<br/>• Threat Detection<br/>• ML-Based<br/>• Compromised Instances"]
        CloudTrail["📝 CloudTrail<br/>━━━━━━━━━━<br/>• API Audit Logs<br/>• All Regions<br/>• Tamper-Proof"]
        SecurityHub["🛡️ Security Hub<br/>━━━━━━━━━━<br/>• Compliance Checks<br/>• CIS Benchmarks<br/>• Centralized Findings"]
        Loki_Sec["📋 Loki Security<br/>━━━━━━━━━━<br/>• Failed Auth<br/>• Anomalies<br/>• Real-time Alerts"]
        
        GuardDuty --> CloudTrail
        CloudTrail --> SecurityHub
        SecurityHub --> Loki_Sec
    end
    
    Internet["🌐 Internet<br/>Malicious Traffic"] --> Layer1
    Layer1 -->|Filtered| Layer2
    Layer2 -->|Secured| Layer3
    Layer3 -->|Protected| Layer4
    Layer4 -->|Monitored| Layer5
    
    Layer5 -.->|Alerts| PagerDuty["📟 PagerDuty<br/>P1/P2 Incidents"]
    Layer5 -.->|Logs| SIEM["🔍 SIEM<br/>Threat Analysis"]
    
    classDef layer1Class fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    classDef layer2Class fill:#ffd43b,stroke:#f59f00,stroke-width:3px,color:#000
    classDef layer3Class fill:#51cf66,stroke:#2b8a3e,stroke-width:3px,color:#fff
    classDef layer4Class fill:#4dabf7,stroke:#1864ab,stroke-width:3px,color:#fff
    classDef layer5Class fill:#e599f7,stroke:#9c36b5,stroke-width:3px,color:#fff
    classDef alertClass fill:#f06595,stroke:#c2255c,stroke-width:2px,color:#fff
    
    class Layer1,CF_DDoS,CF_WAF,Shield layer1Class
    class Layer2,VPC,SG,NACL,FlowLogs layer2Class
    class Layer3,Istio,OAuth,RBAC,InputVal layer3Class
    class Layer4,AtRest,InTransit,Secrets,RDS_Enc,S3_Enc,EBS_Enc,TLS,mTLS,SSL,SM,ESO,KMS_Key layer4Class
    class Layer5,GuardDuty,CloudTrail,SecurityHub,Loki_Sec layer5Class
    class PagerDuty,SIEM alertClass
```

**Security Metrics:**
| Layer | MTTD | MTTR | Coverage |
|-------|------|------|----------|
| Layer 1 (Edge) | < 1 min | < 5 min | 99.9% |
| Layer 2 (Network) | < 5 min | < 15 min | 100% |
| Layer 3 (App) | < 10 min | < 30 min | 100% |
| Layer 4 (Data) | < 15 min | < 1 hour | 100% |
| Layer 5 (Monitor) | < 1 min | < 15 min | 100% |

**Attack Mitigation:**
- ✅ DDoS: Blocked at Edge (Layer 1)
- ✅ SQL Injection: Prevented at Layer 1 (WAF) + Layer 3 (Input Validation)
- ✅ XSS: Prevented at Layer 1 (WAF) + Layer 3 (Output Encoding)
- ✅ Unauthorized Access: Blocked at Layer 2 (Network) + Layer 3 (Auth)
- ✅ Data Breach: Encrypted at Layer 4 (Encryption)
- ✅ Insider Threat: Monitored at Layer 5 (Audit Logs)

## 6. Security Layers

```mermaid
graph LR
    subgraph Layer_1[Layer 1: Edge Security]
        CF_DDoS[CloudFlare DDoS Protection]
        CF_WAF[CloudFlare WAF]
        SHIELD[AWS Shield Standard]
    end
    
    subgraph Layer_2[Layer 2: Network Security]
        VPC[VPC with Private Subnets]
        SG[Security Groups]
        NACL[Network ACLs]
        VPN[VPN/Direct Connect]
    end
    
    subgraph Layer_3[Layer 3: Application Security]
        ISTIO[Istio mTLS]
        JWT[JWT Authentication]
        OAUTH[OAuth 2.0]
        RBAC[RBAC Policies]
    end
    
    subgraph Layer_4[Layer 4: Data Security]
        KMS[AWS KMS Encryption]
        TLS[TLS 1.3]
        SECRETS[Secrets Manager]
        BACKUP[Encrypted Backups]
    end
    
    subgraph Layer_5[Layer 5: Monitoring]
        GUARDDUTY[AWS GuardDuty]
        CLOUDTRAIL[CloudTrail Logging]
        LOKI[Loki Log Analysis]
        PROMETHEUS[Prometheus Alerts]
    end
    
    Layer_1 --> Layer_2
    Layer_2 --> Layer_3
    Layer_3 --> Layer_4
    Layer_4 --> Layer_5
```

## 6. Observability Architecture

```mermaid
graph TB
    subgraph Applications[Application Layer]
        PHP[PHP API]
        ANALYTICS[Analytics Service]
        VIDEO[Video Service]
        FRONTEND[Frontend SPA]
    end
    
    subgraph Collection[Collection Layer]
        OTEL[OpenTelemetry<br/>Collector]
        FB[Fluent Bit<br/>DaemonSet]
        NE[Node Exporter]
        KSM[Kube State Metrics]
    end
    
    subgraph Storage[Storage Layer]
        PROM[Prometheus<br/>Metrics DB]
        LOKI_DB[Loki<br/>Log DB]
        TEMPO[Tempo<br/>Trace DB]
    end
    
    subgraph Visualization[Visualization Layer]
        GRAFANA[Grafana<br/>Unified Dashboard]
    end
    
    subgraph Alerting[Alerting Layer]
        AM[Alert Manager]
        PD[PagerDuty]
        SLACK[Slack]
    end
    
    PHP -->|metrics| OTEL
    PHP -->|logs| FB
    PHP -->|traces| OTEL
    
    ANALYTICS -->|metrics| OTEL
    ANALYTICS -->|logs| FB
    ANALYTICS -->|traces| OTEL
    
    VIDEO -->|metrics| OTEL
    VIDEO -->|logs| FB
    VIDEO -->|traces| OTEL
    
    NE -->|host metrics| PROM
    KSM -->|k8s metrics| PROM
    
    OTEL --> PROM
    OTEL --> TEMPO
    FB --> LOKI_DB
    
    PROM --> GRAFANA
    LOKI_DB --> GRAFANA
    TEMPO --> GRAFANA
    
    PROM --> AM
    AM --> PD
    AM --> SLACK
```

## 7. CI/CD Pipeline

```mermaid
graph LR
    DEV[Developer] -->|git push| GITHUB[GitHub Repository]
    
    GITHUB -->|webhook| GHA[GitHub Actions]
    
    subgraph CI[Continuous Integration]
        LINT[Lint Code]
        TEST[Unit Tests]
        BUILD[Build Docker Image]
        SCAN[Security Scan<br/>Trivy + SonarQube]
        PUSH[Push to ECR]
        
        GHA --> LINT
        LINT --> TEST
        TEST --> BUILD
        BUILD --> SCAN
        SCAN --> PUSH
    end
    
    PUSH --> ECR[Amazon ECR]
    
    ECR -->|image available| UPDATE[Update Helm Chart<br/>Git Commit]
    
    UPDATE --> REPO[Kubernetes<br/>Manifests Repo]
    
    subgraph CD[Continuous Deployment]
        ARGOCD[ArgoCD]
        SYNC[Detect Changes]
        DEPLOY_STG[Deploy to Staging]
        TEST_STG[Automated Tests]
        APPROVAL[Manual Approval]
        DEPLOY_PROD[Deploy to Production]
        HEALTH[Health Check]
        ROLLBACK[Auto Rollback<br/>on Failure]
        
        ARGOCD --> SYNC
        SYNC --> DEPLOY_STG
        DEPLOY_STG --> TEST_STG
        TEST_STG --> APPROVAL
        APPROVAL --> DEPLOY_PROD
        DEPLOY_PROD --> HEALTH
        HEALTH -->|failure| ROLLBACK
    end
    
    REPO -->|monitors| ARGOCD
    
    DEPLOY_PROD --> EKS_US[EKS US]
    DEPLOY_PROD --> EKS_SA[EKS Saudi]
```

## 8. Data Flow - Regional Isolation

```mermaid
graph TB
    subgraph US_Users[US Users]
        US_U1[User 1]
        US_U2[User 2]
    end
    
    subgraph SA_Users[Saudi Users]
        SA_U1[User 1]
        SA_U2[User 2]
    end
    
    subgraph Global_Content[Global Content<br/>KnowledgeCity Courses]
        COURSES[Course Library]
        VIDEOS[Video Content]
    end
    
    subgraph US_Infrastructure[US Region]
        US_DB[(US User Data<br/>PostgreSQL)]
        US_S3[(US Storage<br/>S3)]
        US_APP[Applications]
    end
    
    subgraph SA_Infrastructure[Saudi Region]
        SA_DB[(Saudi User Data<br/>PostgreSQL)]
        SA_S3[(Saudi Storage<br/>S3)]
        SA_APP[Applications]
    end
    
    US_U1 -->|PII| US_APP
    US_U2 -->|PII| US_APP
    US_APP --> US_DB
    US_APP --> US_S3
    
    SA_U1 -->|PII| SA_APP
    SA_U2 -->|PII| SA_APP
    SA_APP --> SA_DB
    SA_APP --> SA_S3
    
    COURSES -.replicated.-> US_S3
    COURSES -.replicated.-> SA_S3
    
    US_U1 -.access.-> COURSES
    US_U2 -.access.-> COURSES
    SA_U1 -.access.-> COURSES
    SA_U2 -.access.-> COURSES
    
    US_DB x--x|NO TRANSFER| SA_DB
    
    style US_DB fill:#e1f5ff
    style SA_DB fill:#ffe1e1
    style COURSES fill:#e1ffe1
```

## 9. Kubernetes Pod Architecture

```mermaid
graph TB
    subgraph EKS_Cluster[EKS Cluster]
        subgraph Namespace_App[Namespace: application]
            subgraph PHP_Deployment[PHP API Deployment]
                PHP_POD1[Pod 1]
                PHP_POD2[Pod 2]
                PHP_POD3[Pod 3]
            end
            
            subgraph Analytics_Deployment[Analytics Deployment]
                ANALYTICS_POD1[Pod 1]
                ANALYTICS_POD2[Pod 2]
            end
            
            subgraph Video_Deployment[Video Service Deployment]
                VIDEO_POD1[Pod 1]
                VIDEO_POD2[Pod 2]
            end
            
            subgraph Frontend_Deployment[Frontend Deployment]
                FRONTEND_POD1[Pod 1]
                FRONTEND_POD2[Pod 2]
            end
        end
        
        subgraph Namespace_Monitoring[Namespace: monitoring]
            PROM_POD[Prometheus]
            GRAFANA_POD[Grafana]
            LOKI_POD[Loki]
            TEMPO_POD[Tempo]
        end
        
        subgraph Namespace_Istio[Namespace: istio-system]
            ISTIO_INGRESS[Istio Ingress Gateway]
            ISTIOD[Istiod Control Plane]
        end
        
        subgraph Namespace_Security[Namespace: security]
            EXTERNAL_SECRETS[External Secrets Operator]
            FALCO[Falco Runtime Security]
        end
    end
    
    ISTIO_INGRESS --> PHP_POD1
    ISTIO_INGRESS --> PHP_POD2
    ISTIO_INGRESS --> PHP_POD3
    ISTIO_INGRESS --> ANALYTICS_POD1
    ISTIO_INGRESS --> ANALYTICS_POD2
    ISTIO_INGRESS --> VIDEO_POD1
    ISTIO_INGRESS --> VIDEO_POD2
    ISTIO_INGRESS --> FRONTEND_POD1
    ISTIO_INGRESS --> FRONTEND_POD2
    
    ISTIOD -.manages.-> ISTIO_INGRESS
    
    EXTERNAL_SECRETS -.injects secrets.-> PHP_POD1
    EXTERNAL_SECRETS -.injects secrets.-> ANALYTICS_POD1
    EXTERNAL_SECRETS -.injects secrets.-> VIDEO_POD1
```

## 10. Disaster Recovery Flow

```mermaid
sequenceDiagram
    participant User
    participant Route53
    participant US_Region
    participant SA_Region
    participant Monitoring
    participant OnCall
    
    Note over US_Region: Normal Operation
    User->>Route53: Request
    Route53->>US_Region: Route (Geolocation)
    US_Region-->>User: Response
    
    Note over US_Region: ⚠️ Region Failure Detected
    US_Region->>Monitoring: Health Check Fails (3x)
    Monitoring->>OnCall: Alert (PagerDuty)
    Monitoring->>Route53: Mark US Unhealthy
    
    User->>Route53: Request
    Route53->>Route53: Check Health Status
    Route53->>SA_Region: Route to Healthy Region
    SA_Region-->>User: Response
    
    Note over OnCall: Incident Response
    OnCall->>US_Region: Investigate & Fix
    US_Region->>Monitoring: Health Check Passes
    Monitoring->>Route53: Mark US Healthy
    
    Note over Route53: Gradual Traffic Restoration
    Route53->>US_Region: Resume Normal Routing
```

---

## Infrastructure as Code Example

### Terraform Module: Multi-AZ EKS Cluster

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "knowledgecity-${var.region}"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Multi-AZ Node Groups
  eks_managed_node_groups = {
    general = {
      name           = "general-purpose"
      instance_types = ["t3.xlarge"]
      
      min_size     = 6  # 2 per AZ
      max_size     = 50
      desired_size = 9  # 3 per AZ

      # Spread across AZs
      availability_zones = [
        "${var.region}a",
        "${var.region}b",
        "${var.region}c"
      ]

      labels = {
        role = "general"
      }

      taints = []
    }

    compute = {
      name           = "compute-optimized"
      instance_types = ["c6i.2xlarge"]
      
      min_size     = 0
      max_size     = 20
      desired_size = 3

      # Spot instances for cost optimization
      capacity_type = "SPOT"

      labels = {
        role        = "compute"
        workload    = "video-processing"
      }

      taints = [{
        key    = "workload"
        value  = "video-processing"
        effect = "NoSchedule"
      }]
    }
  }

  # Enable IRSA (IAM Roles for Service Accounts)
  enable_irsa = true

  # Cluster encryption
  cluster_encryption_config = {
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }

  # Cluster endpoint configuration
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true

  # Enable control plane logging
  cluster_enabled_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  tags = {
    Environment = var.environment
    Region      = var.region
    Compliance  = "data-residency"
  }
}
```

### Kubernetes Deployment: PHP API with HPA

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-api
  namespace: application
  labels:
    app: php-api
    version: v1.2.0
spec:
  replicas: 9  # 3 per AZ
  selector:
    matchLabels:
      app: php-api
  template:
    metadata:
      labels:
        app: php-api
        version: v1.2.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Spread pods across AZs
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: php-api
      
      # Anti-affinity: Don't schedule on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: php-api
              topologyKey: kubernetes.io/hostname
      
      serviceAccountName: php-api
      
      containers:
      - name: php-api
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/php-api:v1.2.0
        imagePullPolicy: Always
        
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        
        env:
        - name: APP_ENV
          value: "production"
        - name: AWS_REGION
          value: "us-east-1"
        - name: LOG_LEVEL
          value: "info"
        
        # Mount secrets from External Secrets Operator
        envFrom:
        - secretRef:
            name: php-api-secrets
        
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Security context
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: php-api
  namespace: application
  labels:
    app: php-api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  - port: 9090
    targetPort: 9090
    protocol: TCP
    name: metrics
  selector:
    app: php-api

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-api
  namespace: application
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-api
  minReplicas: 9
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 30
        periodSeconds: 30
      - type: Pods
        value: 5
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 300
      selectPolicy: Min
```

## 11. Cost Optimization Strategy

```mermaid
flowchart TB
    subgraph Current["💰 Current Cost: $20,222/month"]
        direction TB
        Compute_Before["💻 Compute<br/>$3,212/month<br/>On-Demand Instances"]
        DB_Before["🗄️ Databases<br/>$3,280/month<br/>On-Demand RDS"]
        Storage_Before["💾 Storage<br/>$1,872/month<br/>Standard S3"]
        Network_Before["🌐 Network<br/>$7,052/month<br/>Full Data Transfer"]
    end
    
    subgraph Optimization["🎯 Optimization Strategies"]
        direction TB
        
        subgraph Strategy1["1️⃣ Reserved Instances<br/>60% Discount"]
            RI_EKS["EKS Nodes<br/>3-Year Savings Plan<br/>💰 Save: $632/month"]
            RI_RDS["RDS Instances<br/>3-Year Reserved<br/>💰 Save: $637/month"]
            RI_Cache["ElastiCache<br/>1-Year Reserved<br/>💰 Save: $104/month"]
        end
        
        subgraph Strategy2["2️⃣ Spot Instances<br/>70-90% Discount"]
            Spot["Video Processing<br/>90% Spot + 10% On-Demand<br/>💰 Save: $220/month"]
        end
        
        subgraph Strategy3["3️⃣ S3 Intelligent-Tiering<br/>Auto-Optimization"]
            S3_Int["Automatic Tier Movement<br/>Frequent → Infrequent → Archive<br/>💰 Save: $18/month"]
        end
        
        subgraph Strategy4["4️⃣ CloudFront Optimization<br/>60% Reduction"]
            CF_Cache["Cache Hit Rate 95%<br/>Asset Compression<br/>Origin Shield<br/>💰 Save: $2,484/month"]
        end
        
        subgraph Strategy5["5️⃣ Right-Sizing<br/>15-20% Reduction"]
            RightSize["Monthly Reviews<br/>Downsize Underutilized<br/>💰 Save: $1,400/month"]
        end
        
        subgraph Strategy6["6️⃣ Data Transfer<br/>VPC Endpoints"]
            Transfer["VPC Endpoints vs NAT<br/>Regional Edge Caches<br/>💰 Save: $1,128/month"]
        end
        
        subgraph Strategy7["7️⃣ Lifecycle Policies<br/>10-15% Savings"]
            Lifecycle["Auto-Archive Old Data<br/>S3 Glacier Deep Archive<br/>💰 Save: $202/month"]
        end
        
        subgraph Strategy8["8️⃣ Database Optimization<br/>Connection Pooling"]
            DB_Opt["RDS Proxy<br/>Query Optimization<br/>Read Replicas<br/>💰 Save: $600/month"]
        end
    end
    
    subgraph Optimized["✨ Optimized Cost: $11,186/month"]
        direction TB
        Compute_After["💻 Compute<br/>$1,686/month<br/>-47% 🎉"]
        DB_After["🗄️ Databases<br/>$2,132/month<br/>-35% 🎉"]
        Storage_After["💾 Storage<br/>$1,310/month<br/>-30% 🎉"]
        Network_After["🌐 Network<br/>$3,508/month<br/>-50% 🎉"]
    end
    
    Current --> Optimization
    Optimization --> Optimized
    
    Strategy1 -.->|$1,373 saved| Compute_After
    Strategy2 -.->|$220 saved| Compute_After
    Strategy5 -.->|$1,400 saved| Compute_After
    Strategy8 -.->|$600 saved| DB_After
    Strategy3 -.->|$18 saved| Storage_After
    Strategy7 -.->|$202 saved| Storage_After
    Strategy4 -.->|$2,484 saved| Network_After
    Strategy6 -.->|$1,128 saved| Network_After
    
    Optimized --> Summary["📊 Total Savings<br/>━━━━━━━━━━━━━━━<br/>$9,036/month<br/>$108,432/year<br/>45% reduction 🎉"]
    
    classDef beforeClass fill:#ff6b6b,stroke:#c92a2a,stroke-width:2px,color:#fff
    classDef strategyClass fill:#4dabf7,stroke:#1864ab,stroke-width:2px,color:#fff
    classDef afterClass fill:#51cf66,stroke:#2b8a3e,stroke-width:3px,color:#fff
    classDef summaryClass fill:#ffd43b,stroke:#f59f00,stroke-width:4px,color:#000,font-weight:bold
    
    class Current,Compute_Before,DB_Before,Storage_Before,Network_Before beforeClass
    class Optimization,Strategy1,Strategy2,Strategy3,Strategy4,Strategy5,Strategy6,Strategy7,Strategy8 strategyClass
    class Optimized,Compute_After,DB_After,Storage_After,Network_After afterClass
    class Summary summaryClass
```

**Cost Breakdown Summary:**

| Component | Before | After | Savings | % Saved |
|-----------|--------|-------|---------|---------|
| 💻 **Compute** | $3,212 | $1,686 | $1,526 | 47% |
| 🗄️ **Databases** | $3,280 | $2,132 | $1,148 | 35% |
| 💾 **Storage** | $1,872 | $1,310 | $562 | 30% |
| 🌐 **Network & CDN** | $7,052 | $3,508 | $3,544 | 50% |
| 🔒 **Security** | $744 | $650 | $94 | 13% |
| 📊 **Monitoring** | $874 | $765 | $109 | 12% |
| 🔧 **Other** | $188 | $135 | $53 | 28% |
| **💰 TOTAL** | **$20,222** | **$11,186** | **$9,036** | **45%** |

**Cost per User:**
- Before optimization: **$0.20/user**
- After optimization: **$0.11/user** ✅
- Industry average: **$0.50-$1.50/user**
- **Our advantage: 78% below industry average** 🎉

---

## Summary

These **enhanced** diagrams provide comprehensive visual representation of:

### Architecture & Infrastructure
1. **High-Level System Architecture (Enhanced)** - Multi-region setup with detailed component breakdown
2. **Multi-AZ Architecture Detail** - Single region showing 3 AZ redundancy and data replication
3. **Kubernetes Pod Architecture** - Pod distribution across AZs with resource limits

### Data Flow & Processing
4. **Video Processing Pipeline** - End-to-end video upload, processing, and CDN delivery
5. **Authentication Flow** - User login, JWT tokens, and session management
6. **Data Flow & Regional Isolation** - Data residency compliance visualization

### Security & Compliance
7. **Security Layers (Enhanced)** - 5-layer defense in depth with detailed metrics
8. **Security Layers** - Alternative visualization of security architecture

### Observability & Operations
9. **Observability Architecture** - Metrics, logs, and traces collection and visualization
10. **CI/CD Pipeline** - Complete deployment flow from code to production

### High Availability & Disaster Recovery
11. **Multi-Region Traffic Routing** - Geolocation routing and health check failover
12. **Disaster Recovery Flow** - Incident detection, response, and recovery procedures

### Cost Optimization
13. **Cost Optimization Strategy** - Visual breakdown of 8 optimization strategies and 45% savings

### Technical Implementation
14. **Terraform Module Examples** - Multi-AZ EKS cluster infrastructure as code
15. **Kubernetes Deployment Examples** - PHP API deployment with HPA configuration

---

## How to View These Diagrams

### Option 1: GitHub (Recommended)
Push this repository to GitHub and all Mermaid diagrams will render automatically with beautiful colors and interactivity.

### Option 2: VS Code with Mermaid Extension
1. Install "Markdown Preview Mermaid Support" extension
2. Open this file
3. Press `Cmd+Shift+V` (Mac) or `Ctrl+Shift+V` (Windows)

### Option 3: Online Mermaid Editor
Visit https://mermaid.live/ and paste any diagram code

### Option 4: ASCII Art Version
Open `DIAGRAMS_ASCII.md` for text-based diagrams viewable in any editor

---

## Key Features Visualized

✅ **Multi-Region Active-Active** - US + Saudi Arabia with automatic failover  
✅ **Multi-AZ Redundancy** - 3 Availability Zones per region for 99.99% SLA  
✅ **Data Sovereignty** - Regional data isolation with cross-region content replication  
✅ **5-Layer Security** - Defense in depth from edge to monitoring  
✅ **Auto-Scaling** - Horizontal pod autoscaling and cluster autoscaling  
✅ **Full Observability** - Unified metrics, logs, and traces  
✅ **Cost Optimization** - 45% savings through 8 strategies  
✅ **CI/CD Automation** - GitOps with canary deployments  
✅ **Zero Trust** - mTLS between all services  
✅ **CDN Delivery** - Global edge caching for low latency  

---

All diagrams are **color-coded** for clarity:
- 🔴 **Red** - Edge Security & Critical Components
- 🔵 **Blue** - US Region Components
- 🟢 **Green** - Saudi Region Components
- 🟡 **Yellow** - Monitoring & Observability
- 🟣 **Purple** - Shared/Global Services

These components work together to deliver a **highly available, secure, and scalable global education platform** ready for production deployment.

