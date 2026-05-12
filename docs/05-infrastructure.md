# 🌐 Infrastructure & Déploiement - Pennylane Premium OS

## Architecture Cloud AWS

### Regions

```
┌──────────────────────────────────────────────────────┐
│                AWS Global Infrastructure             │
├──────────────────────────────────────────────────────┤
│                                                      │
│  eu-west-1 (Dublin) - PRIMARY                       │
│  ├── EKS Cluster (3 nodes)                          │
│  ├── RDS PostgreSQL (Primary)                       │
│  ├── ElastiCache Redis (6 nodes)                    │
│  └── S3 / MinIO Storage                             │
│                                                      │
│  us-east-1 (N. Virginia) - SECONDARY                │
│  ├── EKS Cluster (2 nodes)                          │
│  ├── RDS PostgreSQL (Read Replica)                  │
│  └── CloudFront Distribution                        │
│                                                      │
│  ap-southeast-1 (Singapore) - BACKUP                │
│  └── RDS Backup (Daily)                             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

## Kubernetes Configuration

### Helm Charts

```bash
# Deploy Pennylane
helm install pennylane ./infrastructure/helm/charts/pennylane \
  --namespace pennylane \
  --values ./infrastructure/helm/values-prod.yaml

# Upgrade
helm upgrade pennylane ./infrastructure/helm/charts/pennylane \
  --namespace pennylane \
  --values ./infrastructure/helm/values-prod.yaml
```

### Resource Requests

```yaml
# services/auth-service/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: pennylane
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: registry.pennylane-os.io/auth-service:latest
        ports:
        - containerPort: 3001
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Terraform Configuration

### VPC Setup

```hcl
# infrastructure/terraform/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket         = "pennylane-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = "pennylane"
      ManagedBy   = "terraform"
    }
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "pennylane-vpc"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 3
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "pennylane-public-${count.index + 1}"
  }
}

# Private Subnets (RDS)
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + 3)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "pennylane-private-${count.index + 1}"
  }
}
```

## RDS PostgreSQL

```hcl
# infrastructure/terraform/rds.tf
resource "aws_db_instance" "main" {
  identifier     = "pennylane-postgres"
  engine         = "postgres"
  engine_version = "15.3"
  instance_class = "db.r5.xlarge"

  allocated_storage = 100
  storage_type      = "gp3"
  iops              = 3000
  storage_encrypted = true

  db_name  = "pennylane_prod"
  username = var.db_username
  password = random_password.db_password.result

  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "mon:04:00-mon:05:00"

  skip_final_snapshot       = false
  final_snapshot_identifier = "pennylane-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  enabled_cloudwatch_logs_exports = ["postgresql"]

  # High Availability
  multi_az = true

  # Parameter group
  parameter_group_name = aws_db_parameter_group.postgres.name

  tags = {
    Name = "pennylane-postgres"
  }
}

# Read Replicas
resource "aws_db_instance" "replica" {
  count              = 2
  identifier         = "pennylane-postgres-replica-${count.index + 1}"
  replicate_source_db = aws_db_instance.main.identifier
  instance_class     = "db.r5.large"
  skip_final_snapshot = true

  tags = {
    Name = "pennylane-postgres-replica-${count.index + 1}"
  }
}
```

## CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run type-check

      - name: Run tests
        run: npm run test:ci
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions
          aws-region: eu-west-1

      - name: Login to ECR
        run: aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/pennylane:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/pennylane:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions
          aws-region: eu-west-1

      - name: Update EKS kubeconfig
        run: aws eks update-kubeconfig --region eu-west-1 --name pennylane-prod

      - name: Update deployment
        run: |
          kubectl set image deployment/pennylane-web \
            web=${{ secrets.ECR_REGISTRY }}/pennylane:${{ github.sha }} \
            --namespace=pennylane

      - name: Wait for rollout
        run: kubectl rollout status deployment/pennylane-web -n pennylane --timeout=5m
```

## Monitoring & Alerting

### Prometheus

```yaml
# infrastructure/monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - pennylane

alert_rules:
  - /etc/prometheus/rules/*.yml
```

### Grafana Dashboards

```
- System Health
- Application Metrics
- Database Performance
- API Response Times
- Error Rates
- Business KPIs
```

## Backup Strategy

```
Daily backups
├── Automated 03:00 UTC
├── Stored in S3 (eu-west-1)
├── Replicated to us-east-1
├── Retention: 30 days
└── Tested monthly
```

## Disaster Recovery

```
RTO (Recovery Time Objective): < 1 hour
RPO (Recovery Point Objective): < 5 minutes

Strategy:
1. Multi-region failover
2. Auto-scaling recovery
3. Database point-in-time restore
4. Load balancer switchover
```

---

Terraform modules: `infrastructure/terraform/modules/`
Helm charts: `infrastructure/helm/charts/`
