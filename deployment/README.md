---
status: "🚧 Planned"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - devops-agent-v2
  - documentation-agent-v2
related_services:
  - src/deploy/orchestrator.ts
  - scripts/final-validation.sh
verification_notes: >
  No production or staging deployment evidence was identified; historical runbooks
  have been preserved in the archive pending validation.
---

🚧 **Planned** — Deployment procedures are awaiting verified runbooks and
environment evidence before promotion.

# Deployment Overview

## Scope
- Maintain a curated index for deployment materials now stored under
  `docs/archive/deployment/`.
- Track the verification backlog required to republish executable deployment
  guidance.

## Evidence Reviewed
- Verified SpecKit deployment orchestration code in
  `src/deploy/orchestrator.ts` and supporting types in `src/deploy/plan.ts`.
- Confirmed CI guard-rail scripts such as `scripts/final-validation.sh` and
  `scripts/quality-gates/` utilities that enforce policy compliance.

## Verification Gaps
1. Capture recent CI/CD pipeline runs (GitHub Actions or alternative) that prove
   successful container builds and deployments.
2. Provide infrastructure-as-code repositories or modules demonstrating how AWS
   environments are provisioned.
3. Attach rollback test logs to prove blue/green or canary deployment flows.

## Archive Index
- `docs/archive/deployment/DEPLOYMENT_*`
- `docs/archive/deployment/FINAL_DEPLOYMENT_MANIFEST.md`
- `docs/archive/deployment/CI_CD_SECURITY_GATES_CONFIGURATION.md`
- `docs/archive/deployment/DOCKER_DEPLOYMENT_FIXES_LOG.md`
- `docs/archive/deployment/MULTI_AZ_DEPLOYMENT_SPEC.md`

> The archived documents retain historical knowledge but should not be used as
> step-by-step guidance until the verification gaps above are closed.

## Next Actions
- Coordinate with the DevOps agent to regenerate deployment runbooks based on
  live pipeline telemetry.
- Store deployment evidence under `logs/deployment/` and link it from this file
  during the next review.
- Update this README to ✅ once validated scripts, infrastructure manifests, and
  rollback exercises are documented.

## 🐳 Container Setup

### 1. Docker Images

#### Base Dockerfile Template
```dockerfile
# Base image for Node.js services
FROM node:18-alpine AS base

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./
COPY turbo.json ./

# Install dependencies
FROM base AS deps
RUN npm ci --only=production && npm cache clean --force

# Build stage
FROM base AS build
COPY . .
RUN npm ci
RUN npx turbo build

# Production stage
FROM base AS runtime
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=build --chown=nextjs:nodejs /app/dist ./dist
COPY --from=build --chown=nextjs:nodejs /app/public ./public
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nextjs:nodejs /app/package.json ./package.json

USER nextjs

EXPOSE 3000

ENV NODE_ENV=production
ENV PORT=3000

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["npm", "start"]
```

#### Service-Specific Dockerfiles

```dockerfile
# Patient Portal Dockerfile
FROM node:18-alpine AS base
WORKDIR /app

# Copy source code
COPY apps/patient-portal/ ./
COPY packages/ ../packages/

# Install dependencies
RUN npm ci --only=production

# Build application
RUN npm run build

EXPOSE 3000
CMD ["npm", "start"]
```

```dockerfile
# Auth Service Dockerfile  
FROM node:18-alpine AS base
WORKDIR /app

# Copy source code
COPY services/auth-service/ ./
COPY packages/ ../packages/

# Install dependencies
RUN npm ci --only=production

# Build TypeScript
RUN npm run build

EXPOSE 4000
CMD ["npm", "start"]
```

### 2. Build and Push Images

#### Build Script
```bash
#!/bin/bash
# scripts/build-images.sh

set -e

AWS_REGION=${AWS_REGION:-ap-south-1}
AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID:-123456789012}
PROJECT_NAME=${PROJECT_NAME:-jibonflow}
ENVIRONMENT=${ENVIRONMENT:-production}

# Authenticate Docker to ECR
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Build and push frontend applications
FRONTEND_APPS=("patient-portal" "provider-console" "pharmacy-portal" "pharma-portal" "chw-companion" "admin-console")

for app in "${FRONTEND_APPS[@]}"; do
  echo "Building $app..."
  
  # Build image
  docker build -t $PROJECT_NAME-$app:latest -f apps/$app/Dockerfile .
  
  # Tag for ECR
  docker tag $PROJECT_NAME-$app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$app:latest
  docker tag $PROJECT_NAME-$app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$app:$ENVIRONMENT
  
  # Push to ECR
  docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$app:latest
  docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$app:$ENVIRONMENT
done

# Build and push backend services
BACKEND_SERVICES=("auth-service" "patient-management-service" "telemedicine-service" "payment-service" "medicine-verification" "logistics-tracking" "loyalty-rewards" "notification-service" "audit-logging" "telemedicine-db-service")

for service in "${BACKEND_SERVICES[@]}"; do
  echo "Building $service..."
  
  # Build image
  docker build -t $PROJECT_NAME-$service:latest -f services/$service/Dockerfile .
  
  # Tag for ECR
  docker tag $PROJECT_NAME-$service:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$service:latest
  docker tag $PROJECT_NAME-$service:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$service:$ENVIRONMENT
  
  # Push to ECR
  docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$service:latest
  docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-$service:$ENVIRONMENT
done

echo "All images built and pushed successfully!"
```

#### Execute Build
```bash
# Make script executable
chmod +x scripts/build-images.sh

# Build and push all images
./scripts/build-images.sh
```

---

## ⚙️ ECS Infrastructure

### 1. ECS Cluster Setup

#### ECS Cluster with CDK
```typescript
// infrastructure/ecs-stack.ts
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';

export class EcsStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, vpc: ec2.Vpc, props?: cdk.StackProps) {
    super(scope, id, props);

    // ECS Cluster
    const cluster = new ecs.Cluster(this, 'JibonFlowCluster', {
      vpc,
      clusterName: 'jibonflow-production',
      containerInsights: true,
    });

    // Application Load Balancer
    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc,
      internetFacing: true,
      loadBalancerName: 'jibonflow-alb',
    });

    // HTTPS Listener
    const httpsListener = alb.addListener('HttpsListener', {
      port: 443,
      certificates: [/* SSL certificate */],
      defaultAction: elbv2.ListenerAction.fixedResponse(404),
    });

    // HTTP to HTTPS redirect
    alb.addListener('HttpListener', {
      port: 80,
      defaultAction: elbv2.ListenerAction.redirect({
        protocol: 'HTTPS',
        port: '443',
        permanent: true,
      }),
    });

    return { cluster, alb, httpsListener };
  }
}
```

### 2. Service Definitions

#### Frontend Service Template
```typescript
// infrastructure/frontend-service.ts
export function createFrontendService(
  scope: cdk.Stack,
  serviceName: string,
  cluster: ecs.Cluster,
  listener: elbv2.ApplicationListener,
  vpc: ec2.Vpc
) {
  // Task Definition
  const taskDef = new ecs.FargateTaskDefinition(scope, `${serviceName}TaskDef`, {
    memoryLimitMiB: 512,
    cpu: 256,
  });

  // Container Definition
  const container = taskDef.addContainer(`${serviceName}Container`, {
    image: ecs.ContainerImage.fromRegistry(
      `${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/jibonflow-${serviceName}:production`
    ),
    memoryLimitMiB: 512,
    environment: {
      NODE_ENV: 'production',
      PORT: '3000',
    },
    secrets: {
      DATABASE_URL: ecs.Secret.fromSecretsManager(/* secret */),
      REDIS_URL: ecs.Secret.fromSecretsManager(/* secret */),
    },
    logging: ecs.LogDrivers.awsLogs({
      streamPrefix: serviceName,
      logRetention: logs.RetentionDays.ONE_MONTH,
    }),
  });

  container.addPortMappings({
    containerPort: 3000,
    protocol: ecs.Protocol.TCP,
  });

  // ECS Service
  const service = new ecs.FargateService(scope, `${serviceName}Service`, {
    cluster,
    taskDefinition: taskDef,
    desiredCount: 2,
    assignPublicIp: false,
    vpcSubnets: {
      subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
    },
    healthCheckGracePeriod: cdk.Duration.minutes(5),
  });

  // Target Group
  const targetGroup = new elbv2.ApplicationTargetGroup(scope, `${serviceName}TG`, {
    vpc,
    port: 3000,
    protocol: elbv2.ApplicationProtocol.HTTP,
    targets: [service],
    healthCheck: {
      path: '/health',
      healthyHttpCodes: '200',
      interval: cdk.Duration.seconds(30),
      timeout: cdk.Duration.seconds(10),
      healthyThresholdCount: 2,
      unhealthyThresholdCount: 5,
    },
  });

  // Add listener rule
  listener.addTargetGroups(`${serviceName}Rule`, {
    priority: 100, // Adjust priority as needed
    conditions: [
      elbv2.ListenerCondition.hostHeaders([`${serviceName}.jibonflow.com`]),
    ],
    targetGroups: [targetGroup],
  });

  // Auto Scaling
  const scaling = service.autoScaleTaskCount({
    minCapacity: 2,
    maxCapacity: 10,
  });

  scaling.scaleOnCpuUtilization(`${serviceName}CpuScaling`, {
    targetUtilizationPercent: 70,
    scaleInCooldown: cdk.Duration.minutes(5),
    scaleOutCooldown: cdk.Duration.minutes(2),
  });

  scaling.scaleOnMemoryUtilization(`${serviceName}MemoryScaling`, {
    targetUtilizationPercent: 80,
    scaleInCooldown: cdk.Duration.minutes(5),
    scaleOutCooldown: cdk.Duration.minutes(2),
  });

  return service;
}
```

#### Backend Service Template
```typescript
// infrastructure/backend-service.ts
export function createBackendService(
  scope: cdk.Stack,
  serviceName: string,
  cluster: ecs.Cluster,
  vpc: ec2.Vpc,
  port: number
) {
  // Task Definition with higher resources for backend
  const taskDef = new ecs.FargateTaskDefinition(scope, `${serviceName}TaskDef`, {
    memoryLimitMiB: 1024,
    cpu: 512,
  });

  // Container Definition
  const container = taskDef.addContainer(`${serviceName}Container`, {
    image: ecs.ContainerImage.fromRegistry(
      `${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/jibonflow-${serviceName}:production`
    ),
    memoryLimitMiB: 1024,
    environment: {
      NODE_ENV: 'production',
      PORT: port.toString(),
    },
    secrets: {
      DATABASE_URL: ecs.Secret.fromSecretsManager(/* database secret */),
      REDIS_URL: ecs.Secret.fromSecretsManager(/* redis secret */),
      JWT_SECRET: ecs.Secret.fromSecretsManager(/* jwt secret */),
      AGORA_APP_ID: ecs.Secret.fromSecretsManager(/* agora secret */),
    },
    logging: ecs.LogDrivers.awsLogs({
      streamPrefix: serviceName,
      logRetention: logs.RetentionDays.THREE_MONTHS,
    }),
  });

  container.addPortMappings({
    containerPort: port,
    protocol: ecs.Protocol.TCP,
  });

  // ECS Service
  const service = new ecs.FargateService(scope, `${serviceName}Service`, {
    cluster,
    taskDefinition: taskDef,
    desiredCount: 3, // Higher count for backend services
    assignPublicIp: false,
    vpcSubnets: {
      subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
    },
    healthCheckGracePeriod: cdk.Duration.minutes(5),
    enableLogging: true,
  });

  // Service Discovery
  const namespace = ecs.PrivateDnsNamespace.fromPrivateDnsNamespaceAttributes(
    scope,
    'Namespace',
    {
      namespaceId: 'ns-xxxxx',
      namespaceName: 'jibonflow.local',
      namespaceArn: 'arn:aws:servicediscovery:region:account:namespace/ns-xxxxx',
    }
  );

  service.enableCloudMap({
    name: serviceName,
    cloudMapNamespace: namespace,
    dnsRecordType: ecs.DnsRecordType.A,
  });

  // Auto Scaling with more aggressive scaling for backend
  const scaling = service.autoScaleTaskCount({
    minCapacity: 3,
    maxCapacity: 20,
  });

  scaling.scaleOnCpuUtilization(`${serviceName}CpuScaling`, {
    targetUtilizationPercent: 60, // Lower threshold for backend
    scaleInCooldown: cdk.Duration.minutes(10),
    scaleOutCooldown: cdk.Duration.minutes(3),
  });

  scaling.scaleOnMemoryUtilization(`${serviceName}MemoryScaling`, {
    targetUtilizationPercent: 70,
    scaleInCooldown: cdk.Duration.minutes(10),
    scaleOutCooldown: cdk.Duration.minutes(3),
  });

  return service;
}
```

---

## 🚀 Deployment Process

### 1. Staging Environment Deployment

#### Staging Configuration
```bash
# staging environment variables
export ENVIRONMENT=staging
export DOMAIN_NAME=staging.jibonflow.com
export DATABASE_INSTANCE_TYPE=db.t3.medium
export REDIS_NODE_TYPE=cache.t3.micro
export ECS_DESIRED_COUNT=1
```

#### Deploy Staging Infrastructure
```bash
# Deploy infrastructure stacks
npx cdk deploy VpcStack --profile jibonflow-staging
npx cdk deploy DatabaseStack --profile jibonflow-staging
npx cdk deploy RedisStack --profile jibonflow-staging
npx cdk deploy EcsStack --profile jibonflow-staging

# Deploy application services
npx cdk deploy FrontendServicesStack --profile jibonflow-staging
npx cdk deploy BackendServicesStack --profile jibonflow-staging
```

#### Staging Validation
```bash
# Health check script
#!/bin/bash
# scripts/health-check.sh

STAGING_DOMAIN="staging.jibonflow.com"

echo "Running staging environment health checks..."

# Check frontend applications
FRONTEND_APPS=("patient" "provider" "pharmacy" "pharma" "chw" "admin")

for app in "${FRONTEND_APPS[@]}"; do
  echo "Checking $app portal..."
  status=$(curl -s -o /dev/null -w "%{http_code}" https://$app.$STAGING_DOMAIN/health)
  if [ $status -eq 200 ]; then
    echo "✅ $app portal is healthy"
  else
    echo "❌ $app portal returned status $status"
    exit 1
  fi
done

# Check backend services
BACKEND_SERVICES=("auth" "patient-management" "telemedicine" "payment" "medicine" "logistics" "loyalty" "notification" "audit")

for service in "${BACKEND_SERVICES[@]}"; do
  echo "Checking $service service..."
  status=$(curl -s -o /dev/null -w "%{http_code}" https://api.$STAGING_DOMAIN/$service/health)
  if [ $status -eq 200 ]; then
    echo "✅ $service service is healthy"
  else
    echo "❌ $service service returned status $status"
    exit 1
  fi
done

echo "All health checks passed! 🎉"
```

### 2. Production Deployment

#### Pre-Production Checklist
```bash
# Pre-production deployment checklist
echo "Pre-production deployment checklist:"
echo "□ All staging tests passed"
echo "□ Database migration scripts reviewed"
echo "□ SSL certificates configured"
echo "□ Monitoring and alerting configured"
echo "□ Backup strategy implemented"
echo "□ Rollback plan prepared"
echo "□ Security scan completed"
echo "□ Performance testing completed"
echo "□ Change management approval obtained"
```

#### Blue-Green Deployment Strategy
```typescript
// infrastructure/blue-green-deployment.ts
export class BlueGreenDeployment {
  private cluster: ecs.Cluster;
  private listener: elbv2.ApplicationListener;
  
  constructor(cluster: ecs.Cluster, listener: elbv2.ApplicationListener) {
    this.cluster = cluster;
    this.listener = listener;
  }

  async deployService(serviceName: string, newImageTag: string) {
    // 1. Create new task definition with updated image
    const newTaskDef = this.createTaskDefinition(serviceName, newImageTag);
    
    // 2. Create green service
    const greenService = new ecs.FargateService(this.scope, `${serviceName}-Green`, {
      cluster: this.cluster,
      taskDefinition: newTaskDef,
      desiredCount: 0, // Start with 0, will scale up
    });

    // 3. Create green target group
    const greenTargetGroup = new elbv2.ApplicationTargetGroup(this.scope, `${serviceName}-Green-TG`, {
      vpc: this.vpc,
      port: 3000,
      targets: [greenService],
    });

    // 4. Health check green environment
    await this.waitForHealthyTargets(greenTargetGroup);

    // 5. Gradually shift traffic (blue-green with canary)
    await this.shiftTraffic(serviceName, greenTargetGroup, [
      { weight: 10, duration: '5m' },   // 10% traffic for 5 minutes
      { weight: 50, duration: '10m' },  // 50% traffic for 10 minutes
      { weight: 100, duration: '0' },   // 100% traffic
    ]);

    // 6. Monitor error rates and rollback if necessary
    const errorRate = await this.monitorErrorRate(serviceName);
    if (errorRate > 0.01) { // 1% error threshold
      await this.rollback(serviceName);
      throw new Error(`Deployment failed: Error rate ${errorRate} exceeded threshold`);
    }

    // 7. Clean up blue environment
    await this.cleanupBlueEnvironment(serviceName);
  }

  private async shiftTraffic(
    serviceName: string, 
    targetGroup: elbv2.ApplicationTargetGroup, 
    stages: Array<{weight: number, duration: string}>
  ) {
    for (const stage of stages) {
      console.log(`Shifting ${stage.weight}% traffic to green environment`);
      
      // Update listener rules to split traffic
      await this.updateListenerRule(serviceName, {
        blueWeight: 100 - stage.weight,
        greenWeight: stage.weight,
      });

      // Wait for specified duration
      if (stage.duration !== '0') {
        await this.sleep(this.parseDuration(stage.duration));
      }

      // Check health metrics
      const metrics = await this.getHealthMetrics(serviceName);
      if (!metrics.isHealthy) {
        throw new Error('Health check failed during traffic shifting');
      }
    }
  }
}
```

#### Production Deployment Script
```bash
#!/bin/bash
# scripts/deploy-production.sh

set -e

echo "🚀 Starting production deployment..."

# 1. Pre-deployment checks
echo "Running pre-deployment checks..."
./scripts/pre-deployment-checks.sh

# 2. Database migrations
echo "Running database migrations..."
npm run migrate:production

# 3. Build and push new images
echo "Building and pushing container images..."
./scripts/build-images.sh

# 4. Deploy infrastructure updates (if any)
echo "Deploying infrastructure updates..."
npx cdk deploy --all --profile jibonflow-production --require-approval never

# 5. Deploy services with blue-green strategy
echo "Deploying services..."
SERVICES=("patient-portal" "provider-console" "auth-service" "telemedicine-service")

for service in "${SERVICES[@]}"; do
  echo "Deploying $service..."
  aws ecs update-service \
    --cluster jibonflow-production \
    --service $service \
    --force-new-deployment \
    --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"
  
  # Wait for deployment to complete
  aws ecs wait services-stable \
    --cluster jibonflow-production \
    --services $service
  
  echo "✅ $service deployed successfully"
done

# 6. Post-deployment validation
echo "Running post-deployment validation..."
./scripts/health-check.sh

# 7. Update DNS (if needed)
echo "Updating DNS records..."
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://dns-updates.json

echo "🎉 Production deployment completed successfully!"
```

---

## 📊 Monitoring & Observability

### 1. CloudWatch Configuration

#### Custom Metrics and Alarms
```typescript
// infrastructure/monitoring-stack.ts
export class MonitoringStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Application Load Balancer Alarms
    new cloudwatch.Alarm(this, 'ALBHighLatency', {
      metric: new cloudwatch.Metric({
        namespace: 'AWS/ApplicationELB',
        metricName: 'TargetResponseTime',
        dimensionsMap: {
          LoadBalancer: 'app/jibonflow-alb/xxxxx',
        },
        statistic: 'Average',
      }),
      threshold: 1, // 1 second
      evaluationPeriods: 2,
      treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
    });

    // ECS Service Alarms
    new cloudwatch.Alarm(this, 'ECSHighCPU', {
      metric: new cloudwatch.Metric({
        namespace: 'AWS/ECS',
        metricName: 'CPUUtilization',
        dimensionsMap: {
          ServiceName: 'jibonflow-production',
          ClusterName: 'jibonflow-cluster',
        },
        statistic: 'Average',
      }),
      threshold: 80,
      evaluationPeriods: 3,
    });

    // Database Alarms
    new cloudwatch.Alarm(this, 'DatabaseConnections', {
      metric: new cloudwatch.Metric({
        namespace: 'AWS/RDS',
        metricName: 'DatabaseConnections',
        dimensionsMap: {
          DBInstanceIdentifier: 'jibonflow-db',
        },
        statistic: 'Average',
      }),
      threshold: 80, // 80% of max connections
      evaluationPeriods: 2,
    });

    // Custom Business Metrics
    new cloudwatch.Alarm(this, 'ConsultationFailureRate', {
      metric: new cloudwatch.Metric({
        namespace: 'JibonFlow/Telemedicine',
        metricName: 'ConsultationFailureRate',
        statistic: 'Average',
      }),
      threshold: 0.05, // 5% failure rate
      evaluationPeriods: 2,
    });
  }
}
```

### 2. X-Ray Tracing

#### Service Configuration
```typescript
// Enable X-Ray tracing in ECS task definitions
const container = taskDef.addContainer('AppContainer', {
  // ... other configuration
  environment: {
    _X_AMZN_TRACE_ID: ecs.Secret.fromSecretsManager(traceSecret),
    AWS_XRAY_TRACING_NAME: serviceName,
    AWS_XRAY_DEBUG_MODE: 'true',
  },
});

// Add X-Ray daemon sidecar
taskDef.addContainer('XRayDaemon', {
  image: ecs.ContainerImage.fromRegistry('amazon/aws-xray-daemon:latest'),
  memoryLimitMiB: 32,
  cpu: 32,
  essential: false,
  portMappings: [{
    containerPort: 2000,
    protocol: ecs.Protocol.UDP,
  }],
});
```

---

## 🔐 Security Configuration

### 1. IAM Roles and Policies

#### ECS Task Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::jibonflow-assets/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:jibonflow/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:*:*:key/jibonflow-key-id"
    }
  ]
}
```

#### ECS Execution Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

### 2. Secrets Management

#### Database Credentials
```bash
# Create database credentials in Secrets Manager
aws secretsmanager create-secret \
  --name "jibonflow/database/credentials" \
  --description "Database credentials for JibonFlow" \
  --secret-string '{
    "username": "postgres",
    "password": "secure-random-password",
    "host": "jibonflow-db.cluster-xxxxx.ap-south-1.rds.amazonaws.com",
    "port": 5432,
    "database": "jibonflow"
  }'

# Create JWT secrets
aws secretsmanager create-secret \
  --name "jibonflow/auth/jwt-secret" \
  --description "JWT signing secret" \
  --secret-string "very-secure-jwt-secret-key"

# Create external API keys
aws secretsmanager create-secret \
  --name "jibonflow/external/agora" \
  --description "Agora RTC API credentials" \
  --secret-string '{
    "appId": "agora-app-id",
    "appCertificate": "agora-app-certificate"
  }'
```

---

## 🔄 CI/CD Pipeline

### 1. GitHub Actions Workflow

#### Production Deployment Pipeline
```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  AWS_REGION: ap-south-1
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  PROJECT_NAME: jibonflow

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm run test:ci
      
      - name: Run security scan
        run: npm audit --audit-level=high
      
      - name: Run linting
        run: npm run lint

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.PROJECT_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=raw,value=latest
      
      - name: Build and push images
        run: |
          chmod +x scripts/build-images.sh
          ./scripts/build-images.sh

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy infrastructure
        run: |
          npm install -g aws-cdk
          cd infrastructure
          npm ci
          npx cdk deploy --all --require-approval never
      
      - name: Update ECS services
        run: |
          chmod +x scripts/deploy-production.sh
          ./scripts/deploy-production.sh
      
      - name: Run health checks
        run: |
          chmod +x scripts/health-check.sh
          ./scripts/health-check.sh
      
      - name: Notify deployment success
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-type: application/json' \
            --data '{"text":"🎉 JibonFlow production deployment successful!"}'
      
      - name: Notify deployment failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-type: application/json' \
            --data '{"text":"❌ JibonFlow production deployment failed!"}'
```

### 2. Rollback Procedures

#### Automated Rollback Script
```bash
#!/bin/bash
# scripts/rollback.sh

set -e

SERVICE_NAME=$1
TARGET_REVISION=$2

if [ -z "$SERVICE_NAME" ] || [ -z "$TARGET_REVISION" ]; then
  echo "Usage: $0 <service-name> <task-definition-revision>"
  echo "Example: $0 auth-service 42"
  exit 1
fi

echo "🔄 Rolling back $SERVICE_NAME to revision $TARGET_REVISION..."

# Update service to previous task definition
aws ecs update-service \
  --cluster jibonflow-production \
  --service $SERVICE_NAME \
  --task-definition "${SERVICE_NAME}:${TARGET_REVISION}" \
  --force-new-deployment

# Wait for rollback to complete
echo "Waiting for rollback to complete..."
aws ecs wait services-stable \
  --cluster jibonflow-production \
  --services $SERVICE_NAME

# Verify health
echo "Verifying service health..."
./scripts/health-check.sh

echo "✅ Rollback completed successfully!"
```

---

## 🚨 Disaster Recovery

### 1. Backup Strategy

#### Automated Backup Configuration
```typescript
// infrastructure/backup-stack.ts
export class BackupStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Backup Vault
    const backupVault = new backup.BackupVault(this, 'JibonFlowBackupVault', {
      backupVaultName: 'jibonflow-backup-vault',
      encryptionKey: kms.Key.fromKeyArn(this, 'BackupKey', 'arn:aws:kms:...'),
    });

    // Backup Plan
    const backupPlan = backup.BackupPlan.dailyWeeklyMonthly5YearRetention(
      this,
      'JibonFlowBackupPlan'
    );

    // Add RDS to backup plan
    backupPlan.addSelection('DatabaseBackup', {
      resources: [
        backup.BackupResource.fromRdsDatabase(database),
      ],
    });

    // Add S3 to backup plan
    backupPlan.addSelection('S3Backup', {
      resources: [
        backup.BackupResource.fromArn('arn:aws:s3:::jibonflow-assets'),
      ],
    });
  }
}
```

### 2. Cross-Region Replication

#### Database Cross-Region Setup
```bash
# Create read replica in different region
aws rds create-db-instance-read-replica \
  --db-instance-identifier jibonflow-db-replica-singapore \
  --source-db-instance-identifier jibonflow-db-primary \
  --db-instance-class db.r5.large \
  --availability-zone ap-southeast-1a \
  --publicly-accessible false \
  --multi-az false
```

---

## 📋 Troubleshooting Guide

### Common Deployment Issues

#### 1. Container Health Check Failures
```bash
# Check container logs
aws logs get-log-events \
  --log-group-name /ecs/jibonflow-auth-service \
  --log-stream-name ecs/auth-service/task-id

# Check service events
aws ecs describe-services \
  --cluster jibonflow-production \
  --services auth-service \
  --query 'services[0].events'
```

#### 2. Database Connection Issues
```bash
# Test database connectivity
aws rds describe-db-instances \
  --db-instance-identifier jibonflow-db \
  --query 'DBInstances[0].DBInstanceStatus'

# Check security group rules
aws ec2 describe-security-groups \
  --group-ids sg-xxxxx \
  --query 'SecurityGroups[0].IpPermissions'
```

#### 3. Load Balancer Target Health
```bash
# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --query 'TargetHealthDescriptions'
```

### Performance Optimization

#### 1. ECS Task Resource Optimization
```bash
# Monitor task resource utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=auth-service Name=ClusterName,Value=jibonflow-production \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Average
```

#### 2. Database Performance Tuning
```sql
-- Check slow queries
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Check database connections
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

---

## 📞 Support and Maintenance

### Deployment Support Contacts
- **DevOps Team**: devops@jibonflow.com
- **On-call Engineer**: +880-1700-DEVOPS
- **Incident Response**: incidents@jibonflow.com

### Maintenance Windows
- **Staging**: Daily 2:00-4:00 AM BST
- **Production**: Sunday 2:00-6:00 AM BST
- **Emergency**: As needed with 2-hour notice

### Documentation Updates
This deployment guide should be updated:
- After each major deployment
- When infrastructure changes are made
- When new services are added
- Quarterly architecture reviews

---

*Last updated: January 2024 | Version: 2.1.0*
*Next review: April 2024*