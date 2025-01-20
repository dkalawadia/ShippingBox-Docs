# API Design Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. API Overview

### 1.1 API Standards
- RESTful design principles
- JSON request/response format
- JWT-based authentication
- Versioned endpoints
- Rate limiting
- HTTPS only

### 1.2 Base URLs
```plaintext
Production: https://api.shippingbox.com
Staging:    https://api.staging.shippingbox.com
Development: https://api.dev.shippingbox.com
```

## 2. Authentication and Authorization

### 2.1 Authentication
```yaml
schemes:
  - Bearer Authentication

headers:
  Authorization: Bearer <jwt_token>

endpoints:
  /auth/login:
    post:
      description: Authenticate user and get tokens
      request:
        body:
          email: string
          password: string
      response:
        accessToken: string
        refreshToken: string
        expiresIn: number

  /auth/refresh:
    post:
      description: Refresh access token
      request:
        body:
          refreshToken: string
      response:
        accessToken: string
        expiresIn: number
```

### 2.2 Authorization Rules
```yaml
roles:
  admin:
    - '*'
  user:
    - 'calculator:use'
    - 'shipping:getRates'
    - 'account:view'
    - 'account:edit'
  guest:
    - 'calculator:use'
    - 'shipping:getRates'
```

## 3. API Endpoints

### 3.1 Calculator API
```yaml
/api/v1/calculator:
  /calculate:
    post:
      summary: Calculate optimal box size
      security:
        - Bearer: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - dimensions
                - weight
                - fragility
              properties:
                dimensions:
                  type: object
                  properties:
                    length:
                      type: number
                      minimum: 0.1
                    width:
                      type: number
                      minimum: 0.1
                    height:
                      type: number
                      minimum: 0.1
                weight:
                  type: number
                  minimum: 0.1
                fragility:
                  type: string
                  enum: [low, medium, high]
                quantity:
                  type: integer
                  minimum: 1
                  default: 1
      responses:
        '200':
          description: Successful calculation
          content:
            application/json:
              schema:
                type: object
                properties:
                  calculationId:
                    type: string
                    format: uuid
                  recommendations:
                    type: array
                    items:
                      $ref: '#/components/schemas/BoxRecommendation'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
```

### 3.2 Shipping API
```yaml
/api/v1/shipping:
  /rates:
    post:
      summary: Get shipping rates
      security:
        - Bearer: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - origin
                - destination
                - package
              properties:
                origin:
                  $ref: '#/components/schemas/Address'
                destination:
                  $ref: '#/components/schemas/Address'
                package:
                  $ref: '#/components/schemas/Package'
                preferences:
                  $ref: '#/components/schemas/ShippingPreferences'
      responses:
        '200':
          description: Successful rate retrieval
          content:
            application/json:
              schema:
                type: object
                properties:
                  quotes:
                    type: array
                    items:
                      $ref: '#/components/schemas/ShippingQuote'
```

## 4. Common Schema Definitions

### 4.1 Request/Response Schema
```yaml
components:
  schemas:
    BoxRecommendation:
      type: object
      properties:
        boxId:
          type: string
          format: uuid
        dimensions:
          $ref: '#/components/schemas/Dimensions'
        cost:
          type: number
        packingStrategy:
          $ref: '#/components/schemas/PackingStrategy'

    Address:
      type: object
      required:
        - street1
        - city
        - state
        - postalCode
        - country
      properties:
        companyName:
          type: string
        street1:
          type: string
        street2:
          type: string
        city:
          type: string
        state:
          type: string
        postalCode:
          type: string
        country:
          type: string
        phone:
          type: string
        email:
          type: string
        isResidential:
          type: boolean
          default: true

    ShippingQuote:
      type: object
      properties:
        carrier:
          type: string
        service:
          type: string
        cost:
          type: number
        currency:
          type: string
        transitDays:
          type: integer
        guaranteedDelivery:
          type: boolean
```

## 5. Error Handling

### 5.1 Error Response Format
```yaml
components:
  responses:
    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            type: object
            properties:
              code:
                type: string
              message:
                type: string
              details:
                type: object

    ValidationError:
      description: Validation error
      content:
        application/json:
          schema:
            type: object
            properties:
              errors:
                type: array
                items:
                  type: object
                  properties:
                    field:
                      type: string
                    message:
                      type: string
                    code:
                      type: string
```

## 6. Rate Limiting

### 6.1 Rate Limit Configuration
```yaml
rateLimit:
  anonymous:
    rate: 30
    per: minute
  authenticated:
    rate: 100
    per: minute
  premium:
    rate: 1000
    per: minute

headers:
  X-RateLimit-Limit: number
  X-RateLimit-Remaining: number
  X-RateLimit-Reset: number
```

## 7. Versioning Strategy

### 7.1 Version Control
```yaml
versioning:
  type: URI
  format: /api/v{version}/
  supported:
    - v1
  deprecated:
    - none
  sunset:
    policy: 6 months notice
```

## 8. API Documentation

### 8.1 Swagger Configuration
```yaml
swagger:
  enabled: true
  path: /api/docs
  spec: /api/swagger.json
  ui:
    enabled: true
    path: /api/docs/ui
    config:
      displayRequestDuration: true
      filter: true
      tryItOutEnabled: true
```

## 9. Testing Endpoints

### 9.1 Health Check
```yaml
/health:
  get:
    summary: Check API health status
    security: []
    responses:
      '200':
        description: Health check response
        content:
          application/json:
            schema:
              type: object
              properties:
                status:
                  type: string
                  enum: [healthy, degraded, unhealthy]
                version:
                  type: string
                timestamp:
                  type: string
                  format: date-time
```