---
name: microservices-patterns
description: Microservices architecture patterns for Node.js including service communication, API gateway, shared auth, logging, deployment, and migration from monolith. Covers Amgrant v2 multi-service stack.
origin: DRIS
---

# Microservices Patterns (Node.js)

## When to Activate

- Working on Amgrant v2 microservice architecture
- Designing inter-service communication
- Setting up shared authentication across services
- Configuring API gateway or service mesh
- Debugging cross-service issues
- Planning microservice to monolith migration (v2 → v3)

## Amgrant v2 Service Architecture

```
┌────────────────────────────────────────────────────────┐
│                    API Gateway (Nginx)                  │
│              (Rate limiting, SSL, routing)              │
└────────────────────┬───────────────────────────────────┘
                     │
        ┌────────────┼────────────┬──────────────┐
        │            │            │              │
   ┌────▼────┐ ┌────▼────┐ ┌────▼────┐  ┌──────▼──────┐
   │ Auth    │ │ Project │ │Benefi-  │  │ Reports     │
   │ Service │ │ Service │ │ciary    │  │ Service     │
   │ :3001   │ │ :3002   │ │Service  │  │ :3004       │
   │         │ │         │ │ :3003   │  │             │
   └────┬────┘ └────┬────┘ └────┬────┘  └──────┬──────┘
        │            │            │              │
        └────────────┴────────────┴──────────────┘
                          │
                    ┌─────▼─────┐
                    │  MongoDB  │
                    │ (shared)  │
                    └───────────┘
```

## Service Communication

### HTTP (Synchronous)

```typescript
// services/http-client.ts
// Shared HTTP client for inter-service calls
import axios, { AxiosInstance, AxiosError } from 'axios';

export function createServiceClient(serviceName: string, baseURL: string): AxiosInstance {
  const client = axios.create({
    baseURL,
    timeout: 5000,
    headers: { 'X-Service-Name': serviceName },
  });

  // Retry with exponential backoff
  client.interceptors.response.use(undefined, async (error: AxiosError) => {
    const config = error.config as any;
    if (!config || config._retryCount >= 3) throw error;

    config._retryCount = (config._retryCount || 0) + 1;
    const delay = Math.pow(2, config._retryCount) * 100;
    await new Promise(resolve => setTimeout(resolve, delay));

    return client(config);
  });

  return client;
}

// Usage
const projectService = createServiceClient('beneficiary-service', 'http://project-service:3002');
const authService = createServiceClient('beneficiary-service', 'http://auth-service:3001');

// Call another service
async function getProjectDetails(projectId: string, token: string) {
  const { data } = await projectService.get(`/api/projects/${projectId}`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return data;
}
```

### Event-Driven (Asynchronous)

```typescript
// events/event-bus.ts
// Use Redis pub/sub or a message queue for async communication
import Redis from 'ioredis';

const publisher = new Redis(process.env.REDIS_URL!);
const subscriber = new Redis(process.env.REDIS_URL!);

export async function publishEvent(channel: string, event: any) {
  await publisher.publish(channel, JSON.stringify({
    ...event,
    timestamp: new Date().toISOString(),
    service: process.env.SERVICE_NAME,
  }));
}

export function subscribeToEvent(channel: string, handler: (event: any) => void) {
  subscriber.subscribe(channel);
  subscriber.on('message', (ch, message) => {
    if (ch === channel) {
      try {
        handler(JSON.parse(message));
      } catch (err) {
        console.error(`Error processing event on ${channel}:`, err);
      }
    }
  });
}

// In beneficiary service:
// When a beneficiary is created, notify other services
publishEvent('beneficiary.created', {
  type: 'BENEFICIARY_CREATED',
  data: { id: beneficiary._id, project: beneficiary.project }
});

// In reports service:
// Listen for beneficiary events to update aggregates
subscribeToEvent('beneficiary.created', async (event) => {
  await updateProjectStats(event.data.project);
});
```

## Shared Authentication

```typescript
// shared/auth-middleware.ts
// This module is shared across all services via npm package or git submodule
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

export interface AuthUser {
  id: string;
  email: string;
  role: string;
  district?: string;
  permissions: string[];
}

export function authenticate(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as AuthUser;
    (req as any).user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}

export function authorize(roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = (req as any).user as AuthUser;
    if (!user || !roles.includes(user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
```

## API Gateway Configuration

```nginx
# nginx/api-gateway.conf
upstream auth_service {
    server auth-service:3001;
}
upstream project_service {
    server project-service:3002;
}
upstream beneficiary_service {
    server beneficiary-service:3003;
}
upstream reports_service {
    server reports-service:3004;
}

server {
    listen 80;
    server_name api.amgrant.dris.com;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req zone=api burst=50 nodelay;

    # Auth service
    location /api/auth/ {
        proxy_pass http://auth_service;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Project service
    location /api/projects/ {
        proxy_pass http://project_service;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Beneficiary service
    location /api/beneficiaries/ {
        proxy_pass http://beneficiary_service;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Reports service
    location /api/reports/ {
        proxy_pass http://reports_service;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Health check aggregation
    location /health {
        access_log off;
        default_type application/json;
        return 200 '{"status":"ok"}';
    }
}
```

## Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    ports: ['27017:27017']
    volumes: ['mongo-data:/data/db']
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}

  redis:
    image: redis:7-alpine
    ports: ['6379:6379']

  auth-service:
    build: ./services/auth
    ports: ['3001:3001']
    env_file: .env
    depends_on: [mongodb, redis]

  project-service:
    build: ./services/project
    ports: ['3002:3002']
    env_file: .env
    depends_on: [mongodb, redis]

  beneficiary-service:
    build: ./services/beneficiary
    ports: ['3003:3003']
    env_file: .env
    depends_on: [mongodb, redis]

  reports-service:
    build: ./services/reports
    ports: ['3004:3004']
    env_file: .env
    depends_on: [mongodb, redis]

  nginx:
    image: nginx:alpine
    ports: ['80:80']
    volumes: ['./nginx:/etc/nginx/conf.d']
    depends_on:
      - auth-service
      - project-service
      - beneficiary-service
      - reports-service

volumes:
  mongo-data:
```

## Logging & Observability

```typescript
// shared/logger.ts
import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME,
    version: process.env.APP_VERSION,
  },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

// Structured logging for cross-service tracing
export function logRequest(req: any, res: any, duration: number) {
  logger.info('request', {
    method: req.method,
    path: req.path,
    status: res.statusCode,
    duration,
    user: req.user?.email,
    traceId: req.headers['x-trace-id'],  // Propagate across services
  });
}
```

## Migration Strategy: v2 → v3

When migrating from Amgrant v2 (microservices) to v3 (Frappe):

1. **Strangler Fig Pattern**: Gradually route traffic from v2 services to v3 Frappe endpoints
2. **Data sync**: Run bidirectional sync during transition period
3. **Feature parity**: Map each microservice's features to Frappe DocTypes/modules
4. **API compatibility**: Keep v2 API endpoints alive as proxies to v3

```
Phase 1: Run v2 and v3 side by side
Phase 2: New features built in v3 only
Phase 3: Migrate data from MongoDB to Frappe/MySQL
Phase 4: Route all traffic to v3
Phase 5: Decommission v2 services
```
