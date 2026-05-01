# eSup-Signature Deployment Guide - Production Grade

**Version:** 2.0  
**Date:** 2026-05-01  
**Status:** Production Ready

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Infrastructure Setup](#infrastructure-setup)
5. [Kubernetes Deployment](#kubernetes-deployment)
6. [Security Configuration](#security-configuration)
7. [CI/CD Pipeline](#cicd-pipeline)
8. [Monitoring & Logging](#monitoring--logging)
9. [Backup & Disaster Recovery](#backup--disaster-recovery)
10. [Troubleshooting](#troubleshooting)
11. [Production Checklist](#production-checklist)

---

## Overview

**eSup-Signature** is an enterprise-grade electronic signature platform designed to digitalize and simplify signature workflows and approval processes.

### Key Features

- ✅ **Digital Signatures** - eIDAS compliant electronic signatures
- ✅ **Workflow Management** - Sequential and parallel approval workflows
- ✅ **Multi-tenant Support** - Multiple organizations in one instance
- ✅ **Audit Trail** - Complete traceability of all actions
- ✅ **Integration** - LDAP, OAuth2, SAML, API
- ✅ **High Availability** - Clustering and redundancy
- ✅ **Open Source** - Apache 2.0 License, community-driven

### Technology Stack

| Component | Technology | Version |
|-----------|-----------|----------|
| **Backend** | Java / Spring Boot | 2.x |
| **Database** | PostgreSQL | 13+ |
| **Caching** | Redis | 6+ |
| **Orchestration** | Kubernetes | 1.24+ |
| **Container Runtime** | Docker/containerd | 20.10+ |
| **Monitoring** | Prometheus | 2.40+ |
| **Logging** | ELK Stack | 8.0+ |
| **Ingress** | NGINX Ingress | 1.24+ |
| **CI/CD** | GitHub Actions | native |

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                  External Users                      │
│              (via HTTPS / Ingress)                   │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │  Ingress Controller     │
        │  + TLS Termination      │
        │  + Load Balancing       │
        └────────────┬────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼─────┐   ┌───▼─────┐
│ Pod 1    │    │ Pod 2     │   │ Pod 3   │
│ eSup-App │    │ eSup-App  │   │ eSup-App│
│ (JVM)    │    │ (JVM)     │   │ (JVM)   │
└────┬─────┘    └─────┬─────┘   └───┬─────┘
     │                │             │
     └────────────────┼─────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    ┌───▼──────┐ ┌───▼───┐   ┌────▼─────┐
    │PostgreSQL│ │ Redis │   │ File     │
    │  (HA)    │ │Cluster│   │Storage(S3)│
    │Primary   │ │       │   │           │
    │+ Replica │ └───────┘   └───────────┘
    └──────────┘
         │
    ┌────▼────────────┐
    │ WAL Archiving   │
    │ (S3 / NFS)      │
    └─────────────────┘

Monitoring Layer:
┌──────────────────────────────────────┐
│ Prometheus │ Grafana │ AlertManager │
└──────────────────────────────────────┘

Logging Layer:
┌────────────────────────────────────┐
│ Fluentd → Elasticsearch → Kibana   │
└────────────────────────────────────┘
```

### Component Responsibilities

#### **eSup-Signature Application (Java/Spring Boot)**
- Signature workflow management
- Document handling
- User authentication & authorization
- API endpoints
- Business logic

#### **PostgreSQL Database**
- Primary data store
- Document metadata
- User accounts
- Workflow states
- Audit logs
- High Availability via streaming replication

#### **Redis Cache**
- Session storage
- Application cache
- Message queue for async tasks
- Cluster mode for resilience

#### **Kubernetes Orchestration**
- Container management
- Service discovery
- Self-healing
- Auto-scaling
- Rolling updates

---

## Prerequisites

### Infrastructure Requirements

#### **Development Environment**
- CPU: 4 cores
- Memory: 8 GB
- Storage: 100 GB
- Network: 10 Mbps minimum

#### **Staging Environment**
- CPU: 8 cores
- Memory: 16 GB
- Storage: 500 GB
- Network: 50 Mbps minimum
- HA: 2 nodes minimum

#### **Production Environment**
- CPU: 16+ cores
- Memory: 32+ GB
- Storage: 2+ TB
- Network: 100 Mbps minimum
- HA: 3+ nodes
- Backup infrastructure (S3, NFS)
- Monitoring infrastructure

### Software Prerequisites

```bash
# Kubernetes
kubectl >= 1.24
helm >= 3.10
kubectl version --client

# Container Runtime
docker >= 20.10 OR containerd >= 1.6
docker --version

# CLI Tools
jq >= 1.6 (for JSON processing)
curl >= 7.68 (for API testing)
openssl >= 1.1.1 (for certificate generation)

# Git
git >= 2.30
git --version
```

### Access & Permissions

```bash
# Kubernetes cluster access
- kubeconfig file configured
- RBAC permissions for namespace creation
- Ability to create persistent volumes
- Container registry access (if using private images)

# Cloud Provider (if applicable)
- AWS: IAM permissions for EKS, RDS, S3, Route53
- Azure: Role assignments for AKS, Database, Storage
- GCP: Project roles for GKE, CloudSQL, Cloud Storage
```

### Domain & SSL/TLS

```bash
# DNS
- Domain name for signature service
- DNS management access
- A/CNAME record creation

# SSL Certificates
- Option 1: Let's Encrypt (automated via cert-manager)
- Option 2: Self-signed (development only)
- Option 3: Company CA (production)

# Certificate Manager
Helm chart: cert-manager
Version: v1.11+
```

---

## Infrastructure Setup

### Step 1: Create Namespace

```bash
kubectl apply -f k8s/01-namespace.yaml

# Verify
kubectl get namespace signature
kubectl describe namespace signature
```

**What this creates:**
- Kubernetes namespace `signature`
- Network Policies (default deny)
- RBAC ServiceAccounts
- ResourceQuotas
- Pod Security Policies

### Step 2: Configure Secrets

**⚠️ SECURITY WARNING:** Secrets in Kubernetes can be dangerous if not properly managed!

#### Generate Strong Secrets

```bash
# Generate database password (32 characters)
DB_PASSWORD=$(openssl rand -base64 32)
echo "Database Password: $DB_PASSWORD"

# Generate JWT secret
JWT_SECRET=$(openssl rand -base64 32)
echo "JWT Secret: $JWT_SECRET"

# Generate Redis password
REDIS_PASSWORD=$(openssl rand -base64 32)
echo "Redis Password: $REDIS_PASSWORD"

# Generate App secret
APP_SECRET=$(openssl rand -base64 32)
echo "App Secret: $APP_SECRET"
```

#### Create Secrets Manifest

```bash
# Edit k8s/02-secrets.yaml and replace all placeholders
vi k8s/02-secrets.yaml

# Base64 encode secrets
echo -n "your-password" | base64

# Apply secrets (use Sealed Secrets for production!)
kubectl apply -f k8s/02-secrets.yaml

# Verify (do NOT display content in production!)
kubectl get secrets -n signature
```

#### Using Sealed Secrets (Recommended for Production)

```bash
# Install Sealed Secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system --version v0.21.0

# Wait for controller to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=sealed-secrets \
  -n kube-system --timeout=300s

# Seal a secret
echo -n 'my-secret-value' | kubectl create secret generic my-secret \
  --dry-run=client \
  --from-file=/dev/stdin \
  -o yaml | \
  kubeseal -f - -w sealed-secret.yaml

# Apply sealed secret
kubectl apply -f sealed-secret.yaml
```

#### Using External Secrets (AWS/Azure/GCP)

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets-system --create-namespace

# Create SecretStore (AWS Secrets Manager example)
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretstore
  namespace: signature
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
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: esup-signature-secrets
  namespace: signature
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretstore
    kind: SecretStore
  target:
    name: esup-signature-secrets-external
    creationPolicy: Owner
  data:
  - secretKey: JWT_SECRET
    remoteRef:
      key: esup-signature/jwt-secret
  - secretKey: APP_SECRET
    remoteRef:
      key: esup-signature/app-secret
EOF
```

### Step 3: Configure ConfigMaps

```bash
# Apply configuration
kubectl apply -f k8s/03-configmap.yaml

# Verify
kubectl get configmaps -n signature
kubectl describe configmap postgres-config -n signature
```

---

## Kubernetes Deployment

### Step 1: Deploy PostgreSQL

```bash
# Create directory for StatefulSet files
mkdir -p k8s/postgres

# Create StatefulSet manifest (see postgres-statefulset.yaml)
kubectl apply -f k8s/postgres/10-statefulset.yaml

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod \
  -l app=postgresql \
  -n signature \
  --timeout=300s

# Verify
kubectl get statefulset -n signature
kubectl get pvc -n signature
kubectl logs -n signature postgresql-0

# Port-forward to test (optional)
kubectl port-forward -n signature postgresql-0 5432:5432
# Connect: psql -h localhost -U esup_user -d esup_signature
```

### Step 2: Deploy Redis

```bash
# Create directory for Redis files
mkdir -p k8s/redis

# Deploy Redis Cluster
kubectl apply -f k8s/redis/10-statefulset.yaml

# Wait for Redis
kubectl wait --for=condition=ready pod \
  -l app=redis \
  -n signature \
  --timeout=300s

# Verify
kubectl get statefulset -n signature
kubectl logs -n signature redis-0
```

### Step 3: Deploy eSup-Signature Application

```bash
# Create directory
mkdir -p k8s/esup-signature

# Deploy eSup-Signature
kubectl apply -f k8s/esup-signature/10-deployment.yaml

# Check deployment status
kubectl rollout status deployment/esup-signature -n signature

# Verify pods are running
kubectl get pods -n signature
kubectl describe pod <pod-name> -n signature
kubectl logs <pod-name> -n signature
```

### Step 4: Configure Ingress

```bash
# Install NGINX Ingress Controller (if not already installed)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values ingress-values.yaml

# Install cert-manager for TLS
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create Certificate Issuer
kubectl apply -f k8s/cert-issuer.yaml

# Deploy Ingress
kubectl apply -f k8s/ingress.yaml

# Verify Ingress
kubectl get ingress -n signature
kubectl describe ingress signature-ingress -n signature
```

---

## Security Configuration

### Network Security

#### Network Policies

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: signature
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow eSup-Signature from Ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: signature
spec:
  podSelector:
    matchLabels:
      app: esup-signature
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080

---
# Allow eSup-Signature to access PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-postgres
  namespace: signature
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: esup-signature
    ports:
    - protocol: TCP
      port: 5432
```

#### Pod Security Policies

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
    seLinuxOptions:
      level: "s0:c123,c456"
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'MustRunAs'
  readOnlyRootFilesystem: true
```

### RBAC Configuration

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: esup-signature
  namespace: signature

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: esup-signature-role
  namespace: signature
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: esup-signature-rolebinding
  namespace: signature
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: esup-signature-role
subjects:
  - kind: ServiceAccount
    name: esup-signature
    namespace: signature
```

### TLS/SSL Configuration

#### Let's Encrypt Integration (Recommended)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: signature-cert
  namespace: signature
spec:
  secretName: signature-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - signature.example.com
```

### Audit Logging

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all requests at the Metadata level
- level: Metadata
  omitStages:
  - RequestReceived
# Log specific actions
- level: RequestResponse
  verbs: ["create", "update", "delete", "patch"]
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  namespaces: ["signature"]
# Log pod exec
- level: Metadata
  verbs: ["exec"]
  resources:
  - group: ""
    resources: ["pods", "pods/exec"]
# A catch-all rule
- level: Metadata
  omitStages:
  - RequestReceived
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Deploy eSup-Signature to Production

on:
  push:
    branches:
      - main
    paths:
      - 'k8s/**'
      - '.github/workflows/**'
  pull_request:
    branches:
      - main

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Validate Kubernetes manifests
        run: |
          kubectl apply -f k8s/ --dry-run=client --validate=true
      - name: Run Helm lint
        run: |
          helm lint ./helm/esup-signature

  build:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: |
          docker build -t esup-signature:latest .
      - name: Push to registry
        run: |
          echo "${{ secrets.GHCR_PASSWORD }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker tag esup-signature:latest ghcr.io/${{ github.repository }}/esup-signature:latest
          docker push ghcr.io/${{ github.repository }}/esup-signature:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - uses: actions/checkout@v3
      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/01-namespace.yaml
          kubectl apply -f k8s/02-secrets.yaml
          kubectl apply -f k8s/03-configmap.yaml
          kubectl apply -f k8s/esup-signature/10-deployment.yaml
          kubectl rollout status deployment/esup-signature -n signature
      - name: Run smoke tests
        run: |
          kubectl run smoke-test --rm -i --restart=Never \
            --image=curlimages/curl:latest -- \
            curl -f http://esup-signature:8080/actuator/health
```

---

## Monitoring & Logging

### Prometheus Setup

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: signature
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: signature
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.40.0
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus/'
          - '--web.console.libraries=/usr/share/prometheus/console_libraries'
          - '--web.console.templates=/usr/share/prometheus/consoles'
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus
        - name: prometheus-storage
          mountPath: /prometheus
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {}
```

### Grafana Dashboards

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-esup
  namespace: signature
data:
  esup-signature.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "links": [],
      "panels": [
        {
          "datasource": "Prometheus",
          "fieldConfig": {
            "defaults": {},
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 0
          },
          "id": 1,
          "options": {},
          "targets": [
            {
              "expr": "rate(jvm_threads_live_threads{pod=~\"esup-signature.*\"}[5m])",
              "refId": "A"
            }
          ],
          "title": "JVM Thread Count",
          "type": "timeseries"
        }
      ],
      "refresh": "30s",
      "schemaVersion": 35,
      "style": "dark",
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "eSup-Signature Monitoring",
      "uid": "esup-signature",
      "version": 0
    }
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

```bash
# Install Elasticsearch
helm repo add elastic https://Helm.elastic.co
helm install elasticsearch elastic/elasticsearch \
  -n signature \
  --set replicas=3 \
  --set esJavaOpts="-Xmx512m -Xms512m"

# Install Logstash (optional, Fluentd recommended)
helm install logstash elastic/logstash \
  -n signature \
  --values logstash-values.yaml

# Install Kibana
helm install kibana elastic/kibana \
  -n signature \
  --set service.type=LoadBalancer
```

### Alert Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: esup-signature-alerts
  namespace: signature
spec:
  groups:
  - name: esup-signature.rules
    interval: 30s
    rules:
    - alert: EsupSignatureDown
      expr: |
        up{job="esup-signature"} == 0
      for: 5m
      annotations:
        summary: "eSup-Signature is down"
        description: "eSup-Signature pod {{ $labels.pod }} is not responding"
    
    - alert: HighJVMMemory
      expr: |
        jvm_memory_used_bytes{pod=~"esup-signature.*",area="heap"} 
        / jvm_memory_max_bytes{pod=~"esup-signature.*",area="heap"} > 0.8
      for: 5m
      annotations:
        summary: "High JVM memory usage"
        description: "JVM heap memory is above 80% on {{ $labels.pod }}"
    
    - alert: PostgreSQLDown
      expr: |
        up{job="postgresql"} == 0
      for: 5m
      annotations:
        summary: "PostgreSQL database is down"

    - alert: PostgreSQLConnections
      expr: |
        pg_stat_activity_count{pod=~"postgres.*"} > 180
      for: 5m
      annotations:
        summary: "High PostgreSQL connections"
        description: "PostgreSQL connections approaching limit (180/200)"
```

---

## Backup & Disaster Recovery

### PostgreSQL Backup Strategy

#### Continuous WAL Archiving

```yaml
# Enable WAL archiving to S3
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-backup-config
  namespace: signature
data:
  wal-g.env: |
    PGBACKREST_REPO1_TYPE=s3
    PGBACKREST_REPO1_S3_BUCKET=signature-backups
    PGBACKREST_REPO1_S3_REGION=us-east-1
    PGBACKREST_REPO1_PATH=/pgbackrest
    PGBACKREST_BACKUP_TYPE=full
    PGBACKREST_ARCHIVE_MODE=on
    PGBACKREST_ARCHIVE_TIMEOUT=300
```

#### Scheduled Full Backups

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: signature
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM UTC
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: postgres-backup
          containers:
          - name: backup
            image: pgbackrest/pgbackrest:latest
            command:
            - /bin/sh
            - -c
            - |
              pgbackrest backup \
                --stanza=esup-signature \
                --repo1-retention-full=7 \
                --repo1-retention-diff=3
            env:
            - name: PGBACKREST_PG1_PATH
              value: /pgdata
            volumeMounts:
            - name: pgdata
              mountPath: /pgdata
          restartPolicy: OnFailure
          volumes:
          - name: pgdata
            persistentVolumeClaim:
              claimName: postgres-data
```

### Backup Verification

```bash
# List backups
pgbackrest info --stanza=esup-signature

# Test restore (on test environment)
pgbackrest restore --stanza=esup-signature \
  --delta \
  --archive-mode=off

# Verify backup integrity
pgbackrest verify --stanza=esup-signature
```

### Disaster Recovery Plan

#### RTO/RPO Targets

| Scenario | RTO | RPO |
|----------|-----|-----|
| PostgreSQL failover | 5 min | < 1 min |
| Single pod crash | 1 min | N/A |
| Node failure | 5 min | < 1 min |
| Data center loss | 30 min | < 5 min |
| Complete loss | 1-4 hours | < 1 day |

#### Recovery Procedures

```bash
# 1. PostgreSQL Recovery from Backup
kubectl exec -it postgresql-0 -n signature -- \
  pg_basebackup -D /pgdata -Ft -z -X stream

# 2. Point-in-Time Recovery (PITR)
pgbackrest restore --stanza=esup-signature \
  --type=time \
  --target='2025-01-15 14:30:00'

# 3. Application Recovery
kubectl rollout undo deployment/esup-signature -n signature

# 4. Data Verification
kubectl exec -it <pod> -n signature -- \
  curl -s http://localhost:8080/actuator/health
```

---

## Troubleshooting

### Common Issues

#### 1. Pod Startup Failures

```bash
# Check pod status
kubectl describe pod <pod-name> -n signature

# View logs
kubectl logs <pod-name> -n signature
kubectl logs <pod-name> -n signature --tail=100
kubectl logs <pod-name> -n signature --previous  # Previous crash

# Check resource limits
kubectl top pod <pod-name> -n signature
kubectl describe node <node-name>
```

#### 2. Database Connection Issues

```bash
# Test PostgreSQL connectivity
kubectl run -it --rm debug --image=postgres:13 --restart=Never -- \
  psql -h postgresql -U esup_user -d esup_signature

# Check PostgreSQL logs
kubectl logs -n signature postgresql-0

# Check connection pool
kubectl exec -it postgresql-0 -n signature -- \
  psql -c "SELECT count(*) FROM pg_stat_activity;"
```

#### 3. Memory Issues

```bash
# Check JVM heap usage
kubectl exec -it <esup-pod> -n signature -- \
  curl -s http://localhost:8080/actuator/metrics/jvm.memory.used | jq

# Increase heap size in deployment
kubectl set env deployment/esup-signature -n signature \
  JAVA_OPTS="-Xms2g -Xmx4g"

# Check for memory leaks
kubectl port-forward <pod> 9010:9010 -n signature
# Connect JProfiler or JVisualVM to localhost:9010
```

#### 4. Slow Queries

```bash
# Enable query logging in PostgreSQL
kubectl exec -it postgresql-0 -n signature -- \
  psql -c "ALTER SYSTEM SET log_min_duration_statement = 1000;"

# Check slow query logs
kubectl logs -n signature postgresql-0 | grep duration

# Analyze query plans
kubectl exec -it postgresql-0 -n signature -- \
  psql -c "EXPLAIN ANALYZE SELECT ...;"
```

### Debug Commands

```bash
# Get detailed cluster info
kubectl cluster-info
kubectl get nodes -o wide

# Check events
kubectl get events -n signature --sort-by='.lastTimestamp'

# Exec into pod
kubectl exec -it <pod-name> -n signature -- /bin/bash

# Port-forward to service
kubectl port-forward svc/esup-signature 8080:8080 -n signature

# Get all resources in namespace
kubectl get all -n signature

# Describe all pods
kubectl describe pods -n signature
```

---

## Production Checklist

Before deploying to production, ensure:

### Infrastructure ✅
- [ ] Kubernetes cluster ≥1.24 configured
- [ ] Storage classes configured (fast for DB, standard for cache)
- [ ] Load balancer ready
- [ ] DNS configured
- [ ] Backup infrastructure (S3/NFS) ready
- [ ] Monitoring infrastructure deployed
- [ ] Logging infrastructure (ELK) deployed

### Security ✅
- [ ] Network policies implemented
- [ ] RBAC roles configured
- [ ] Pod Security Policies enforced
- [ ] Secrets encrypted (Sealed Secrets or External Secrets)
- [ ] TLS/SSL certificates ready (Let's Encrypt)
- [ ] LDAP/OAuth2 configured
- [ ] Audit logging enabled
- [ ] Firewall rules configured

### Configuration ✅
- [ ] All secrets generated and stored securely
- [ ] All ConfigMaps reviewed and tested
- [ ] Resource limits/requests configured appropriately
- [ ] Autoscaling policies reviewed
- [ ] Backup policies tested
- [ ] Disaster recovery procedures documented

### Testing ✅
- [ ] Kubernetes manifests validated
- [ ] Docker images scanned for vulnerabilities
- [ ] Application tested against staging
- [ ] Load testing completed (100+ concurrent users)
- [ ] Failover tested
- [ ] Backup/restore tested
- [ ] Disaster recovery drill completed

### Documentation ✅
- [ ] Architecture diagram created
- [ ] Runbooks for common tasks written
- [ ] Troubleshooting guide reviewed
- [ ] Oncall documentation prepared
- [ ] Change log maintained

### Monitoring ✅
- [ ] Prometheus metrics verified
- [ ] Grafana dashboards created
- [ ] Alert rules configured and tested
- [ ] Log aggregation working
- [ ] Performance baselines established

---

## References

- [eSup-Signature GitHub](https://github.com/EsupPortail/esup-signature)
- [eSup-Signature Documentation](https://www.esup-portail.org/wiki/display/SIGN)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [PostgreSQL High Availability](https://wiki.postgresql.org/wiki/Replication,_Clustering,_and_Connection_Pooling)
- [Redis Cluster Guide](https://redis.io/topics/cluster-tutorial)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [ELK Stack](https://www.elastic.co/what-is/elk-stack)

---

## Support & Community

- **eSup-Signature Issues**: https://github.com/EsupPortail/esup-signature/issues
- **eSup-Portail Community**: https://www.esup-portail.org/
- **Kubernetes Community**: https://kubernetes.io/community/
- **PostgreSQL Community**: https://www.postgresql.org/community/

---

**Last Updated:** 2026-05-01  
**Maintained By:** DevOps Team  
**Version:** 2.0 (Production Ready)
