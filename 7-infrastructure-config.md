# Infrastructure Configuration Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Cloud Infrastructure (AWS)

### 1.1 VPC Configuration
```terraform
# VPC Configuration
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "shippingbox-vpc"
    Environment = var.environment
    Terraform   = "true"
  }
}

# Subnet Configuration
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "public-${var.availability_zones[count.index]}"
    Environment = var.environment
    Type        = "Public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "private-${var.availability_zones[count.index]}"
    Environment = var.environment
    Type        = "Private"
  }
}

# Security Group Configuration
resource "aws_security_group" "api" {
  name        = "api-security-group"
  description = "Security group for API servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 1.2 EKS Configuration
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: shippingbox-cluster
  region: us-east-1
  version: "1.24"

managedNodeGroups:
  - name: app-nodes
    instanceType: t3.medium
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 50
    tags:
      Environment: production
    labels:
      role: application
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true

  - name: monitoring-nodes
    instanceType: t3.large
    minSize: 1
    maxSize: 3
    desiredCapacity: 2
    volumeSize: 100
    tags:
      Environment: production
    labels:
      role: monitoring

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
```

## 2. Container Orchestration

### 2.1 Kubernetes Base Configuration
```yaml
# Namespace Configuration
apiVersion: v1
kind: Namespace
metadata:
  name: shippingbox
  labels:
    name: shippingbox
    environment: production

---
# Resource Quotas
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: shippingbox
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi

---
# Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: shippingbox
spec:
  podSelector:
    matchLabels:
      app: shippingbox-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
```

### 2.2 Application Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shippingbox-api
  namespace: shippingbox
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shippingbox-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: shippingbox-api
    spec:
      containers:
        - name: api
          image: shippingbox/api:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          env:
            - name: NODE_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: url
```

## 3. Database Infrastructure

### 3.1 RDS Configuration
```terraform
resource "aws_db_instance" "main" {
  identifier        = "shippingbox-db"
  engine           = "postgres"
  engine_version   = "14.5"
  instance_class   = "db.r6g.xlarge"
  allocated_storage = 100
  storage_type      = "gp3"

  multi_az          = true
  publicly_accessible = false

  db_name  = "shippingbox"
  username = var.db_username
  password = var.db_password

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  vpc_security_group_ids = [aws_security_group.database.id]
  db_subnet_group_name   = aws_db_subnet_group.database.name

  performance_insights_enabled = true
  monitoring_interval         = 60
  monitoring_role_arn        = aws_iam_role.rds_monitoring.arn

  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}
```

### 3.2 Redis Configuration
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: shippingbox
spec:
  serviceName: redis-cluster
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:6.2-alpine
          ports:
            - containerPort: 6379
            - containerPort: 16379
          resources:
            requests:
              cpu: "200m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

## 4. Network Configuration

### 4.1 Load Balancer Configuration
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: shippingbox
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  tls:
    - hosts:
        - api.shippingbox.com
      secretName: api-tls
  rules:
    - host: api.shippingbox.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

### 4.2 DNS Configuration
```terraform
resource "aws_route53_zone" "main" {
  name = "shippingbox.com"

  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}

resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.shippingbox.com"
  type    = "A"

  alias {
    name                   = aws_lb.api.dns_name
    zone_id                = aws_lb.api.zone_id
    evaluate_target_health = true
  }
}
```

## 5. Monitoring Infrastructure

### 5.1 CloudWatch Configuration
```terraform
resource "aws_cloudwatch_log_group" "api" {
  name              = "/aws/eks/shippingbox/api"
  retention_in_days = 30

  tags = {
    Environment = var.environment
    Application = "api"
  }
}

resource "aws_cloudwatch_metric_alarm" "api_errors" {
  alarm_name          = "api-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name        = "5XXError"
  namespace          = "AWS/ApplicationELB"
  period             = "300"
  statistic          = "Sum"
  threshold          = "10"
  alarm_description  = "This metric monitors API error rate"
  alarm_actions      = [aws_sns_topic.alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.api.arn_suffix
  }
}
```