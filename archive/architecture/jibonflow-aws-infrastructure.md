# JibonFlow AWS Infrastructure Topology

**Version**: 1.0.0  
**Generated**: 2025-10-11  
**Phase**: Architecture Documentation (Day 1-2)  
**Status**: Production-Ready  
**Cloud Provider**: Amazon Web Services (AWS)  
**Primary Region**: ap-southeast-1 (Singapore)  
**DR Region**: ap-south-1 (Mumbai)

---

## 1. High-Level Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           INTERNET (Public)                                 │
└────────────────────────────┬──────────────────────────────────────────────┘
                             │
                    ┌────────▼─────────┐
                    │  Route 53 (DNS)  │
                    │  jibonflow.com   │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  CloudFront CDN  │
                    │  (Static Assets) │
                    └────────┬─────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────────────┐
│                  AWS VPC (10.0.0.0/16)                                      │
│                                                                              │
│  ┌───────────────────────┬──────────────────────────────────────────────┐  │
│  │  PUBLIC SUBNETS      │  PRIVATE SUBNETS                              │  │
│  │  (10.0.1.0/24)       │  (10.0.10.0/24, 10.0.11.0/24)                 │  │
│  │                       │                                                │  │
│  │  ┌─────────────────┐ │  ┌────────────────────────────────────────┐  │  │
│  │  │ ALB (Public)    │ │  │  ECS Fargate Cluster                   │  │  │
│  │  │ Port 80/443     │ │  │  ┌──────────────┬─────────────────┐   │  │  │
│  │  └────────┬────────┘ │  │  │ API Gateway  │ Backend Services │   │  │  │
│  │           │           │  │  │ (Port 3000)  │ (Ports 4000-400│   │  │  │
│  │           │           │  │  │ 4 tasks      │ 20 tasks total) │   │  │  │
│  │  ┌────────▼────────┐ │  │  └──────┬───────┴──────┬──────────┘   │  │  │
│  │  │ NAT Gateway     │ │  │         │              │               │  │  │
│  │  │ (For Outbound)  │ │  │  ┌──────▼──────────────▼──────────┐   │  │  │
│  │  └─────────────────┘ │  │  │  RDS PostgreSQL 14 Multi-AZ    │   │  │  │
│  │                       │  │  │  Primary: 10.0.10.50           │   │  │  │
│  │                       │  │  │  Standby: 10.0.11.50          │   │  │  │
│  └───────────────────────┘  │  └────────────────────────────────┘   │  │
│                              │                                        │  │
│                              │  ┌────────────────────────────────┐   │  │
│                              │  │  ElastiCache Redis 7 Cluster   │   │  │
│                              │  │  Primary: 10.0.10.60          │   │  │
│                              │  │  Replica: 10.0.11.60          │   │  │
│                              │  └────────────────────────────────┘   │  │
│                              └────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                       │
                              ┌────────▼─────────┐
                              │  AWS S3 Buckets  │
                              │  - PHI Documents │
                              │  - Audit Logs    │
                              │  - Backups       │
                              └──────────────────┘
                                       │
                              ┌────────▼─────────┐
                              │  AWS KMS Keys    │
                              │  - PHI Encryption│
                              │  - S3 Encryption │
                              └──────────────────┘
```

---

## 2. Network Architecture

### 2.1 VPC Configuration

**VPC CIDR**: `10.0.0.0/16` (65,536 IP addresses)

| Subnet Type | AZ | CIDR Block | Purpose | Route Table |
|-------------|-----|-----------|---------|-------------|
| **Public Subnet 1** | ap-southeast-1a | `10.0.1.0/24` | ALB, NAT Gateway | Internet Gateway |
| **Public Subnet 2** | ap-southeast-1b | `10.0.2.0/24` | ALB (failover), NAT Gateway | Internet Gateway |
| **Private Subnet 1** | ap-southeast-1a | `10.0.10.0/24` | ECS tasks, RDS primary | NAT Gateway |
| **Private Subnet 2** | ap-southeast-1b | `10.0.11.0/24` | ECS tasks, RDS standby | NAT Gateway |
| **Database Subnet 1** | ap-southeast-1a | `10.0.20.0/24` | RDS, ElastiCache | No internet access |
| **Database Subnet 2** | ap-southeast-1b | `10.0.21.0/24` | RDS, ElastiCache | No internet access |

**Terraform Configuration**:
```hcl
# terraform/vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "jibonflow-vpc-${var.environment}"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "jibonflow-public-${count.index + 1}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "jibonflow-private-${count.index + 1}"
    Type = "private"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "jibonflow-igw"
  }
}

resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = {
    Name = "jibonflow-nat-${count.index + 1}"
  }
}
```

### 2.2 Security Groups

**Security Group Architecture**:

```
┌────────────────────────────────────────────────────────┐
│  ALB Security Group (sg-alb)                           │
│  Inbound:  0.0.0.0/0:443 (HTTPS)                      │
│  Outbound: sg-ecs:3000-4010                           │
└────────────────────────────────────────────────────────┘
                         │
            ┌────────────▼─────────────┐
            │  ECS Security Group      │
            │  (sg-ecs)                │
            │  Inbound:  sg-alb:*      │
            │  Outbound: sg-rds:5432   │
            │            sg-redis:6379 │
            │            0.0.0.0/0:443 │
            └────────┬─────────┬───────┘
                     │         │
        ┌────────────▼──┐  ┌──▼────────────┐
        │ RDS SG        │  │ Redis SG      │
        │ (sg-rds)      │  │ (sg-redis)    │
        │ In: sg-ecs:5432│  │ In: sg-ecs:6379│
        └───────────────┘  └───────────────┘
```

**Terraform Configuration**:
```hcl
# terraform/security-groups.tf
resource "aws_security_group" "alb" {
  name        = "jibonflow-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTP from internet (redirect to HTTPS)"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    description     = "To ECS tasks"
    from_port       = 3000
    to_port         = 4010
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }
  
  tags = {
    Name = "jibonflow-alb-sg"
  }
}

resource "aws_security_group" "ecs" {
  name        = "jibonflow-ecs-sg"
  description = "Security group for ECS tasks"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description     = "From ALB"
    from_port       = 3000
    to_port         = 4010
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  
  egress {
    description     = "To RDS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
  }
  
  egress {
    description     = "To Redis"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.redis.id]
  }
  
  egress {
    description = "To internet (external APIs)"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "jibonflow-ecs-sg"
  }
}

resource "aws_security_group" "rds" {
  name        = "jibonflow-rds-sg"
  description = "Security group for RDS PostgreSQL"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description     = "From ECS tasks"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }
  
  tags = {
    Name = "jibonflow-rds-sg"
  }
}

resource "aws_security_group" "redis" {
  name        = "jibonflow-redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description     = "From ECS tasks"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }
  
  tags = {
    Name = "jibonflow-redis-sg"
  }
}
```

---

## 3. Compute Layer (ECS Fargate)

### 3.1 ECS Cluster

**Cluster Configuration**:
- **Cluster Name**: `jibonflow-cluster-production`
- **Launch Type**: AWS Fargate (serverless, no EC2 instance management)
- **Capacity Providers**: Fargate (on-demand) + Fargate Spot (70% cost savings for non-critical tasks)

**Terraform Configuration**:
```hcl
# terraform/ecs-cluster.tf
resource "aws_ecs_cluster" "main" {
  name = "jibonflow-cluster-${var.environment}"
  
  setting {
    name  = "containerInsights"
    value = "enabled"  # Enable CloudWatch Container Insights
  }
  
  tags = {
    Name        = "jibonflow-cluster"
    Environment = var.environment
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 2   # Minimum 2 tasks on Fargate
    weight            = 1
    capacity_provider = "FARGATE"
  }
  
  default_capacity_provider_strategy {
    weight            = 4  # 80% of tasks on Fargate Spot
    capacity_provider = "FARGATE_SPOT"
  }
}
```

### 3.2 ECS Services

**Service Definitions**:

| Service | Port | Task Count (Min/Max) | CPU (vCPU) | Memory (MB) | Auto-Scaling Target |
|---------|------|---------------------|------------|-------------|---------------------|
| **api-gateway** | 3000 | 2 / 10 | 0.5 | 1024 | CPU > 70% |
| **auth-service** | 4000 | 2 / 10 | 0.25 | 512 | CPU > 70% |
| **patient-management** | 4001 | 2 / 10 | 0.5 | 1024 | CPU > 70% |
| **telemedicine-service** | 4002 | 2 / 10 | 1 | 2048 | CPU > 70% |
| **payment-service** | 4003 | 2 / 10 | 0.5 | 1024 | CPU > 70% |
| **medicine-verification** | 4004 | 2 / 5 | 0.25 | 512 | CPU > 70% |
| **logistics-tracking** | 4005 | 2 / 5 | 0.25 | 512 | CPU > 70% |
| **loyalty-rewards** | 4006 | 2 / 5 | 0.25 | 512 | CPU > 70% |
| **notification-service** | 4007 | 2 / 10 | 0.5 | 1024 | CPU > 70% |
| **audit-logging** | 4008 | 2 | 10 | 0.5 | 1024 | N/A (fixed count) |

**Task Definition Example** (auth-service):
```hcl
# terraform/ecs-task-auth.tf
resource "aws_ecs_task_definition" "auth_service" {
  family                   = "jibonflow-auth-service"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"   # 0.25 vCPU
  memory                   = "512"   # 512 MB
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([{
    name  = "auth-service"
    image = "${aws_ecr_repository.auth_service.repository_url}:latest"
    
    portMappings = [{
      containerPort = 4000
      protocol      = "tcp"
    }]
    
    environment = [
      { name = "NODE_ENV", value = "production" },
      { name = "PORT", value = "4000" },
      { name = "DATABASE_HOST", value = aws_db_instance.postgres.address }
    ]
    
    secrets = [
      {
        name      = "DATABASE_PASSWORD"
        valueFrom = aws_secretsmanager_secret.db_password.arn
      },
      {
        name      = "JWT_PRIVATE_KEY"
        valueFrom = aws_secretsmanager_secret.jwt_private_key.arn
      },
      {
        name      = "TWILIO_AUTH_TOKEN"
        valueFrom = aws_secretsmanager_secret.twilio_auth_token.arn
      }
    ]
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/jibonflow-auth-service"
        "awslogs-region"        = "ap-southeast-1"
        "awslogs-stream-prefix" = "ecs"
      }
    }
    
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:4000/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
  }])
  
  tags = {
    Name    = "jibonflow-auth-service"
    Service = "auth-service"
  }
}
```

### 3.3 Auto-Scaling Policies

**See NFR Documentation Section 4.2** for complete auto-scaling configuration.

**Scaling Metrics**:
- **Scale Out**: When CPU > 70% for 2 consecutive minutes
- **Scale In**: When CPU < 30% for 5 consecutive minutes
- **Cooldown**: 60 seconds (scale out), 300 seconds (scale in)

---

## 4. Database Layer

### 4.1 RDS PostgreSQL 14

**Instance Configuration**:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Instance Class** | db.r6g.xlarge | 4 vCPU, 32 GB RAM (baseline for 10,000 patients) |
| **Engine** | PostgreSQL 14.9 | Latest stable version with JSONB support |
| **Multi-AZ** | ✅ Enabled | Automatic failover to standby in ap-southeast-1b |
| **Storage Type** | gp3 (SSD) | 3,000 IOPS baseline, scalable to 16,000 IOPS |
| **Allocated Storage** | 500 GB | Auto-scaling enabled (max 2 TB) |
| **Backup Retention** | 30 days | Daily automated snapshots |
| **Encryption** | ✅ Enabled (AWS KMS) | HIPAA requirement |
| **Performance Insights** | ✅ Enabled | Query performance monitoring |
| **Enhanced Monitoring** | ✅ Enabled (60s interval) | OS-level metrics |

**Terraform Configuration**:
```hcl
# terraform/rds.tf
resource "aws_db_instance" "postgres" {
  identifier     = "jibonflow-postgres-${var.environment}"
  engine         = "postgres"
  engine_version = "14.9"
  
  instance_class        = "db.r6g.xlarge"
  allocated_storage     = 500
  max_allocated_storage = 2000  # Auto-scaling enabled
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds_encryption.arn
  
  db_name  = "jibonflow"
  username = "jibonflow_admin"
  password = random_password.db_password.result
  
  multi_az               = true
  publicly_accessible    = false
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 30
  backup_window           = "03:00-04:00"  # 3-4 AM Bangladesh Time
  maintenance_window      = "sun:04:00-sun:05:00"
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  performance_insights_enabled    = true
  performance_insights_retention_period = 7
  
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "jibonflow-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"
  
  tags = {
    Name        = "jibonflow-postgres"
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

resource "aws_db_subnet_group" "main" {
  name       = "jibonflow-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
  
  tags = {
    Name = "jibonflow-db-subnet-group"
  }
}
```

**Parameter Group** (Optimized for OLTP):
```hcl
resource "aws_db_parameter_group" "postgres" {
  name   = "jibonflow-postgres-14"
  family = "postgres14"
  
  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4096}"  # 25% of RAM
  }
  
  parameter {
    name  = "effective_cache_size"
    value = "{DBInstanceClassMemory/1365}"  # 75% of RAM
  }
  
  parameter {
    name  = "work_mem"
    value = "16384"  # 16 MB per query
  }
  
  parameter {
    name  = "maintenance_work_mem"
    value = "1048576"  # 1 GB for VACUUM/ANALYZE
  }
  
  parameter {
    name  = "random_page_cost"
    value = "1.1"  # SSD-optimized
  }
  
  parameter {
    name  = "log_min_duration_statement"
    value = "500"  # Log queries > 500ms
  }
  
  parameter {
    name  = "log_connections"
    value = "1"
  }
  
  parameter {
    name  = "log_disconnections"
    value = "1"
  }
  
  parameter {
    name  = "ssl"
    value = "1"  # Force TLS connections
  }
  
  tags = {
    Name = "jibonflow-postgres-params"
  }
}
```

### 4.2 ElastiCache Redis 7

**Cluster Configuration**:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Node Type** | cache.r6g.large | 2 vCPU, 13.07 GB RAM |
| **Engine** | Redis 7.0 | Latest version with cluster mode enabled |
| **Cluster Mode** | ✅ Enabled | Horizontal scaling with sharding |
| **Shards** | 3 | Distribute keys across 3 primary nodes |
| **Replicas per Shard** | 1 | 1 replica for each primary (total 6 nodes) |
| **Encryption in Transit** | ✅ Enabled (TLS 1.2) | HIPAA requirement |
| **Encryption at Rest** | ✅ Enabled (AWS managed key) | HIPAA requirement |
| **Automatic Failover** | ✅ Enabled | Promote replica to primary on failure |
| **Snapshot Retention** | 7 days | Daily automated backups |

**Terraform Configuration**:
```hcl
# terraform/elasticache.tf
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "jibonflow-redis-${var.environment}"
  replication_group_description = "JibonFlow Redis cluster"
  engine                     = "redis"
  engine_version             = "7.0"
  node_type                  = "cache.r6g.large"
  num_node_groups            = 3  # 3 shards
  replicas_per_node_group    = 1  # 1 replica per shard
  
  port                       = 6379
  parameter_group_name       = aws_elasticache_parameter_group.redis.name
  subnet_group_name          = aws_elasticache_subnet_group.main.name
  security_group_ids         = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token_enabled         = true
  auth_token                 = random_password.redis_auth_token.result
  
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  snapshot_retention_limit   = 7
  snapshot_window            = "03:00-04:00"
  maintenance_window         = "sun:04:00-sun:05:00"
  
  notification_topic_arn     = aws_sns_topic.elasticache_alerts.arn
  
  tags = {
    Name        = "jibonflow-redis"
    Environment = var.environment
  }
}

resource "aws_elasticache_subnet_group" "main" {
  name       = "jibonflow-redis-subnet-group"
  subnet_ids = aws_subnet.private[*].id
  
  tags = {
    Name = "jibonflow-redis-subnet-group"
  }
}

resource "aws_elasticache_parameter_group" "redis" {
  name   = "jibonflow-redis-7"
  family = "redis7"
  
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"  # Evict least recently used keys when memory full
  }
  
  parameter {
    name  = "timeout"
    value = "300"  # Close idle connections after 5 minutes
  }
  
  parameter {
    name  = "tcp-keepalive"
    value = "60"  # Send TCP keepalive every 60 seconds
  }
  
  tags = {
    Name = "jibonflow-redis-params"
  }
}
```

---

## 5. Storage Layer

### 5.1 S3 Buckets

**Bucket Architecture**:

| Bucket Name | Purpose | Encryption | Versioning | Lifecycle Policy | Access |
|-------------|---------|------------|-----------|------------------|--------|
| **jibonflow-phi-documents** | PHI documents (prescriptions, test results) | KMS (customer-managed) | ✅ Enabled | Archive to Glacier after 90 days | Private |
| **jibonflow-audit-logs** | HIPAA audit trail exports | KMS (customer-managed) | ✅ Enabled | Archive to Glacier after 30 days, delete after 6 years | Private |
| **jibonflow-backups** | RDS/Redis snapshots | KMS (AWS-managed) | ✅ Enabled | Delete after 90 days | Private |
| **jibonflow-static-assets** | Frontend static files (images, CSS, JS) | AES-256 (S3-managed) | ❌ Disabled | None (CDN cache invalidation) | Public (via CloudFront) |
| **jibonflow-uploads-temp** | Temporary user uploads | KMS (customer-managed) | ❌ Disabled | Auto-delete after 7 days | Private |

**Terraform Configuration** (PHI Documents Bucket):
```hcl
# terraform/s3-phi-documents.tf
resource "aws_s3_bucket" "phi_documents" {
  bucket = "jibonflow-phi-documents-${var.environment}"
  
  tags = {
    Name        = "jibonflow-phi-documents"
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

# Enable versioning (HIPAA requirement)
resource "aws_s3_bucket_versioning" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  versioning_configuration {
    status = "Enabled"
    mfa_delete = "Enabled"  # Require MFA to delete versions
  }
}

# Enable encryption with KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.phi_encryption.arn
    }
    bucket_key_enabled = true
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle policy (archive to Glacier)
resource "aws_s3_bucket_lifecycle_configuration" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  rule {
    id     = "archive-old-documents"
    status = "Enabled"
    
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    
    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }
    
    # HIPAA requires 6-year retention
    expiration {
      days = 2190  # 6 years
    }
  }
}

# Cross-region replication (DR)
resource "aws_s3_bucket_replication_configuration" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  role   = aws_iam_role.s3_replication.arn
  
  rule {
    id     = "replicate-to-mumbai"
    status = "Enabled"
    
    destination {
      bucket        = aws_s3_bucket.phi_documents_dr.arn
      storage_class = "STANDARD_IA"
      
      replication_time {
        status = "Enabled"
        time {
          minutes = 15  # Replicate within 15 minutes
        }
      }
      
      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
    }
  }
  
  depends_on = [aws_s3_bucket_versioning.phi_documents]
}
```

### 5.2 KMS Encryption Keys

**Key Hierarchy**:

| Key Alias | Purpose | Rotation | Used By |
|-----------|---------|----------|---------|
| **alias/jibonflow-phi-encryption** | PHI encryption (database, S3) | Annual (automatic) | RDS, S3 PHI bucket, application secrets |
| **alias/jibonflow-rds-encryption** | RDS storage encryption | Annual (automatic) | RDS PostgreSQL |
| **alias/jibonflow-elasticache-encryption** | Redis data encryption | Annual (automatic) | ElastiCache Redis |
| **alias/jibonflow-secrets-encryption** | AWS Secrets Manager | Annual (automatic) | Secrets Manager |

**Terraform Configuration**:
```hcl
# terraform/kms.tf
resource "aws_kms_key" "phi_encryption" {
  description             = "JibonFlow PHI Encryption Key"
  deletion_window_in_days = 30
  enable_key_rotation     = true  # Auto-rotate every year
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow ECS tasks to decrypt"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.ecs_task.arn
        }
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = "*"
      },
      {
        Sid    = "Allow S3 to use key"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })
  
  tags = {
    Name       = "jibonflow-phi-encryption-key"
    Compliance = "HIPAA"
  }
}

resource "aws_kms_alias" "phi_encryption" {
  name          = "alias/jibonflow-phi-${var.environment}"
  target_key_id = aws_kms_key.phi_encryption.key_id
}
```

---

## 6. Load Balancing & CDN

### 6.1 Application Load Balancer (ALB)

**ALB Configuration**:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Scheme** | Internet-facing | Accept traffic from public internet |
| **IP Address Type** | IPv4 | Bangladesh internet infrastructure primarily IPv4 |
| **Subnets** | Public subnets (ap-southeast-1a, 1b) | Multi-AZ for high availability |
| **SSL Certificate** | ACM certificate (*.jibonflow.com) | Free, auto-renewing TLS certificate |
| **Security Policy** | ELBSecurityPolicy-TLS-1-2-2017-01 | TLS 1.2+ only (HIPAA compliance) |
| **Access Logs** | ✅ Enabled (S3) | Audit trail for all requests |
| **Connection Draining** | 300 seconds | Gracefully shutdown connections during deployment |

**Terraform Configuration**:
```hcl
# terraform/alb.tf
resource "aws_lb" "main" {
  name               = "jibonflow-alb-${var.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = true
  enable_http2               = true
  enable_cross_zone_load_balancing = true
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true
  }
  
  tags = {
    Name        = "jibonflow-alb"
    Environment = var.environment
  }
}

# HTTP listener (redirect to HTTPS)
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type = "redirect"
    
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# HTTPS listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api_gateway.arn
  }
}

# Target group for API Gateway
resource "aws_lb_target_group" "api_gateway" {
  name        = "jibonflow-api-gateway-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"  # Required for Fargate
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }
  
  deregistration_delay = 300  # 5-minute connection draining
  
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 28800  # 8 hours (session duration)
    enabled         = true
  }
  
  tags = {
    Name = "jibonflow-api-gateway-tg"
  }
}
```

### 6.2 CloudFront CDN

**Distribution Configuration**:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Origin** | S3 (jibonflow-static-assets) | Serve static files from S3 |
| **Edge Locations** | All (global distribution) | Low latency for users worldwide |
| **Price Class** | PriceClass_100 | US, Canada, Europe, Asia (Bangladesh included) |
| **Viewer Protocol** | Redirect HTTP to HTTPS | Force TLS encryption |
| **Minimum TLS Version** | TLSv1.2_2021 | HIPAA compliance |
| **Caching** | 1 hour (default), 1 week (immutable assets) | Balance freshness vs. performance |
| **Compression** | ✅ Enabled (Gzip, Brotli) | Reduce bandwidth usage |

**Terraform Configuration**:
```hcl
# terraform/cloudfront.tf
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_s3_bucket.static_assets.bucket_regional_domain_name
    origin_id   = "S3-jibonflow-static-assets"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }
  
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "JibonFlow Static Assets CDN"
  default_root_object = "index.html"
  
  aliases = ["static.jibonflow.com"]
  
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-jibonflow-static-assets"
    
    forwarded_values {
      query_string = false
      
      cookies {
        forward = "none"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600    # 1 hour
    max_ttl                = 604800  # 1 week
    compress               = true
  }
  
  # Custom cache behavior for immutable assets (long TTL)
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-jibonflow-static-assets"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 31536000  # 1 year
    default_ttl            = 31536000
    max_ttl                = 31536000
    compress               = true
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cloudfront.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
  
  price_class = "PriceClass_100"
  
  tags = {
    Name        = "jibonflow-cdn"
    Environment = var.environment
  }
}
```

---

## 7. Monitoring & Observability

### 7.1 CloudWatch Dashboards

**Dashboard Components**:

```
┌─────────────────────────────────────────────────────────────┐
│  JibonFlow Production Dashboard                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ API Latency  │  │ Error Rate   │  │  Throughput  │     │
│  │  180ms p95   │  │    0.2%      │  │  150 req/s   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ECS CPU Utilization (per service)                    │  │
│  │  api-gateway:        65% ████████████                │  │
│  │  auth-service:       45% ████████                    │  │
│  │  patient-management: 55% ██████████                  │  │
│  │  telemedicine:       70% ██████████████ (scaling)   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Database Metrics                                      │  │
│  │  Connections: 120 / 200 (60%)                        │  │
│  │  CPU: 45%                                            │  │
│  │  IOPS: 1,200 / 3,000                                 │  │
│  │  Slowest Query: SELECT FROM patients (450ms)        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Redis Cache Hit Ratio: 85%                           │  │
│  │  Evictions: 0                                        │  │
│  │  Memory Usage: 8 GB / 13 GB (62%)                    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Terraform Configuration**:
```hcl
# terraform/cloudwatch-dashboard.tf
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "JibonFlow-Production"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", { stat = "p95" }]
          ]
          period = 300
          stat   = "Average"
          region = "ap-southeast-1"
          title  = "API Latency (p95)"
          yAxis = {
            left = {
              min = 0
              max = 500
            }
          }
        }
      },
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ECS", "CPUUtilization", { serviceName = "jibonflow-api-gateway" }],
            ["...", { serviceName = "jibonflow-auth-service" }],
            ["...", { serviceName = "jibonflow-telemedicine-service" }]
          ]
          period = 300
          stat   = "Average"
          region = "ap-southeast-1"
          title  = "ECS CPU Utilization"
        }
      }
    ]
  })
}
```

### 7.2 CloudWatch Alarms

**Critical Alarms**:

| Alarm Name | Metric | Threshold | Action |
|------------|--------|-----------|--------|
| **HighAPILatency** | ALB TargetResponseTime (p95) | > 500ms for 2 min | SNS → PagerDuty |
| **HighErrorRate** | ALB HTTPCode_Target_5XX_Count | > 10 errors/min | SNS → PagerDuty |
| **DatabaseCPUHigh** | RDS CPUUtilization | > 80% for 5 min | SNS → Slack |
| **DatabaseConnectionsHigh** | RDS DatabaseConnections | > 160 (80% of max) | SNS → Slack |
| **RedisCacheMemoryHigh** | ElastiCache DatabaseMemoryUsagePercentage | > 90% | SNS → Slack |
| **ECSTaskCountLow** | ECS DesiredTaskCount | < 2 for any service | SNS → PagerDuty |
| **S3BucketReplicationFailed** | S3 ReplicationLatency | > 30 min | SNS → Slack |

**Terraform Configuration**:
```hcl
# terraform/cloudwatch-alarms.tf
resource "aws_cloudwatch_metric_alarm" "high_api_latency" {
  alarm_name          = "JibonFlow-HighAPILatency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Average"
  threshold           = 0.5  # 500ms
  alarm_description   = "API latency exceeded 500ms p95"
  alarm_actions       = [aws_sns_topic.pagerduty.arn]
  
  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
  }
}

resource "aws_cloudwatch_metric_alarm" "database_cpu_high" {
  alarm_name          = "JibonFlow-DatabaseCPUHigh"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Database CPU utilization exceeded 80%"
  alarm_actions       = [aws_sns_topic.slack.arn]
  
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.postgres.id
  }
}
```

### 7.3 X-Ray Distributed Tracing

**Tracing Configuration**:

```typescript
// shared-packages/xray-tracing/src/index.ts
import AWSXRay from 'aws-xray-sdk-core';
import AWS from 'aws-sdk';

// Instrument AWS SDK
const instrumentedAWS = AWSXRay.captureAWS(AWS);

// Instrument HTTP clients
const http = AWSXRay.captureHTTPsGlobal(require('http'));
const https = AWSXRay.captureHTTPsGlobal(require('https'));

// Express middleware
export function xrayMiddleware() {
  return AWSXRay.express.openSegment('JibonFlow-API');
}

export function xrayCloseMiddleware() {
  return AWSXRay.express.closeSegment();
}

// Custom subsegment for database queries
export async function traceQuery<T>(
  name: string,
  query: () => Promise<T>
): Promise<T> {
  const segment = AWSXRay.getSegment();
  const subsegment = segment?.addNewSubsegment(name);
  
  try {
    const result = await query();
    subsegment?.close();
    return result;
  } catch (error) {
    subsegment?.addError(error as Error);
    subsegment?.close();
    throw error;
  }
}
```

---

## 8. Disaster Recovery Architecture

### 8.1 Multi-Region Failover

**DR Region**: ap-south-1 (Mumbai, India)

**Failover Strategy**:

```
┌─────────────────────────────────────────────────────────────┐
│  NORMAL OPERATION (ap-southeast-1 Primary)                  │
│                                                              │
│  Route 53 (Health Check) → ap-southeast-1 ALB              │
│                             │                                │
│                             ├─→ ECS Cluster (Active)        │
│                             ├─→ RDS Primary (Active)        │
│                             └─→ S3 (Replicating to Mumbai)  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  FAILOVER (ap-south-1 DR)                                    │
│                                                              │
│  Route 53 (Health Check Failed) → ap-south-1 ALB           │
│                                    │                         │
│                                    ├─→ ECS Cluster (Standby)│
│                                    ├─→ RDS Restored         │
│                                    └─→ S3 Replica (Primary) │
└─────────────────────────────────────────────────────────────┘
```

**Route 53 Failover**:
```hcl
# terraform/route53.tf
resource "aws_route53_record" "api_primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.jibonflow.com"
  type    = "A"
  
  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
  
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  health_check_id = aws_route53_health_check.api.id
}

resource "aws_route53_record" "api_secondary" {
  provider = aws.mumbai  # DR region
  
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.jibonflow.com"
  type    = "A"
  
  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
  
  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
}

resource "aws_route53_health_check" "api" {
  fqdn              = "api.jibonflow.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
  
  tags = {
    Name = "jibonflow-api-health-check"
  }
}
```

---

## 9. Cost Estimation

### 9.1 Monthly Infrastructure Cost (Year 1)

| Service | Configuration | Monthly Cost (USD) | Notes |
|---------|---------------|-------------------|-------|
| **ECS Fargate** | 24 tasks × 0.25 vCPU × 0.5 GB × 730 hrs | $175 | Baseline capacity |
| **RDS PostgreSQL** | db.r6g.xlarge × 730 hrs | $350 | Multi-AZ included |
| **RDS Storage** | 500 GB gp3 | $65 | |
| **RDS Backups** | 500 GB × 30 days | $50 | |
| **ElastiCache Redis** | cache.r6g.large × 6 nodes × 730 hrs | $525 | Cluster mode enabled |
| **ALB** | 1 ALB × 730 hrs + 1M LCU | $40 | |
| **S3 Storage** | 500 GB Standard + 500 GB Glacier | $15 | |
| **S3 Requests** | 10M PUT/GET requests | $50 | |
| **CloudFront** | 10 TB data transfer | $850 | Bangladesh region |
| **Route 53** | 1 hosted zone + 10M queries | $10 | |
| **CloudWatch** | 100 GB logs + 50 custom metrics | $150 | |
| **KMS** | 4 keys + 100K requests | $5 | |
| **Secrets Manager** | 20 secrets | $40 | |
| **Data Transfer** | 5 TB outbound | $450 | |
| **Total** | | **$2,775/month** | **$33,300/year** |

**Cost Optimization Opportunities**:
- Use Savings Plans (1-year commitment): Save 30% = $9,990/year
- Use Fargate Spot for non-critical tasks: Save $50/month
- Enable S3 Intelligent-Tiering: Save $200/month

---

## 10. Infrastructure as Code (Terraform)

### 10.1 Project Structure

```
terraform/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── dev/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs/
│   │   ├── cluster.tf
│   │   ├── services.tf
│   │   ├── task-definitions.tf
│   │   └── autoscaling.tf
│   ├── rds/
│   │   ├── main.tf
│   │   ├── parameter-groups.tf
│   │   └── backups.tf
│   ├── elasticache/
│   │   ├── main.tf
│   │   └── parameter-groups.tf
│   ├── s3/
│   │   ├── buckets.tf
│   │   └── policies.tf
│   ├── alb/
│   │   ├── main.tf
│   │   └── target-groups.tf
│   └── monitoring/
│       ├── cloudwatch.tf
│       ├── alarms.tf
│       └── sns-topics.tf
└── README.md
```

### 10.2 Deployment Workflow

**CI/CD Pipeline** (GitHub Actions):

```yaml
# .github/workflows/terraform-deploy.yml
name: Terraform Deploy

on:
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0
      
      - name: Terraform Init
        run: terraform init
        working-directory: terraform/environments/production
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: terraform/environments/production
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
        working-directory: terraform/environments/production
```

---

**Generated**: 2025-10-11  
**Phase**: Architecture Documentation (Day 1-2) - COMPLETE  
**Next**: Merge all sections into final architecture blueprint  
**Invocation Tag**: jibonflow-bootstrap-2025-10-11
