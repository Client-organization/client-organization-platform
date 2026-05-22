# Deployment & DevOps Documentation

## Deployment Guide

This document covers deployment procedures, environments, and DevOps practices for the Client Organization Platform.

## Table of Contents

- [Environment Setup](#environment-setup)
- [Pre-Deployment Checklist](#pre-deployment-checklist)
- [Deployment Procedures](#deployment-procedures)
- [Monitoring & Logging](#monitoring--logging)
- [Rollback Procedures](#rollback-procedures)
- [CI/CD Pipeline](#cicd-pipeline)

## Environment Setup

### Development Environment

- **Purpose**: Local development and testing
- **Database**: Local PostgreSQL/MongoDB
- **Configuration**: `.env.development`
- **Services**: Mocked external services

### Staging Environment

- **Purpose**: Pre-production testing
- **Database**: Staging PostgreSQL (separate from production)
- **Configuration**: `.env.staging`
- **Services**: Integration with real APIs
- **Data**: Test data mirroring production structure

### Production Environment

- **Purpose**: Live customer-facing system
- **Database**: Production PostgreSQL with replication
- **Configuration**: `.env.production`
- **Services**: All external services enabled
- **Availability**: 99.9% SLA target
- **Backups**: Automated daily backups with encryption

## Environment Variables

### Common Variables

```env
NODE_ENV=production
PORT=3000
LOG_LEVEL=info

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=client_org_platform
DB_USER=postgres
DB_PASSWORD=secure_password

# Authentication
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRY=3600

# External Services
API_KEY_SERVICE_1=key_here
API_KEY_SERVICE_2=key_here

# Email Configuration
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=no-reply@client-org.com
SMTP_PASSWORD=smtp_password

# AWS/Cloud Services
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
S3_BUCKET_NAME=client-org-assets
```

## Pre-Deployment Checklist

- [ ] All tests passing (unit, integration, e2e)
- [ ] Code review completed and approved
- [ ] Security scan passed
- [ ] Database migrations prepared and tested
- [ ] Documentation updated
- [ ] Environment variables configured
- [ ] Rollback plan prepared
- [ ] Team notified of deployment window
- [ ] Backup taken (production)

## Deployment Procedures

### Via Docker

1. **Build Docker Image**
   ```bash
   docker build -t client-org-platform:v1.0.0 .
   docker tag client-org-platform:v1.0.0 client-org-platform:latest
   ```

2. **Push to Registry**
   ```bash
   docker push client-org-platform:v1.0.0
   docker push client-org-platform:latest
   ```

3. **Deploy**
   ```bash
   docker run -d \
     --name client-org-platform \
     -p 3000:3000 \
     --env-file .env.production \
     -e NODE_ENV=production \
     -v /data/logs:/app/logs \
     -v /data/uploads:/app/uploads \
     client-org-platform:v1.0.0
   ```

### Via Kubernetes

1. **Update Deployment**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: client-org-platform
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: client-org-platform
     template:
       metadata:
         labels:
           app: client-org-platform
       spec:
         containers:
         - name: client-org-platform
           image: client-org-platform:v1.0.0
           ports:
           - containerPort: 3000
           env:
           - name: NODE_ENV
             value: "production"
           envFrom:
           - configMapRef:
               name: app-config
           - secretRef:
               name: app-secrets
           resources:
             requests:
               memory: "256Mi"
               cpu: "250m"
             limits:
               memory: "512Mi"
               cpu: "500m"
           livenessProbe:
             httpGet:
               path: /health
               port: 3000
             initialDelaySeconds: 30
             periodSeconds: 10
   ```

2. **Apply Configuration**
   ```bash
   kubectl apply -f deployment.yaml
   kubectl rollout status deployment/client-org-platform
   ```

### Manual Deployment

1. **SSH to Server**
   ```bash
   ssh deploy@production.server.com
   ```

2. **Pull Latest Code**
   ```bash
   cd /opt/client-org-platform
   git pull origin main
   ```

3. **Install Dependencies**
   ```bash
   npm install --production
   ```

4. **Run Migrations**
   ```bash
   npm run migrate:prod
   ```

5. **Build Application**
   ```bash
   npm run build
   ```

6. **Restart Service**
   ```bash
   sudo systemctl restart client-org-platform
   ```

## Database Migrations

### Pre-Deployment

1. **Test Migration Locally**
   ```bash
   npm run migrate:test
   ```

2. **Backup Database**
   ```bash
   pg_dump -U postgres client_org_platform > backup_$(date +%Y%m%d).sql
   ```

3. **Run Migration**
   ```bash
   npm run migrate:prod
   ```

### Rollback Migration

```bash
npm run migrate:rollback
```

## Monitoring & Logging

### Application Health

- **Health Check Endpoint**: `GET /health`
- **Metrics Endpoint**: `GET /metrics`

### Logging

```javascript
// Info logs
logger.info('User logged in', { userId: '123' });

// Error logs
logger.error('Database connection failed', { error: err });

// Structured logging
logger.log({
  level: 'info',
  message: 'Transaction processed',
  transactionId: '456',
  amount: 1000
});
```

### Log Aggregation

- **Tool**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Retention**: 30 days
- **Real-time Monitoring**: Kibana dashboards

### Alerting

| Alert | Threshold | Action |
|-------|-----------|--------|
| High CPU | > 80% | Page on-call |
| High Memory | > 85% | Page on-call |
| Error Rate | > 5% | Alert team |
| API Latency | > 1000ms | Alert team |
| Disk Space | < 10% | Page on-call |

### Application Performance Monitoring

- **Tool**: New Relic / DataDog
- **Metrics Tracked**:
  - Response times
  - Throughput
  - Error rates
  - Database query performance
  - External API calls

## Rollback Procedures

### Quick Rollback (< 5 minutes)

```bash
# Kubernetes rollback
kubectl rollout undo deployment/client-org-platform

# Docker rollback
docker run -d --name client-org-platform-prev client-org-platform:v0.9.0

# Manual rollback
cd /opt/client-org-platform
git revert HEAD~1
npm install
npm start
```

### Database Rollback

```bash
# Restore from backup
psql -U postgres client_org_platform < backup_20260522.sql

# Rollback migrations
npm run migrate:rollback -- --steps 1
```

## Post-Deployment

1. **Verify Deployment**
   - Check `/health` endpoint
   - Monitor application logs
   - Run smoke tests

2. **Performance Monitoring**
   - Check response times
   - Monitor error rates
   - Verify database performance

3. **User Communication**
   - Announce deployment completion
   - Document changes
   - Update status page

4. **Documentation**
   - Update deployment log
   - Document any issues
   - Note lessons learned

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
      - run: npm install
      - run: npm test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build and Push Docker Image
        run: |
          docker build -t client-org-platform:${{ github.sha }} .
          docker push client-org-platform:${{ github.sha }}
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/client-org-platform \
            client-org-platform=client-org-platform:${{ github.sha }}
```

## Disaster Recovery

### Recovery Time Objective (RTO)

- **Critical Systems**: < 1 hour
- **Standard Services**: < 4 hours

### Recovery Point Objective (RPO)

- **Database**: < 15 minutes (continuous replication)
- **Application**: < 1 hour (hourly snapshots)

### Backup Strategy

- Daily automated backups
- Off-site backup storage (AWS S3)
- Monthly restoration testing
- Encrypted backups

---

**Last Updated**: 2026-05-22

For questions about deployment, contact the DevOps team or see [README.md](../README.md).
