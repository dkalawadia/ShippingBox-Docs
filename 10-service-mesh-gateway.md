# Service Mesh and API Gateway Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Service Mesh Architecture

### 1.1 Istio Configuration
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: shippingbox-istio-config
spec:
  profile: default
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2048Mi
          limits:
            cpu: 1000m
            memory: 4096Mi
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          resources:
            requests:
              cpu: 500m
              memory: 1024Mi
            limits:
              cpu: 1000m
              memory: 2048Mi
          service:
            type: LoadBalancer
            ports:
              - port: 80
                targetPort: 8080
                name: http2
              - port: 443
                targetPort: 8443
                name: https
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
      mtls:
        enabled: true
```

### 1.2 Service Mesh Policies
```yaml
# Default mTLS Policy
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: shippingbox-api
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/frontend"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/*"]
    - from:
        - source:
            principals: ["cluster.local/ns/monitoring/sa/prometheus"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/metrics"]
```

## 2. Traffic Management

### 2.1 Gateway Configuration
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: shippingbox-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "api.shippingbox.com"
      tls:
        httpsRedirect: true
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "api.shippingbox.com"
      tls:
        mode: SIMPLE
        credentialName: api-tls-cert

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api-routes
spec:
  hosts:
    - "api.shippingbox.com"
  gateways:
    - shippingbox-gateway
  http:
    - match:
        - uri:
            prefix: "/api/v1/calculator"
      route:
        - destination:
            host: calculator-service
            port:
              number: 3000
      timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "connect-failure,refused-stream,unavailable"

    - match:
        - uri:
            prefix: "/api/v1/shipping"
      route:
        - destination:
            host: shipping-service
            port:
              number: 3000
```

### 2.2 Circuit Breaking
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker
spec:
  host: calculator-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 10

---
# Service-specific timeout and retry policies
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: resilient-routing
spec:
  hosts:
    - calculator-service
  http:
    - route:
        - destination:
            host: calculator-service
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "connect-failure,refused-stream,unavailable,5xx"
```

## 3. Security Configurations

### 3.1 Authentication
```yaml
# JWT Authentication
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: shippingbox-api
  jwtRules:
    - issuer: "https://auth.shippingbox.com"
      jwksUri: "https://auth.shippingbox.com/.well-known/jwks.json"
      audiences:
        - "shippingbox-api"
      forwardOriginalToken: true

---
# Authorization Policies
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-authorization
  namespace: production
spec:
  selector:
    matchLabels:
      app: shippingbox-api
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/*"]
      when:
        - key: request.auth.claims[scope]
          values: ["api:access"]
```

### 3.2 mTLS Configuration
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT
  portLevelMtls:
    9090:
      mode: PERMISSIVE  # Allow Prometheus scraping

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: mtls-destination
spec:
  host: "*.production.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

## 4. Observability

### 4.1 Tracing Configuration
```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: tracing-config
spec:
  tracing:
    - randomSamplingPercentage: 100.0
      providers:
        - name: jaeger
          zipkin:
            url: http://jaeger-collector.monitoring:9411
      customTags:
        environment:
          literal:
            value: production

---
# Jaeger Configuration
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
```

### 4.2 Metrics Configuration
```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: metrics-config
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
            mode: CLIENT_AND_SERVER
        - match:
            metric: REQUEST_DURATION
            mode: CLIENT_AND_SERVER
        - match:
            metric: REQUEST_SIZE
            mode: CLIENT_AND_SERVER

---
# Service Monitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-monitor
spec:
  selector:
    matchLabels:
      istio: mesh
  endpoints:
    - port: http-monitoring
      interval: 15s
```

## 5. API Gateway Integration

### 5.1 Kong Gateway Configuration
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongClusterPlugin
metadata:
  name: cors
spec:
  plugin: cors
  config:
    origins:
      - https://*.shippingbox.com
    methods:
      - GET
      - POST
      - PUT
      - DELETE
      - OPTIONS
    headers:
      - Authorization
      - Content-Type
    credentials: true
    max_age: 3600

---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
spec:
  plugin: rate-limiting
  config:
    minute: 60
    limit_by: consumer
    policy: local
    fault_tolerant: true

---
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: api-ingress
spec:
  proxy:
    protocol: https
    path: /api/v1
    retries: 5
    connect_timeout: 60000
    write_timeout: 60000
    read_timeout: 60000
```