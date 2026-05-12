# 🏗️ Structure des Services

## Auth Service - NestJS

### package.json
```json
{
  "name": "@pennylane/auth-service",
  "version": "1.0.0",
  "description": "Authentication & Authorization Service",
  "main": "dist/main.js",
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/main.js",
    "test": "jest",
    "test:cov": "jest --coverage"
  },
  "dependencies": {
    "@nestjs/common": "^10.2.0",
    "@nestjs/core": "^10.2.0",
    "@nestjs/jwt": "^11.0.0",
    "@nestjs/passport": "^10.0.0",
    "@prisma/client": "^5.4.0",
    "bcrypt": "^5.1.0",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.1",
    "redis": "^4.6.0",
    "kafkajs": "^2.2.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.2.0",
    "@types/node": "^20.0.0",
    "typescript": "^5.3.0"
  }
}
```

### Dockerfile
```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

EXPOSE 3001

HEALTHCHECK --interval=10s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3001/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

CMD ["node", "dist/main.js"]
```

## Accounting Service - NestJS

### Structure
```
services/accounting-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── modules/
│   │   ├── journal/
│   │   │   ├── journal.controller.ts
│   │   │   ├── journal.service.ts
│   │   │   └── dto/
│   │   ├── accounts/
│   │   ├── reconciliation/
│   │   └── reporting/
│   ├── database/
│   │   ├── prisma.service.ts
│   │   └── migrations/
│   └── common/
│       ├── filters/
│       ├── guards/
│       └── decorators/
├── test/
├── prisma/
│   └── schema.prisma
└── Dockerfile
```
