# Monitoring and Logging Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Monitoring Infrastructure

### 1.1 Prometheus Configuration
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'api-servers'
    static_configs:
      - targets: ['api:3000']
    metrics_path: '/metrics'
    scheme: 'http'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.*):.*'
        replacement: '$1'

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

### 1.2 Metrics Collection
```typescript
interface MetricsConfig {
  collectors: {
    [key: string]: {
      enabled: boolean;
      interval: number;
      labels: string[];
    };
  };
  aggregations: {
    [key: string]: {
      window: number;
      function: string;
    };
  };
}

const metricsConfig: MetricsConfig = {
  collectors: {
    http: {
      enabled: true,
      interval: 10,
      labels: ['method', 'path', 'status']
    },
    database: {
      enabled: true,
      interval: 30,
      labels: ['query', 'table', 'type']
    },
    cache: {
      enabled: true,
      interval: 15,
      labels: ['operation', 'key']
    },
    business: {
      enabled: true,
      interval: 60,
      labels: ['type', 'status']
    }
  },
  aggregations: {
    requestLatency: {
      window: 300,  // 5 minutes
      function: 'p95'
    },
    errorRate: {
      window: 300,
      function: 'rate'
    }
  }
};
```

## 2. Logging Framework

### 2.1 Logging Configuration
```typescript
interface LoggingConfig {
  general: {
    level: string;
    format: string;
    timestamp: boolean;
    colorize: boolean;
  };
  transports: {
    console: {
      enabled: boolean;
      level: string;
    };
    file: {
      enabled: boolean;
      level: string;
      filename: string;
      maxSize: number;
      maxFiles: number;
    };
    elasticsearch: {
      enabled: boolean;
      level: string;
      node: string;
      index: string;
    };
  };
  context: {
    service: string;
    environment: string;
    version: string;
  };
}

const loggingConfig: LoggingConfig = {
  general: {
    level: process.env.LOG_LEVEL || 'info',
    format: 'json',
    timestamp: true,
    colorize: process.env.NODE_ENV !== 'production'
  },
  transports: {
    console: {
      enabled: true,
      level: 'info'
    },
    file: {
      enabled: true,
      level: 'error',
      filename: 'logs/error.log',
      maxSize: 5242880,  // 5MB
      maxFiles: 5
    },
    elasticsearch: {
      enabled: true,
      level: 'info',
      node: process.env.ELASTICSEARCH_URL,
      index: 'shippingbox-logs'
    }
  },
  context: {
    service: 'shippingbox-api',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION
  }
};
```

### 2.2 Log Schema
```typescript
interface LogSchema {
  timestamp: string;
  level: string;
  message: string;
  context: {
    service: string;
    environment: string;
    version: string;
  };
  metadata: {
    requestId?: string;
    userId?: string;
    ip?: string;
    userAgent?: string;
    path?: string;
    method?: string;
    statusCode?: number;
    duration?: number;
    [key: string]: any;
  };
  error?: {
    name: string;
    message: string;
    stack?: string;
    code?: string;
  };
}
```

## 3. Alerting System

### 3.1 Alert Rules
```yaml
groups:
  - name: ShippingBox
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / 
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: High HTTP error rate
          description: Error rate is {{ $value | humanizePercentage }}

      - alert: APILatencyHigh
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) 
            by (le)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: API latency is high
          description: 95th percentile of request duration is {{ $value }} seconds

      - alert: DatabaseConnectionPoolNearLimit
        expr: |
          pg_stat_activity_count 
          / 
          pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Database connection pool near limit
          description: Connection pool is at {{ $value | humanizePercentage }}
```

### 3.2 Alert Manager Configuration
```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      repeat_interval: 1h
    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@shippingbox.com'
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<pagerduty-service-key>'
        send_resolved: true

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
          {{ end }}
```

## 4. Dashboard Configuration

### 4.1 Grafana Dashboard
```json
{
  "dashboard": {
    "id": null,
    "title": "ShippingBox API Metrics",
    "tags": ["api", "production"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (status)",
            "legendFormat": "{{status}}"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "title": "Response Time",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "p50"
          }
        ]
      }
    ]
  }
}
```

## 5. Tracing Configuration

### 5.1 OpenTelemetry Configuration
```typescript
interface TracingConfig {
  service: {
    name: string;
    version: string;
  };
  exporter: {
    type: string;
    endpoint: string;
    headers: Record<string, string>;
  };
  sampling: {
    probability: number;
    rules: {
      [key: string]: number;
    };
  };
  propagation: string[];
}

const tracingConfig: TracingConfig = {
  service: {
    name: 'shippingbox-api',
    version: process.env.APP_VERSION
  },
  exporter: {
    type: 'jaeger',
    endpoint: process.env.JAEGER_ENDPOINT,
    headers: {
      'X-API-Key': process.env.JAEGER_API_KEY
    }
  },
  sampling: {
    probability: 0.1,
    rules: {
      'api.v1.calculator': 1.0,
      'api.v1.shipping': 0.5
    }
  },
  propagation: ['tracecontext', 'baggage', 'b3']
};
```