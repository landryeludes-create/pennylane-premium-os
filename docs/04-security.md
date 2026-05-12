# 🔐 Sécurité Enterprise - Pennylane Premium OS

## Principes Zero Trust

```
┌─────────────────────────────────────────────────────┐
│         ZERO TRUST SECURITY MODEL                   │
├─────────────────────────────────────────────────────┤
│  Never Trust, Always Verify                         │
│  Every request authenticated & authorized           │
│  Encryption everywhere (in-transit + at-rest)      │
│  Monitor & log all activities                       │
└─────────────────────────────────────────────────────┘
```

## Authentification

### JWT Tokens

```typescript
// Token Structure
interface JWTPayload {
  sub: string;        // User ID
  org: string;        // Organization ID
  roles: string[];    // User roles
  iat: number;        // Issued at
  exp: number;        // Expiration (1h)
  aud: 'pennylane';   // Audience
  iss: 'pennylane';   // Issuer
}
```

### Password Security

- Minimum 12 characters
- Bcrypt with salt rounds: 12
- PBKDF2 for additional hashing
- Password reset tokens expire in 1 hour
- No password history repeat

### MFA (Multi-Factor Authentication)

```
OPTIONS:
├── TOTP (Time-based One-Time Password)
│   └── Google Authenticator, Authy
├── SMS OTP
│   └── Twilio integration
└── Backup Codes (10 codes, single use)
```

## Authorization (RBAC)

### Roles

```typescript
ROLE_HIERARCHY = {
  'super_admin': {
    permissions: ['*'],
    scope: 'GLOBAL'
  },
  'admin': {
    permissions: ['users:*', 'settings:*', 'accounting:*'],
    scope: 'ORGANIZATION'
  },
  'accountant': {
    permissions: ['accounting:read', 'accounting:write', 'invoices:*'],
    scope: 'ORGANIZATION'
  },
  'member': {
    permissions: ['accounting:read', 'invoices:read', 'dashboard:read'],
    scope: 'ORGANIZATION'
  },
  'viewer': {
    permissions: ['accounting:read', 'dashboard:read'],
    scope: 'ORGANIZATION'
  }
}
```

## Chiffrement

### En transit (Transit Encryption)

```
✅ TLS 1.3 obligatoire
✅ HSTS (HTTP Strict Transport Security) headers
✅ Certificate pinning pour mobile
✅ Perfect Forward Secrecy (PFS)
```

### Au repos (At-Rest Encryption)

```sql
-- Colonnes sensibles chiffrées
ALTER TABLE users ADD COLUMN encrypted_ssn TEXT;
CREATE TRIGGER encrypt_ssn_trigger
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION encrypt_column();
```

**Données sensibles à chiffrer:**
- Numbers sécurité sociale
- Numéros de compte bancaire
- Tokens API
- Clés privées

## API Security

### Rate Limiting

```typescript
// Par utilisateur
FREE_TIER:          100 req/minute
PROFESSIONAL_TIER: 1000 req/minute
ENTERPRISE_TIER:   Custom

// Global (DDoS protection)
MAX_REQUESTS_PER_SECOND: 10,000
```

### CORS

```typescript
const corsOptions = {
  origin: process.env.CORS_ORIGINS?.split(','),
  credentials: true,
  optionsSuccessStatus: 200,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
};
```

### CSRF Protection

```typescript
// Double Submit Cookie Pattern
GET  /csrf-token → Returns token
POST /submit-form → Validates token
```

## Input Validation

### Sanitization

```typescript
// Utiliser helmet.js
import helmet from 'helmet';
app.use(helmet());

// XSS Protection
app.use(helmet.xssFilter());

// Content Security Policy
app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  })
);
```

### Data Validation

```typescript
// Utiliser Zod ou Joi
import { z } from 'zod';

const createEntrySchema = z.object({
  date: z.string().date(),
  description: z.string().min(1).max(255),
  lines: z.array(z.object({
    accountId: z.string().uuid(),
    debit: z.number().min(0),
    credit: z.number().min(0),
  }))
});
```

## Secrets Management

### Environment Variables

```bash
# NE JAMAIS commiter .env
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore

# Utiliser AWS Secrets Manager ou HashiCorp Vault
```

### Vault Integration

```typescript
// Fetch secrets at runtime
const secret = await vault.read('secret/pennylane/prod/database');
const dbUrl = secret.data.data.url;
```

## Database Security

### Least Privilege

```sql
-- Service accounts avec permissions minimales
CREATE ROLE accounting_service WITH LOGIN;
GRANT USAGE ON SCHEMA accounting TO accounting_service;
GRANT SELECT, INSERT, UPDATE ON accounting.journal_entries TO accounting_service;
GRANT EXECUTE ON FUNCTION audit_trigger() TO accounting_service;
```

### SQL Injection Prevention

```typescript
// Utiliser ORM (Prisma)
const entries = await prisma.journalEntry.findMany({
  where: {
    description: {
      contains: userInput  // Automatiquement échappé
    }
  }
});

// Parameterized queries si SQL brut
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userEmail]
);
```

## Audit & Logging

### Audit Trail

```typescript
interface AuditLog {
  id: string;
  organizationId: string;
  action: string;
  resourceType: string;
  resourceId: string;
  userId: string;
  oldValues: Record<string, any>;
  newValues: Record<string, any>;
  ipAddress: string;
  userAgent: string;
  timestamp: Date;
}
```

### Structured Logging

```typescript
// Winston logger
logger.info('User login', {
  userId: user.id,
  ipAddress: req.ip,
  userAgent: req.headers['user-agent'],
  timestamp: new Date(),
  organizationId: user.organizationId
});
```

## Compliance

### RGPD (GDPR)

- ✅ Data minimization
- ✅ Purpose limitation
- ✅ Storage limitation
- ✅ Right to deletion
- ✅ Data portability
- ✅ Breach notification

### SOC 2 Type II

- ✅ Security controls
- ✅ Availability
- ✅ Processing integrity
- ✅ Confidentiality
- ✅ Privacy

## Incident Response

### Breach Notification

```
1. Detect → Alert team immediately
2. Isolate → Stop the bleeding
3. Investigate → Root cause analysis
4. Notify → Users within 72 hours
5. Fix → Patch vulnerabilities
6. Review → Update procedures
```

## Security Checklist

```
✅ All passwords hashed (bcrypt)
✅ API keys rotated monthly
✅ TLS 1.3 enforced
✅ HTTPS only
✅ HSTS enabled
✅ CSP headers set
✅ CORS configured
✅ Rate limiting active
✅ Audit logs enabled
✅ Backups encrypted
✅ Monitoring 24/7
✅ Incident response plan
✅ Regular penetration testing
✅ Dependency updates automated
✅ Secrets not in git
```

---

Security updates: [`SECURITY.md`](../SECURITY.md)
