# Deployment Guide

## Overview

This guide covers deploying the AetherWeb3 Multi-EVM Gateway to production environments. We'll cover Google Cloud Run deployment, environment configuration, monitoring setup, and best practices for production operations.

## Prerequisites

- Google Cloud Platform account with billing enabled
- Google Cloud SDK installed and configured
- Docker installed locally
- Domain name for custom domain (optional)
- SSL certificate for HTTPS (handled by Cloud Run)

## Google Cloud Run Deployment

### 1. Project Setup

```bash
# Set your project ID
export PROJECT_ID="your-project-id"
export REGION="us-central1"
export SERVICE_NAME="aetherweb3-gateway"

# Enable required APIs
gcloud services enable run.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable firestore.googleapis.com
gcloud services enable secretmanager.googleapis.com
```

### 2. Environment Configuration

Create environment variables in Google Secret Manager:

```bash
# Create secrets for sensitive configuration
gcloud secrets create jwt-secret --data-file=- <<< "your-jwt-secret-here"
gcloud secrets create stripe-secret-key --data-file=- <<< "sk_live_your_stripe_key"
gcloud secrets create ethereum-rpc-url --data-file=- <<< "https://your-ethereum-rpc-url"
gcloud secrets create base-rpc-url --data-file=- <<< "https://your-base-rpc-url"
gcloud secrets create arbitrum-rpc-url --data-file=- <<< "https://your-arbitrum-rpc-url"

# Create service account for the application
gcloud iam service-accounts create aetherweb3-service \
  --display-name="AetherWeb3 Service Account"

# Grant necessary permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:aetherweb3-service@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:aetherweb3-service@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/datastore.user"
```

### 3. Container Build and Push

Create a production Dockerfile:

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY . .

USER nodejs

EXPOSE 8080

CMD ["node", "src/app.js"]
```

Build and push to Container Registry:

```bash
# Build the image
gcloud builds submit --tag gcr.io/$PROJECT_ID/$SERVICE_NAME

# Alternative: Build locally and push
docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME .
docker push gcr.io/$PROJECT_ID/$SERVICE_NAME
```

### 4. Deploy to Cloud Run

```bash
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$PROJECT_ID/$SERVICE_NAME \
  --platform managed \
  --region $REGION \
  --service-account aetherweb3-service@$PROJECT_ID.iam.gserviceaccount.com \
  --set-env-vars "NODE_ENV=production,PORT=8080,PROJECT_ID=$PROJECT_ID" \
  --set-secrets "JWT_SECRET=jwt-secret:latest,STRIPE_SECRET_KEY=stripe-secret-key:latest,ETHEREUM_RPC_URL=ethereum-rpc-url:latest,BASE_RPC_URL=base-rpc-url:latest,ARBITRUM_RPC_URL=arbitrum-rpc-url:latest" \
  --memory 2Gi \
  --cpu 2 \
  --concurrency 100 \
  --max-instances 50 \
  --allow-unauthenticated \
  --port 8080
```

### 5. Custom Domain Configuration

Set up a custom domain:

```bash
# Map custom domain
gcloud run domain-mappings create \
  --service $SERVICE_NAME \
  --domain api.aetherweb3.xyz \
  --region $REGION

# Get DNS configuration
gcloud run domain-mappings describe \
  --domain api.aetherweb3.xyz \
  --region $REGION
```

Add the required DNS records to your domain:
```
CNAME api.aetherweb3.xyz -> ghs.googlehosted.com
```

## Environment Configuration

### Environment Variables

| Variable | Description | Required | Example |
|----------|-------------|----------|---------|
| NODE_ENV | Environment mode | Yes | production |
| PORT | Server port | Yes | 8080 |
| PROJECT_ID | GCP Project ID | Yes | my-project-123 |
| JWT_SECRET | JWT signing secret | Yes | (from Secret Manager) |
| STRIPE_SECRET_KEY | Stripe API key | Yes | (from Secret Manager) |
| REDIS_URL | Redis connection URL | No | redis://redis-server:6379 |
| LOG_LEVEL | Logging level | No | info |

### Resource Configuration

```yaml
# cloud-run-config.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: aetherweb3-gateway
  annotations:
    run.googleapis.com/ingress: all
    run.googleapis.com/execution-environment: gen2
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "50"
        autoscaling.knative.dev/minScale: "2"
        run.googleapis.com/cpu-throttling: "false"
        run.googleapis.com/memory: "2Gi"
        run.googleapis.com/cpu: "2"
    spec:
      containerConcurrency: 100
      timeoutSeconds: 300
      serviceAccountName: aetherweb3-service@project-id.iam.gserviceaccount.com
      containers:
      - image: gcr.io/project-id/aetherweb3-gateway
        ports:
        - containerPort: 8080
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "8080"
        resources:
          limits:
            cpu: 2000m
            memory: 2Gi
```

## Database Setup

### Firestore Configuration

```javascript
// Configure Firestore for production
const { Firestore } = require('@google-cloud/firestore');

const firestore = new Firestore({
  projectId: process.env.PROJECT_ID,
  databaseId: '(default)', // or your database ID
});

// Create required collections and indexes
async function setupDatabase() {
  // Users collection
  const usersRef = firestore.collection('users');
  
  // Create indexes for common queries
  // These need to be created via Firebase Console or gcloud CLI
  console.log('Ensure these indexes exist in Firestore:');
  console.log('- users: apiKey (ascending)');
  console.log('- users: email (ascending)');
  console.log('- usage: userId (ascending), timestamp (descending)');
  console.log('- apiKeys: keyHash (ascending), active (ascending)');
}
```

Create Firestore indexes:

```bash
# Create index for API key lookups
gcloud firestore indexes composite create \
  --collection-group=users \
  --field-config field-path=apiKey,order=ascending \
  --field-config field-path=active,order=ascending

# Create index for usage tracking
gcloud firestore indexes composite create \
  --collection-group=usage \
  --field-config field-path=userId,order=ascending \
  --field-config field-path=timestamp,order=descending
```

### Redis Setup (Optional)

For rate limiting and caching:

```bash
# Create Redis instance using Memorystore
gcloud redis instances create aetherweb3-cache \
  --size=1 \
  --region=$REGION \
  --redis-version=redis_6_x \
  --network=default

# Get connection details
gcloud redis instances describe aetherweb3-cache \
  --region=$REGION
```

## Monitoring and Logging

### Cloud Monitoring Setup

```bash
# Enable monitoring API
gcloud services enable monitoring.googleapis.com

# Create notification channels
gcloud alpha monitoring channels create \
  --channel-content-from-file=notification-channel.json
```

Notification channel configuration:

```json
{
  "type": "email",
  "displayName": "AetherWeb3 Alerts",
  "labels": {
    "email_address": "alerts@nibertinvestments.com"
  }
}
```

### Alerting Policies

```yaml
# alerting-policy.yaml
displayName: "High Error Rate"
conditions:
  - displayName: "Error rate > 5%"
    conditionThreshold:
      filter: 'resource.type="cloud_run_revision" AND resource.labels.service_name="aetherweb3-gateway"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 0.05
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_RATE
          crossSeriesReducer: REDUCE_MEAN
          groupByFields:
            - "resource.label.service_name"
notificationChannels:
  - "projects/PROJECT_ID/notificationChannels/CHANNEL_ID"
```

### Custom Metrics

```javascript
// src/monitoring/metrics.js
const monitoring = require('@google-cloud/monitoring');

class MetricsCollector {
  constructor(projectId) {
    this.client = new monitoring.MetricServiceClient();
    this.projectId = projectId;
  }

  async recordAPICall(network, method, duration, success) {
    const metricDescriptor = {
      type: 'custom.googleapis.com/api/call_duration',
      metricKind: 'GAUGE',
      valueType: 'DOUBLE',
      description: 'API call duration in milliseconds'
    };

    const point = {
      interval: {
        endTime: {
          seconds: Date.now() / 1000
        }
      },
      value: {
        doubleValue: duration
      }
    };

    const timeSeries = {
      metric: {
        type: metricDescriptor.type,
        labels: {
          network,
          method,
          success: success.toString()
        }
      },
      resource: {
        type: 'cloud_run_revision',
        labels: {
          project_id: this.projectId,
          service_name: 'aetherweb3-gateway'
        }
      },
      points: [point]
    };

    const request = {
      name: this.client.projectPath(this.projectId),
      timeSeries: [timeSeries]
    };

    await this.client.createTimeSeries(request);
  }
}

module.exports = MetricsCollector;
```

### Log Management

```javascript
// src/logging/logger.js
const { Logging } = require('@google-cloud/logging');

class Logger {
  constructor() {
    this.logging = new Logging();
    this.log = this.logging.log('aetherweb3-gateway');
  }

  info(message, metadata = {}) {
    const entry = this.log.entry({
      severity: 'INFO',
      resource: {
        type: 'cloud_run_revision'
      }
    }, {
      message,
      ...metadata,
      timestamp: new Date().toISOString()
    });

    this.log.write(entry);
  }

  error(message, error = null, metadata = {}) {
    const entry = this.log.entry({
      severity: 'ERROR',
      resource: {
        type: 'cloud_run_revision'
      }
    }, {
      message,
      error: error ? {
        message: error.message,
        stack: error.stack
      } : null,
      ...metadata,
      timestamp: new Date().toISOString()
    });

    this.log.write(entry);
  }

  audit(action, userId, details = {}) {
    const entry = this.log.entry({
      severity: 'NOTICE',
      resource: {
        type: 'cloud_run_revision'
      }
    }, {
      eventType: 'AUDIT',
      action,
      userId,
      details,
      timestamp: new Date().toISOString()
    });

    this.log.write(entry);
  }
}

module.exports = new Logger();
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  release:
    types: [published]

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  SERVICE_NAME: aetherweb3-gateway
  REGION: us-central1

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
        run: npm test
        
      - name: Run security audit
        run: npm audit --audit-level=high
        
      - name: Run linting
        run: npm run lint

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Google Cloud
        uses: google-github-actions/setup-gcloud@v0
        with:
          service_account_key: ${{ secrets.GCP_SA_KEY }}
          project_id: ${{ secrets.GCP_PROJECT_ID }}
          
      - name: Configure Docker
        run: gcloud auth configure-docker
        
      - name: Build image
        run: |
          docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA .
          docker tag gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA gcr.io/$PROJECT_ID/$SERVICE_NAME:latest
          
      - name: Push image
        run: |
          docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA
          docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:latest
          
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy $SERVICE_NAME \
            --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA \
            --region $REGION \
            --platform managed \
            --allow-unauthenticated \
            --service-account aetherweb3-service@$PROJECT_ID.iam.gserviceaccount.com \
            --set-env-vars "NODE_ENV=production,GITHUB_SHA=$GITHUB_SHA"
            
      - name: Run post-deployment tests
        run: |
          SERVICE_URL=$(gcloud run services describe $SERVICE_NAME --region $REGION --format 'value(status.url)')
          curl -f $SERVICE_URL/health || exit 1
```

### Automated Testing

```javascript
// tests/deployment/health-check.js
const axios = require('axios');

async function healthCheck(baseUrl) {
  try {
    // Test health endpoint
    const healthResponse = await axios.get(`${baseUrl}/health`);
    if (healthResponse.status !== 200) {
      throw new Error('Health check failed');
    }

    // Test API functionality
    const apiResponse = await axios.post(`${baseUrl}/ethereum`, {
      jsonrpc: '2.0',
      method: 'eth_blockNumber',
      params: [],
      id: 1
    }, {
      headers: {
        'X-API-Key': process.env.TEST_API_KEY,
        'Content-Type': 'application/json'
      }
    });

    if (apiResponse.status !== 200 || !apiResponse.data.result) {
      throw new Error('API test failed');
    }

    console.log('✅ Deployment health check passed');
    return true;
  } catch (error) {
    console.error('❌ Deployment health check failed:', error.message);
    return false;
  }
}

module.exports = healthCheck;
```

## Security Configuration

### Network Security

```bash
# Create VPC for secure networking (optional)
gcloud compute networks create aetherweb3-vpc --subnet-mode=custom

gcloud compute networks subnets create aetherweb3-subnet \
  --network=aetherweb3-vpc \
  --range=10.0.0.0/24 \
  --region=$REGION

# Configure Cloud NAT for outbound internet access
gcloud compute routers create aetherweb3-router \
  --network=aetherweb3-vpc \
  --region=$REGION

gcloud compute routers nats create aetherweb3-nat \
  --router=aetherweb3-router \
  --region=$REGION \
  --nat-all-subnet-ip-ranges
```

### SSL/TLS Configuration

Cloud Run automatically handles SSL certificates for custom domains. For additional security:

```bash
# Force HTTPS redirects (handled by Cloud Run)
# Configure HSTS headers in application
```

Application security headers:

```javascript
// src/middleware/security.js
const helmet = require('helmet');

const securityMiddleware = helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
});

module.exports = securityMiddleware;
```

## Performance Optimization

### Auto-scaling Configuration

```yaml
# Optimized scaling settings
metadata:
  annotations:
    autoscaling.knative.dev/maxScale: "100"
    autoscaling.knative.dev/minScale: "2"
    autoscaling.knative.dev/targetConcurrencyUtilization: "70"
    run.googleapis.com/execution-environment: gen2
```

### Resource Optimization

```dockerfile
# Multi-stage build for smaller images
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS runtime
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
WORKDIR /app

COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./

USER nodejs
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

CMD ["node", "dist/app.js"]
```

## Backup and Disaster Recovery

### Automated Backups

```bash
# Set up Firestore backup
gcloud firestore databases create --database backup-db --location $REGION

# Create backup schedule
gcloud scheduler jobs create app-engine backup-firestore \
  --schedule="0 2 * * *" \
  --time-zone="UTC" \
  --uri="https://firestore.googleapis.com/v1/projects/$PROJECT_ID/databases/(default):exportDocuments" \
  --http-method=POST \
  --headers="Authorization=Bearer $(gcloud auth print-access-token)" \
  --message-body='{"outputUriPrefix":"gs://backup-bucket/firestore"}'
```

### Disaster Recovery Plan

```yaml
# disaster-recovery.yml
recovery_procedures:
  rto: 15 minutes  # Recovery Time Objective
  rpo: 1 hour      # Recovery Point Objective
  
  procedures:
    - name: "Service Outage"
      steps:
        - "Check Cloud Run service status"
        - "Review error logs in Cloud Logging"
        - "Scale up instances if needed"
        - "Deploy previous working version"
        
    - name: "Database Corruption"
      steps:
        - "Restore from latest Firestore backup"
        - "Verify data integrity"
        - "Update application configuration"
        - "Restart services"
```

## Maintenance

### Regular Tasks

```bash
#!/bin/bash
# maintenance.sh - Run weekly maintenance tasks

# Update base images
gcloud builds submit --config cloudbuild-update.yaml

# Check SSL certificate expiry
gcloud run domain-mappings list --region $REGION

# Review resource usage
gcloud run services describe $SERVICE_NAME --region $REGION --format="table(status.traffic)"

# Clean up old container images
gcloud container images list-tags gcr.io/$PROJECT_ID/$SERVICE_NAME \
  --limit=10 --sort-by=~TIMESTAMP \
  --format="get(digest)" | tail -n +6 | \
  xargs -I {} gcloud container images delete gcr.io/$PROJECT_ID/$SERVICE_NAME@{} --quiet
```

### Update Procedures

```bash
# Zero-downtime deployment
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$PROJECT_ID/$SERVICE_NAME:new-version \
  --region $REGION \
  --no-traffic  # Deploy without traffic

# Test new version
gcloud run services update-traffic $SERVICE_NAME \
  --to-revisions new-revision=10 \
  --region $REGION

# If tests pass, route all traffic
gcloud run services update-traffic $SERVICE_NAME \
  --to-latest \
  --region $REGION
```

## Troubleshooting

### Common Deployment Issues

1. **Build Failures**
```bash
# Check build logs
gcloud builds log BUILD_ID

# Common fixes
- Verify Dockerfile syntax
- Check package.json dependencies
- Ensure all files are included in build context
```

2. **Service Start Failures**
```bash
# Check service logs
gcloud logs read "resource.type=cloud_run_revision AND resource.labels.service_name=aetherweb3-gateway" --limit 50

# Common fixes
- Verify environment variables
- Check port configuration (must be 8080)
- Validate service account permissions
```

3. **Performance Issues**
```bash
# Monitor resource usage
gcloud run services describe $SERVICE_NAME --region $REGION --format="table(status.traffic)"

# Scaling troubleshooting
- Increase memory allocation
- Adjust concurrency settings
- Check for memory leaks
```

---

*Last Updated: January 2025*