# Deployment Configuration Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. CI/CD Pipeline

### 1.1 GitHub Actions Workflow
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Type check
        run: npm run typecheck
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  security-scan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run SAST
        uses: github/codeql-action/analyze@v2
        with:
          languages: javascript
      
      - name: Run dependency scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Run container scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  build:
    needs: security-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name shippingbox-staging
      
      - name: Deploy to staging
        run: |
          kubectl set image deployment/shippingbox-api \
            api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=staging
          kubectl rollout status deployment/shippingbox-api \
            --namespace=staging --timeout=300s

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.shippingbox.com
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name shippingbox-production
      
      - name: Deploy to production
        run: |
          kubectl set image deployment/shippingbox-api \
            api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=production
          kubectl rollout status deployment/shippingbox-api \
            --namespace=production --timeout=300s
```

## 2. Deployment Strategies

### 2.1 Rolling Update Configuration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shippingbox-api
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: shippingbox-api
  template:
    metadata:
      labels:
        app: shippingbox-api
    spec:
      containers:
        - name: api
          image: ghcr.io/shippingbox/api:latest
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
```

### 2.2 Canary Deployment Configuration
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: shippingbox-api-vs
spec:
  hosts:
    - api.shippingbox.com
  http:
    - route:
      - destination:
          host: shippingbox-api
          subset: stable
        weight: 90
      - destination:
          host: shippingbox-api
          subset: canary
        weight: 10

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: shippingbox-api-dr
spec:
  host: shippingbox-api
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

## 3. Environment Configurations

### 3.1 Environment Variables
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  API_VERSION: "v1"
  RATE_LIMIT_WINDOW: "900000"
  RATE_LIMIT_MAX: "1000"
  CACHE_TTL: "3600"

---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: production
type: Opaque
data:
  JWT_SECRET: ${JWT_SECRET_BASE64}
  DATABASE_URL: ${DATABASE_URL_BASE64}
  REDIS_URL: ${REDIS_URL_BASE64}
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID_BASE64}
  AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY_BASE64}
```

### 3.2 Resource Management
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: compute-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      max:
        cpu: "2"
        memory: "2Gi"
      min:
        cpu: "100m"
        memory: "128Mi"
```

## 4. Release Management

### 4.1 Version Control
```typescript
interface VersionConfig {
  release: {
    major: number;
    minor: number;
    patch: number;
  };
  apiVersion: string;
  compatibilityMatrix: {
    [key: string]: string[];
  };
}

const versionConfig: VersionConfig = {
  release: {
    major: 1,
    minor: 0,
    patch: 0
  },
  apiVersion: 'v1',
  compatibilityMatrix: {
    'v1': ['1.0.x'],
    'v2': ['2.0.x']
  }
};
```

### 4.2 Database Migrations
```typescript
// Migration Configuration
interface MigrationConfig {
  directory: string;
  tableName: string;
  schemaName: string;
  transactionMode: 'all' | 'none' | 'each';
  lockTimeout: number;
  validateChecksums: boolean;
}

const migrationConfig: MigrationConfig = {
  directory: './migrations',
  tableName: 'migrations',
  schemaName: 'public',
  transactionMode: 'all',
  lockTimeout: 60000,
  validateChecksums: true
};

// Migration Job
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migration
          image: ghcr.io/shippingbox/migrations:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: DATABASE_URL
      restartPolicy: OnFailure
```

## 5. Monitoring and Alerts

### 5.1 Deployment Metrics
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: deployment-monitor
spec:
  selector:
    matchLabels:
      app: shippingbox-api
  endpoints:
    - port: metrics
      interval: 15s
  namespaceSelector:
    matchNames:
      - production

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: deployment-alerts
spec:
  groups:
    - name: deployment
      rules:
        - alert: DeploymentRolloutStuck
          expr: |
            kube_deployment_status_observed_generation{namespace="production"}
            != kube_deployment_metadata_generation{namespace="production"}
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: Deployment rollout stuck
            description: Deployment {{ $labels.deployment }} rollout is stuck
```