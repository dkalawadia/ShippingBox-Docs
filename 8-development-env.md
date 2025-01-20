# Development Environment Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Local Development Setup

### 1.1 Docker Compose Configuration
```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      target: development
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://user:password@db:5432/shippingbox_dev
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev_secret
      - LOG_LEVEL=debug
    depends_on:
      - db
      - redis
    command: npm run dev
    networks:
      - shippingbox-dev

  db:
    image: postgres:14-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=shippingbox_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - shippingbox-dev

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - shippingbox-dev

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"  # SMTP port
      - "8025:8025"  # Web UI port
    networks:
      - shippingbox-dev

  pgadmin:
    image: dpage/pgadmin4
    ports:
      - "8080:80"
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@shippingbox.com
      - PGADMIN_DEFAULT_PASSWORD=admin
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - shippingbox-dev

volumes:
  node_modules:
  postgres_data:
  redis_data:
  pgadmin_data:

networks:
  shippingbox-dev:
    driver: bridge
```

### 1.2 Development Dockerfile
```dockerfile
# Dockerfile.dev
FROM node:18-alpine AS base

# Install development dependencies
RUN apk add --no-cache \
    git \
    python3 \
    make \
    g++ \
    curl

WORKDIR /app

# Install development tools
RUN npm install -g \
    typescript \
    ts-node \
    nodemon \
    @types/node

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Expose ports
EXPOSE 3000 9229

# Start development server with debugging enabled
CMD ["npm", "run", "dev:debug"]
```

## 2. Development Tools Configuration

### 2.1 TypeScript Configuration
```json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "commonjs",
    "lib": ["ES2021"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "moduleResolution": "node",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

### 2.2 ESLint Configuration
```javascript
module.exports = {
  root: true,
  parser: '@typescript-eslint/parser',
  plugins: [
    '@typescript-eslint',
    'jest',
    'prettier',
    'import'
  ],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:jest/recommended',
    'plugin:prettier/recommended'
  ],
  rules: {
    '@typescript-eslint/explicit-function-return-type': ['error'],
    '@typescript-eslint/no-explicit-any': ['error'],
    '@typescript-eslint/no-unused-vars': ['error'],
    'import/order': [
      'error',
      {
        'groups': [
          'builtin',
          'external',
          'internal',
          'parent',
          'sibling',
          'index'
        ],
        'newlines-between': 'always',
        'alphabetize': {
          'order': 'asc',
          'caseInsensitive': true
        }
      }
    ],
    'max-len': ['error', { 
      'code': 100,
      'ignoreUrls': true,
      'ignoreStrings': true,
      'ignoreTemplateLiterals': true
    }]
  },
  settings: {
    'import/resolver': {
      'typescript': {}
    }
  }
}
```

### 2.3 Jest Configuration
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/*.spec.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1'
  },
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.spec.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  coverageReporters: ['text', 'lcov', 'clover'],
  verbose: true
}
```

## 3. Development Scripts

### 3.1 Package.json Scripts
```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "dev:debug": "ts-node-dev --inspect --respawn --transpile-only src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint . --ext .ts",
    "lint:fix": "eslint . --ext .ts --fix",
    "format": "prettier --write \"src/**/*.ts\"",
    "typecheck": "tsc --noEmit",
    "prepare": "husky install",
    "docker:dev": "docker-compose up",
    "docker:dev:build": "docker-compose up --build",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate",
    "db:seed": "ts-node prisma/seed.ts",
    "openapi:generate": "openapi-generator-cli generate -i openapi.yaml -g typescript-axios -o src/api",
    "validate:openapi": "openapi-generator-cli validate -i openapi.yaml"
  }
}
```

## 4. Git Configuration

### 4.1 Git Hooks (Husky)
```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.ts": [
      "eslint --fix",
      "prettier --write",
      "jest --bail --findRelatedTests"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

### 4.2 Commit Message Format
```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'docs',
        'style',
        'refactor',
        'test',
        'chore',
        'perf'
      ]
    ],
    'scope-case': [2, 'always', 'lowercase'],
    'subject-case': [2, 'never', ['upper-case']],
    'subject-full-stop': [2, 'never', '.'],
    'header-max-length': [2, 'always', 72]
  }
};
```

## 5. IDE Configuration

### 5.1 VS Code Settings
```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "jest.autoRun": {
    "watch": true,
    "onSave": "test-file"
  },
  "jest.showCoverageOnLoad": true,
  "files.exclude": {
    "**/.git": true,
    "**/.DS_Store": true,
    "node_modules": true,
    "dist": true,
    "coverage": true
  }
}
```

### 5.2 VS Code Extensions
```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-tslint-plugin",
    "orta.vscode-jest",
    "mikestead.dotenv",
    "prisma.prisma",
    "github.copilot",
    "eamodio.gitlens",
    "ms-azuretools.vscode-docker"
  ]
}
```

## 6. Debugging Configuration

### 6.1 VS Code Launch Configuration
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Debug: Docker",
      "remoteRoot": "/app",
      "localRoot": "${workspaceFolder}",
      "protocol": "inspector",
      "port": 9229,
      "restart": true,
      "skipFiles": [
        "<node_internals>/**"
      ]
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug: Jest Tests",
      "program": "${workspaceFolder}/node_modules/.bin/jest",
      "args": [
        "--runInBand",
        "--watchAll=false"
      ],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    }
  ]
}
```