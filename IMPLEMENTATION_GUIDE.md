# KnowledgeCity Platform - Implementation Guide

## Phase 1: Foundation (Weeks 1-4)

### Week 1: Initial Setup

#### AWS Account Preparation
```bash
# 1. Create AWS Organizations structure
aws organizations create-organization

# 2. Create accounts for each environment
aws organizations create-account \
  --email knowledgecity-prod@company.com \
  --account-name knowledgecity-production

aws organizations create-account \
  --email knowledgecity-staging@company.com \
  --account-name knowledgecity-staging

# 3. Enable required services in all regions
aws --region us-east-1 ec2 describe-regions --output table
aws --region me-central-1 ec2 describe-regions --output table

# 4. Set up cross-account roles
aws iam create-role \
  --role-name OrganizationAccountAccessRole \
  --assume-role-policy-document file://trust-policy.json
```

#### Terraform Setup
```bash
# Initialize Terraform workspace
mkdir -p infrastructure/terraform
cd infrastructure/terraform

# Create backend configuration
cat > backend.tf <<EOF
terraform {
  backend "s3" {
    bucket         = "knowledgecity-terraform-state"
    key            = "global/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
EOF

# Initialize Terraform
terraform init

# Create workspaces for each region
terraform workspace new us-east-1
terraform workspace new me-central-1
```

#### Version Control Setup
```bash
# Create repository structure
mkdir -p {infrastructure,kubernetes,applications}

# Initialize Git repository
git init
git remote add origin git@github.com:knowledgecity/infrastructure.git

# Set up branch protection
# - main: Require PR reviews, status checks
# - staging: Auto-merge from feature branches
# - feature/*: Development branches
```

### Week 2: Network Infrastructure

#### VPC Setup (Terraform)
```hcl
# infrastructure/terraform/modules/vpc/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "knowledgecity-${var.environment}-vpc"
  cidr = var.vpc_cidr  # 10.0.0.0/16 for US, 10.1.0.0/16 for Saudi

  azs              = data.aws_availability_zones.available.names
  private_subnets  = var.private_subnet_cidrs  # [10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24]
  public_subnets   = var.public_subnet_cidrs   # [10.0.101.0/24, 10.0.102.0/24, 10.0.103.0/24]
  database_subnets = var.database_subnet_cidrs # [10.0.201.0/24, 10.0.202.0/24, 10.0.203.0/24]

  enable_nat_gateway   = true
  single_nat_gateway   = false  # Multi-AZ NAT gateways
  one_nat_gateway_per_az = true
  enable_dns_hostnames = true
  enable_dns_support   = true

  # VPC Flow Logs
  enable_flow_log                      = true
  create_flow_log_cloudwatch_iam_role  = true
  create_flow_log_cloudwatch_log_group = true
  flow_log_retention_in_days           = 90

  # Kubernetes tags
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }

  tags = {
    Environment = var.environment
    Terraform   = "true"
    Compliance  = "data-residency"
  }
}

# VPC Endpoints (avoid NAT Gateway charges)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = module.vpc.vpc_id
  service_name = "com.amazonaws.${var.region}.s3"
  route_table_ids = concat(
    module.vpc.private_route_table_ids,
    module.vpc.database_route_table_ids
  )
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

#### Security Groups
```hcl
# ALB Security Group
resource "aws_security_group" "alb" {
  name        = "knowledgecity-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = module.vpc.vpc_id

  # Allow HTTPS from CloudFlare IPs only
  dynamic "ingress" {
    for_each = var.cloudflare_ips
    content {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
      description = "HTTPS from CloudFlare"
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "knowledgecity-alb-sg"
  }
}

# EKS Node Security Group
resource "aws_security_group" "eks_nodes" {
  name        = "knowledgecity-eks-nodes-sg"
  description = "Security group for EKS worker nodes"
  vpc_id      = module.vpc.vpc_id

  # Allow traffic from ALB
  ingress {
    from_port       = 0
    to_port         = 65535
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Allow from ALB"
  }

  # Allow nodes to communicate with each other
  ingress {
    from_port = 0
    to_port   = 65535
    protocol  = "-1"
    self      = true
    description = "Allow node to node communication"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "knowledgecity-eks-nodes-sg"
  }
}

# RDS Security Group
resource "aws_security_group" "rds" {
  name        = "knowledgecity-rds-sg"
  description = "Security group for RDS PostgreSQL"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
    description     = "PostgreSQL from EKS nodes"
  }

  tags = {
    Name = "knowledgecity-rds-sg"
  }
}
```

### Week 3: Compute Infrastructure

#### EKS Cluster Deployment
```bash
# Deploy EKS cluster
cd infrastructure/terraform/environments/us-east-1
terraform workspace select us-east-1
terraform apply -target=module.eks

# Configure kubectl
aws eks update-kubeconfig \
  --region us-east-1 \
  --name knowledgecity-us-east-1 \
  --alias knowledgecity-us

# Verify cluster
kubectl get nodes
kubectl get pods --all-namespaces

# Install essential add-ons
# 1. AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=knowledgecity-us-east-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# 2. EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.25"

# 3. Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=knowledgecity-us-east-1 \
  --set awsRegion=us-east-1
```

### Week 4: Database Setup

#### RDS PostgreSQL
```hcl
# infrastructure/terraform/modules/rds/main.tf
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "knowledgecity-${var.environment}"

  engine               = "postgres"
  engine_version       = "15.4"
  family               = "postgres15"
  major_engine_version = "15"
  instance_class       = "db.r6g.2xlarge"

  allocated_storage     = 500
  max_allocated_storage = 2000
  storage_encrypted     = true
  kms_key_id           = aws_kms_key.rds.arn

  db_name  = "knowledgecity"
  username = "admin"
  port     = 5432

  # Multi-AZ for high availability
  multi_az               = true
  db_subnet_group_name   = module.vpc.database_subnet_group
  vpc_security_group_ids = [aws_security_group.rds.id]

  # Backup configuration
  backup_retention_period = 35
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"
  skip_final_snapshot    = false
  final_snapshot_identifier = "knowledgecity-final-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  # Performance Insights
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  # Parameter group
  parameters = [
    {
      name  = "shared_preload_libraries"
      value = "pg_stat_statements"
    },
    {
      name  = "log_statement"
      value = "all"
    },
    {
      name  = "log_min_duration_statement"
      value = "1000"  # Log queries > 1 second
    }
  ]

  tags = {
    Environment = var.environment
    Compliance  = "data-residency"
  }
}

# RDS Proxy for connection pooling
resource "aws_db_proxy" "main" {
  name                   = "knowledgecity-proxy"
  engine_family          = "POSTGRESQL"
  auth {
    secret_arn  = aws_secretsmanager_secret.rds_credentials.arn
    iam_auth    = "DISABLED"
  }
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_subnet_ids         = module.vpc.database_subnets
  require_tls            = true

  tags = {
    Name = "knowledgecity-rds-proxy"
  }
}

resource "aws_db_proxy_default_target_group" "main" {
  db_proxy_name = aws_db_proxy.main.name

  connection_pool_config {
    max_connections_percent      = 100
    max_idle_connections_percent = 50
    connection_borrow_timeout    = 120
  }
}

resource "aws_db_proxy_target" "main" {
  db_proxy_name          = aws_db_proxy.main.name
  target_group_name      = aws_db_proxy_default_target_group.main.name
  db_instance_identifier = module.rds.db_instance_id
}
```

#### ClickHouse Cluster
```yaml
# kubernetes/infrastructure/clickhouse/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
  namespace: database
spec:
  serviceName: clickhouse
  replicas: 3  # One per AZ
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      labels:
        app: clickhouse
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: clickhouse
      
      containers:
      - name: clickhouse
        image: clickhouse/clickhouse-server:23.8
        ports:
        - containerPort: 8123
          name: http
        - containerPort: 9000
          name: native
        - containerPort: 9009
          name: interserver
        
        env:
        - name: CLICKHOUSE_DB
          value: "analytics"
        - name: CLICKHOUSE_USER
          valueFrom:
            secretKeyRef:
              name: clickhouse-credentials
              key: username
        - name: CLICKHOUSE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: clickhouse-credentials
              key: password
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/clickhouse
        - name: config
          mountPath: /etc/clickhouse-server/config.d
        
        resources:
          requests:
            cpu: 2000m
            memory: 8Gi
          limits:
            cpu: 4000m
            memory: 16Gi
      
      volumes:
      - name: config
        configMap:
          name: clickhouse-config
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 500Gi
```

---

## Phase 2: Application Deployment (Weeks 5-8)

### Week 5: Containerization

#### PHP API Dockerfile
```dockerfile
# applications/php-api/Dockerfile
FROM php:8.2-fpm-alpine AS base

# Install system dependencies
RUN apk add --no-cache \
    postgresql-dev \
    redis \
    libzip-dev \
    oniguruma-dev \
    supervisor \
    nginx

# Install PHP extensions
RUN docker-php-ext-install \
    pdo \
    pdo_pgsql \
    zip \
    bcmath \
    opcache

RUN pecl install redis && docker-php-ext-enable redis
RUN pecl install apcu && docker-php-ext-enable apcu

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Configure PHP
COPY docker/php.ini /usr/local/etc/php/conf.d/custom.ini
COPY docker/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

WORKDIR /var/www/html

# Copy application
COPY . .

# Install dependencies
RUN composer install --no-dev --optimize-autoloader --no-interaction

# Configure Nginx
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisor.conf /etc/supervisor/conf.d/supervisord.conf

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser && \
    chown -R appuser:appuser /var/www/html

USER appuser

EXPOSE 8080 9090

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

#### Build and Push
```bash
# Build multi-arch images
docker buildx create --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 123456789.dkr.ecr.us-east-1.amazonaws.com/php-api:v1.0.0 \
  --push \
  .

# Security scan
trivy image 123456789.dkr.ecr.us-east-1.amazonaws.com/php-api:v1.0.0
```

### Week 6: Kubernetes Deployments

#### Deploy Applications
```bash
# Create namespaces
kubectl create namespace application
kubectl create namespace database
kubectl create namespace monitoring
kubectl create namespace security

# Label namespaces for Istio injection
kubectl label namespace application istio-injection=enabled

# Deploy applications
kubectl apply -f kubernetes/applications/php-api/
kubectl apply -f kubernetes/applications/analytics-service/
kubectl apply -f kubernetes/applications/video-service/
kubectl apply -f kubernetes/applications/frontend/

# Verify deployments
kubectl get pods -n application
kubectl get svc -n application
kubectl get ingress -n application
```

### Week 7: Service Mesh (Istio)

#### Install Istio
```bash
# Install Istio operator
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# Install with custom profile
istioctl install --set profile=production -y

# Enable sidecar injection
kubectl label namespace application istio-injection=enabled

# Deploy Istio addons
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/kiali.yaml

# Create Gateway
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: knowledgecity-gateway
  namespace: application
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: knowledgecity-tls
    hosts:
    - "api.knowledgecity.com"
    - "*.knowledgecity.com"
EOF

# Create VirtualService
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: php-api
  namespace: application
spec:
  hosts:
  - "api.knowledgecity.com"
  gateways:
  - knowledgecity-gateway
  http:
  - match:
    - uri:
        prefix: /api/
    route:
    - destination:
        host: php-api
        port:
          number: 80
      weight: 100
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
EOF
```

### Week 8: Observability Stack

#### Deploy Monitoring
```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values - <<EOF
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    additionalScrapeConfigs:
    - job_name: 'php-api'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: php-api
        action: keep

grafana:
  adminPassword: "CHANGE_ME"
  persistence:
    enabled: true
    size: 10Gi
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-kube-prometheus-prometheus:9090
        isDefault: true
      - name: Loki
        type: loki
        url: http://loki:3100
      - name: Tempo
        type: tempo
        url: http://tempo:3100
EOF

# Install Loki
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values - <<EOF
loki:
  persistence:
    enabled: true
    size: 100Gi
  config:
    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h  # 7 days

promtail:
  enabled: true
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push
EOF

# Install Tempo
helm install tempo grafana/tempo \
  --namespace monitoring \
  --values - <<EOF
tempo:
  persistence:
    enabled: true
    size: 50Gi
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
EOF
```

---

## Phase 3: Security & Compliance (Weeks 9-10)

### Week 9: Security Hardening

#### External Secrets Operator
```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace security \
  --create-namespace

# Create SecretStore
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: application
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
EOF

# Create ExternalSecret
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: php-api-secrets
  namespace: application
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: php-api-secrets
    creationPolicy: Owner
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: /knowledgecity/us-east-1/rds/credentials
      property: password
  - secretKey: JWT_PRIVATE_KEY
    remoteRef:
      key: /knowledgecity/us-east-1/jwt/private-key
  - secretKey: REDIS_PASSWORD
    remoteRef:
      key: /knowledgecity/us-east-1/redis/password
EOF
```

### Week 10: Compliance Setup

#### Data Residency Policies
```yaml
# kubernetes/security/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-region
  namespace: application
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow within cluster
  - to:
    - podSelector: {}
  # Allow to RDS in same region
  - to:
    - ipBlock:
        cidr: 10.0.0.0/16  # VPC CIDR
    ports:
    - protocol: TCP
      port: 5432
  # Block all other egress
```

---

## Phase 4: Go-Live (Weeks 11-12)

### Week 11: Performance Testing

#### Load Testing
```bash
# Install k6
brew install k6

# Run load test
k6 run - <<EOF
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 200 },   // Ramp up
    { duration: '5m', target: 200 },   // Stay at 200 users
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% under 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  let response = http.get('https://api.knowledgecity.com/api/courses');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
EOF
```

### Week 12: Go-Live Checklist

```yaml
Pre-Launch Checklist:
  Infrastructure:
    ☐ Multi-region deployment complete
    ☐ Multi-AZ redundancy verified
    ☐ Auto-scaling tested
    ☐ Disaster recovery tested
  
  Security:
    ☐ WAF rules active
    ☐ DDoS protection enabled
    ☐ SSL certificates installed
    ☐ Secrets rotation configured
    ☐ Security scanning in CI/CD
  
  Monitoring:
    ☐ Prometheus collecting metrics
    ☐ Grafana dashboards created
    ☐ Alerts configured
    ☐ PagerDuty integration tested
  
  Compliance:
    ☐ Data residency verified
    ☐ Encryption enabled (at rest & in transit)
    ☐ Audit logging active
    ☐ Privacy policy updated
  
  Performance:
    ☐ Load testing passed
    ☐ CDN cache hit rate > 85%
    ☐ API response time p95 < 500ms
    ☐ Database query optimization
  
  Documentation:
    ☐ Architecture diagrams
    ☐ Runbooks for incidents
    ☐ API documentation
    ☐ Deployment guide
  
  Training:
    ☐ Operations team trained
    ☐ Support team trained
    ☐ On-call rotation established
```

---

## Maintenance & Operations

### Daily Operations
```bash
# Check cluster health
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running

# Check resource usage
kubectl top nodes
kubectl top pods -n application

# Review logs
kubectl logs -n application -l app=php-api --tail=100

# Check scaling
kubectl get hpa -n application
```

### Weekly Maintenance
```bash
# Review cost reports
aws ce get-cost-and-usage \
  --time-period Start=2025-11-05,End=2025-11-12 \
  --granularity DAILY \
  --metrics "BlendedCost" \
  --group-by Type=SERVICE

# Review security findings
aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}' \
  --max-results 100

# Update container images
kubectl set image deployment/php-api php-api=php-api:v1.1.0 -n application
kubectl rollout status deployment/php-api -n application

# Database maintenance
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,PendingMaintenanceActions]'
```

### Monthly Tasks
- Review and optimize costs
- Security vulnerability patching
- Performance tuning
- Disaster recovery drill
- Compliance audit
- Team retrospective

---

## Troubleshooting Guide

### Common Issues

#### Pod CrashLoopBackOff
```bash
# Check pod logs
kubectl logs -n application <pod-name> --previous

# Describe pod for events
kubectl describe pod -n application <pod-name>

# Check resource limits
kubectl describe node <node-name>

# Common fixes:
# 1. Increase memory/CPU limits
# 2. Fix application configuration
# 3. Check secrets are mounted correctly
```

#### High API Latency
```bash
# Check database performance
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,ReadLatency,WriteLatency]'

# Check Redis cache hit rate
redis-cli INFO stats | grep keyspace_hits

# Check pod resource usage
kubectl top pods -n application -l app=php-api

# Solutions:
# 1. Scale horizontally (increase replicas)
# 2. Optimize database queries
# 3. Increase cache TTL
# 4. Enable database read replicas
```

#### Data Compliance Issue
```bash
# Verify data location
aws s3api get-bucket-location --bucket knowledgecity-us-data
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,AvailabilityZone]'

# Check for cross-region traffic
aws ec2 describe-flow-logs --filter Name=resource-type,Values=VPC

# Audit data access
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::S3::Bucket
```

---

## Cost Optimization Tips

1. **Use Savings Plans**: 50% savings on compute
2. **S3 Intelligent-Tiering**: Automatic cost optimization
3. **Spot Instances**: 70-90% discount for video processing
4. **Reserved Instances**: For RDS and ElastiCache
5. **CloudFront**: Reduce origin requests (90% cache hit rate)
6. **Right-sizing**: Monthly review of resource utilization
7. **Data Transfer**: Use VPC endpoints to avoid NAT charges

Expected savings: **~40% reduction** from initial costs.

---

## Success Metrics

Track these KPIs:
- **Availability**: Target 99.99% (43 minutes downtime/month)
- **API Response Time**: p95 < 500ms
- **Error Rate**: < 0.1%
- **Cache Hit Rate**: > 85%
- **Cost per User**: < $0.50/month
- **Security Incidents**: 0 breaches
- **Compliance Violations**: 0

---

This implementation guide provides a structured, week-by-week approach to deploying the KnowledgeCity platform. Follow this guide systematically, and adjust timelines based on your team's capacity and specific requirements.

