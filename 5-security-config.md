# Security Configuration Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Authentication System

### 1.1 JWT Configuration
```typescript
interface JWTConfig {
  accessToken: {
    secret: string;
    expiresIn: string;
    algorithm: string;
    issuer: string;
    audience: string[];
  };
  refreshToken: {
    secret: string;
    expiresIn: string;
    algorithm: string;
  };
  encryption: {
    algorithm: string;
    keySize: number;
    ivSize: number;
  };
}

const jwtConfig: JWTConfig = {
  accessToken: {
    secret: process.env.JWT_ACCESS_SECRET,
    expiresIn: '15m',
    algorithm: 'HS256',
    issuer: 'shippingbox.com',
    audience: ['api.shippingbox.com']
  },
  refreshToken: {
    secret: process.env.JWT_REFRESH_SECRET,
    expiresIn: '7d',
    algorithm: 'HS256'
  },
  encryption: {
    algorithm: 'aes-256-gcm',
    keySize: 32,
    ivSize: 16
  }
};
```

### 1.2 Password Security
```typescript
interface PasswordConfig {
  hashingConfig: {
    algorithm: string;
    saltRounds: number;
    pepperSecret: string;
    memoryLimit: number;
    iterations: number;
  };
  requirements: {
    minLength: number;
    requireUppercase: boolean;
    requireLowercase: boolean;
    requireNumbers: boolean;
    requireSpecial: boolean;
    maxAge: number;
    preventReuse: number;
  };
}

const passwordConfig: PasswordConfig = {
  hashingConfig: {
    algorithm: 'argon2id',
    saltRounds: 12,
    pepperSecret: process.env.PASSWORD_PEPPER,
    memoryLimit: 65536,
    iterations: 3
  },
  requirements: {
    minLength: 12,
    requireUppercase: true,
    requireLowercase: true,
    requireNumbers: true,
    requireSpecial: true,
    maxAge: 90, // days
    preventReuse: 5  // last 5 passwords
  }
};
```

## 2. Authorization Framework

### 2.1 Role-Based Access Control (RBAC)
```typescript
interface RBACConfig {
  roles: {
    [key: string]: {
      permissions: string[];
      inherits?: string[];
    };
  };
  resources: {
    [key: string]: {
      actions: string[];
      attributes: string[];
    };
  };
}

const rbacConfig: RBACConfig = {
  roles: {
    admin: {
      permissions: ['*']
    },
    manager: {
      permissions: [
        'users:read',
        'users:write',
        'reports:*',
        'settings:read',
        'settings:write'
      ],
      inherits: ['user']
    },
    user: {
      permissions: [
        'calculator:use',
        'shipping:getRates',
        'profile:*'
      ]
    }
  },
  resources: {
    users: {
      actions: ['create', 'read', 'update', 'delete'],
      attributes: ['id', 'email', 'role', 'status']
    },
    shipping: {
      actions: ['getRates', 'book', 'cancel'],
      attributes: ['*']
    }
  }
};
```

## 3. Data Encryption

### 3.1 Encryption Configuration
```typescript
interface EncryptionConfig {
  atRest: {
    algorithm: string;
    keySize: number;
    keyRotationInterval: number;
  };
  inTransit: {
    minTLSVersion: string;
    preferredCipherSuites: string[];
  };
  keyManagement: {
    provider: string;
    region: string;
    keyRotationEnabled: boolean;
  };
}

const encryptionConfig: EncryptionConfig = {
  atRest: {
    algorithm: 'AES-256-GCM',
    keySize: 256,
    keyRotationInterval: 90 // days
  },
  inTransit: {
    minTLSVersion: 'TLSv1.3',
    preferredCipherSuites: [
      'TLS_AES_128_GCM_SHA256',
      'TLS_AES_256_GCM_SHA384',
      'TLS_CHACHA20_POLY1305_SHA256'
    ]
  },
  keyManagement: {
    provider: 'AWS_KMS',
    region: 'us-east-1',
    keyRotationEnabled: true
  }
};
```

## 4. Session Management

### 4.1 Session Configuration
```typescript
interface SessionConfig {
  cookie: {
    secure: boolean;
    httpOnly: boolean;
    sameSite: string;
    maxAge: number;
    domain: string;
  };
  storage: {
    type: string;
    prefix: string;
    ttl: number;
  };
  security: {
    regenerateInterval: number;
    maxConcurrentSessions: number;
    invalidateOnPasswordChange: boolean;
  };
}

const sessionConfig: SessionConfig = {
  cookie: {
    secure: true,
    httpOnly: true,
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    domain: '.shippingbox.com'
  },
  storage: {
    type: 'redis',
    prefix: 'sess:',
    ttl: 86400 // 24 hours
  },
  security: {
    regenerateInterval: 3600, // 1 hour
    maxConcurrentSessions: 5,
    invalidateOnPasswordChange: true
  }
};
```

## 5. API Security

### 5.1 Rate Limiting
```typescript
interface RateLimitConfig {
  global: {
    windowMs: number;
    max: number;
  };
  perUser: {
    windowMs: number;
    max: number;
  };
  perIP: {
    windowMs: number;
    max: number;
  };
  headers: boolean;
  skipSuccessfulRequests: boolean;
}

const rateLimitConfig: RateLimitConfig = {
  global: {
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 1000
  },
  perUser: {
    windowMs: 15 * 60 * 1000,
    max: 100
  },
  perIP: {
    windowMs: 15 * 60 * 1000,
    max: 50
  },
  headers: true,
  skipSuccessfulRequests: false
};
```

### 5.2 CORS Configuration
```typescript
interface CORSConfig {
  origin: string[];
  methods: string[];
  allowedHeaders: string[];
  exposedHeaders: string[];
  credentials: boolean;
  maxAge: number;
}

const corsConfig: CORSConfig = {
  origin: [
    'https://shippingbox.com',
    'https://admin.shippingbox.com',
    /\.shippingbox\.com$/
  ],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'X-Requested-With'
  ],
  exposedHeaders: [
    'Content-Range',
    'X-RateLimit-Limit'
  ],
  credentials: true,
  maxAge: 86400 // 24 hours
};
```

## 6. Compliance and Audit

### 6.1 Audit Logging
```typescript
interface AuditConfig {
  events: {
    [key: string]: {
      level: string;
      retention: number;
    };
  };
  storage: {
    type: string;
    path: string;
    rotation: number;
  };
  alerting: {
    enabled: boolean;
    threshold: number;
    channels: string[];
  };
}

const auditConfig: AuditConfig = {
  events: {
    authentication: {
      level: 'info',
      retention: 90 // days
    },
    authorization: {
      level: 'warn',
      retention: 90
    },
    dataAccess: {
      level: 'info',
      retention: 30
    }
  },
  storage: {
    type: 'elasticsearch',
    path: 'audit-logs',
    rotation: 30 // days
  },
  alerting: {
    enabled: true,
    threshold: 5,
    channels: ['email', 'slack']
  }
};
```

## 7. Security Headers

### 7.1 HTTP Security Headers
```typescript
interface SecurityHeadersConfig {
  helmet: {
    contentSecurityPolicy: {
      directives: {
        [key: string]: string[];
      };
    };
    hsts: {
      maxAge: number;
      includeSubDomains: boolean;
      preload: boolean;
    };
    referrerPolicy: string;
    permittedCrossDomainPolicies: string;
  };
}

const securityHeadersConfig: SecurityHeadersConfig = {
  helmet: {
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https:"],
        connectSrc: ["'self'"],
        fontSrc: ["'self'"],
        objectSrc: ["'none'"],
        mediaSrc: ["'none'"],
        frameSrc: ["'none'"]
      }
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    },
    referrerPolicy: "strict-origin-when-cross-origin",
    permittedCrossDomainPolicies: "none"
  }
};
```