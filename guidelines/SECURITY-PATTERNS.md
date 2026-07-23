# CodeSpline Security Patterns Reference

**Status**: Organization-level reference for security implementation patterns across CodeSpline projects.

**Purpose**: Provide consistent guidance on authentication, authorization, audit, and secrets management.

**Scope**: This guide applies to all CodeSpline backend services (NestJS) and supporting infrastructure. Projects may extend with additional security measures.

---

## Overview

Security in CodeSpline projects is built on four pillars:

1. **Identity**: Who is the user? (Authentication)
2. **Access**: What can they do? (Authorization)
3. **Audit**: What did they do? (Audit trail)
4. **Secrets**: How do we protect sensitive data? (Secrets management)

---

## Identity & Authentication

### SSO (Single Sign-On)

**Primary method**: OAuth 2.0 / OIDC via enterprise identity provider

**Supported providers**:

- Azure AD / Microsoft Entra ID
- Okta
- Google Workspace
- Generic OIDC providers

**Flow**:

```
1. User clicks "Sign In"
2. Redirected to IdP (e.g., Azure AD)
3. User authenticates with IdP
4. IdP returns authorization code
5. Backend exchanges code for access token + refresh token (server-side)
6. Backend creates session and returns short-lived JWT to frontend
7. Frontend stores JWT in memory (not localStorage for security)
8. Subsequent requests include JWT in Authorization header
```

**Implementation**:

- Use industry-standard libraries: `@nestjs/passport`, `passport-azure-ad`, `passport-okta`
- Short-lived JWT (15-30 minutes)
- Refresh token rotation (stored securely server-side)
- No sensitive data in JWT payload

### Local Development

For local development, disable SSO and use mock authentication:

```typescript
// NestJS auth guard (development mode)
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    if (process.env.NODE_ENV === 'development') {
      // Mock user for local development
      context.switchToHttp().getRequest().user = {
        id: 'dev-user-id',
        email: 'dev@example.com',
        roles: ['admin'],
      }
      return true
    }
    // Production: actual SSO validation
    return validateJWT(context)
  }
}
```

### JWT Configuration

- **Signing algorithm**: RS256 (asymmetric, public key verification)
- **Issuer**: Your backend domain
- **Audience**: Your frontend domain
- **Expiration**: 15-30 minutes
- **Not-before**: Optional, set to issue time
- **Subject**: User ID from identity provider

---

## Authorization (RBAC)

### Role-Based Access Control

**Default roles** (customize per project):

| Role | Description | Capabilities |
|------|-------------|--------------|
| Viewer | Read-only access | View dashboards, reports, releases |
| Engineer | Full operational access | Create/update/deploy releases |
| Team Lead | Team management | Manage team members, team config |
| Admin | System-wide administration | User management, system config, integrations |

**Role assignment**:

- Assigned by IdP (Azure AD groups, Okta app roles)
- Synced to user record in database on login
- Can be overridden locally for specific users (if admin allows)

### Implementation

**Database schema**:

```sql
CREATE TABLE "user" (
  id UUID PRIMARY KEY,
  email VARCHAR UNIQUE NOT NULL,
  idp_id VARCHAR UNIQUE NOT NULL,
  roles TEXT[] NOT NULL, -- ['engineer', 'team-lead']
  updated_at TIMESTAMP,
  created_at TIMESTAMP
);

CREATE TABLE "user_role_permission" (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES "user"(id),
  role VARCHAR NOT NULL,
  resource_type VARCHAR NOT NULL, -- 'release', 'team', 'config'
  resource_id VARCHAR,
  permission VARCHAR NOT NULL -- 'read', 'write', 'delete'
);
```

**NestJS guard**:

```typescript
@Injectable()
export class RoleGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest()
    const user = request.user
    const requiredRole = Reflect.getMetadata('roles', context.getHandler())
    
    if (!requiredRole) return true // No role required
    return user.roles.some(role => requiredRole.includes(role))
  }
}

// Usage
@Post('/releases')
@UseGuards(RoleGuard)
@SetMetadata('roles', ['engineer', 'admin'])
createRelease(@Body() dto: CreateReleaseDto) {
  // Only engineers and admins can create releases
}
```

### Fine-Grained Permissions (Future)

As projects grow, move from roles to fine-grained permissions:

```
User → Roles → Permissions → Resources
```

Example: "User with Engineer role on Team A can update releases for Product X"

---

## Audit Trail

### Append-Only Audit Log

**Purpose**: Immutable record of all actions for compliance and debugging.

**Design**:

```sql
CREATE TABLE "audit_event" (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  actor_id UUID NOT NULL, -- User who performed action
  action VARCHAR NOT NULL, -- 'created', 'updated', 'deleted', 'approved', 'deployed'
  resource_type VARCHAR NOT NULL, -- 'release', 'team', 'config'
  resource_id VARCHAR NOT NULL,
  resource_name VARCHAR, -- Snapshot of resource name at time of action
  old_value JSONB, -- State before change
  new_value JSONB, -- State after change
  metadata JSONB, -- Additional context (IP, user-agent, etc.)
  timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE(id) -- Enforce append-only: no updates
);

CREATE INDEX idx_audit_tenant_resource ON audit_event(tenant_id, resource_type, resource_id);
CREATE INDEX idx_audit_timestamp ON audit_event(timestamp);
```

### Captured Events (Examples)

- **Release management**: created, updated, status_changed, approved, deployed, cancelled
- **User management**: user_created, user_role_changed, user_deactivated
- **Configuration**: config_created, config_updated, integration_enabled, integration_disabled
- **Integrations**: sync_started, sync_completed, sync_failed, notification_sent
- **Access**: access_denied, permission_changed, session_created, session_revoked

### Audit Service (NestJS)

```typescript
@Injectable()
export class AuditService {
  constructor(private prisma: PrismaService) {}

  async log(
    tenantId: string,
    actorId: string,
    action: string,
    resourceType: string,
    resourceId: string,
    options: {
      resourceName?: string
      oldValue?: Record<string, any>
      newValue?: Record<string, any>
      metadata?: Record<string, any>
    }
  ) {
    await this.prisma.auditEvent.create({
      data: {
        tenant_id: tenantId,
        actor_id: actorId,
        action,
        resource_type: resourceType,
        resource_id: resourceId,
        resource_name: options.resourceName,
        old_value: options.oldValue,
        new_value: options.newValue,
        metadata: options.metadata,
        timestamp: new Date(),
      },
    })
  }
}

// Usage in controller
@Post('/releases')
async createRelease(@Req() request, @Body() dto: CreateReleaseDto) {
  const release = await this.releaseService.create(dto)
  
  await this.auditService.log(
    request.user.tenant_id,
    request.user.id,
    'created',
    'release',
    release.id,
    {
      resourceName: release.name,
      newValue: release,
      metadata: {
        ip: request.ip,
        userAgent: request.headers['user-agent'],
      }
    }
  )
  
  return release
}
```

### Audit Access Control

- **Only admins** can export/view full audit logs
- **Users** can view audit history for their own resources
- **Audit logs are append-only** — cannot be modified or deleted
- **Retention policy** — keep for X years per compliance requirements

---

## Secrets Management

### Principles

1. **Never commit secrets** to git (no private keys, passwords, API tokens)
2. **Encrypt at rest**: AES-256 encryption for sensitive fields
3. **Rotate regularly**: Passwords every 90 days, API keys every 6 months
4. **Principle of least privilege**: Services only have access to secrets they need
5. **Audit access**: Log who accessed what secrets and when

### Sensitive Data

**Categories**:

- Database credentials (PostgreSQL, Redis connection strings)
- API keys (Jira, GitHub, Azure DevOps, Slack, email services)
- OAuth client secrets (for IdP integration)
- Encryption keys (data encryption, signing)
- SSL/TLS certificates
- SSH keys (for deployment)

### Local Development

Use `.env.local` (git-ignored):

```bash
# .env.local
DATABASE_URL="postgresql://dev:dev@localhost:5432/myapp_dev"
REDIS_URL="redis://localhost:6379"
JIRA_API_KEY="your-local-test-key"
```

**Never commit `.env.local` to git.**

### Production Deployment

Use external secrets manager:

**Options**:

- AWS Secrets Manager
- Azure Key Vault
- HashiCorp Vault
- Kubernetes Secrets (mounted as files)

**Pattern**:

1. Secrets manager stores encrypted secrets
2. Service authenticates to secrets manager on startup
3. Secrets mounted as environment variables or files
4. Application reads from environment
5. Secrets never logged or exposed

**Example (AWS Secrets Manager)**:

```typescript
// NestJS config module
import * as AWS from 'aws-sdk'

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [
        async () => {
          const secretsManager = new AWS.SecretsManager()
          const secret = await secretsManager
            .getSecretValue({ SecretId: 'myapp/prod' })
            .promise()
          return JSON.parse(secret.SecretString)
        },
      ],
    }),
  ],
})
export class ConfigurationModule {}
```

### Encrypted Database Fields

For sensitive user or application data stored in database (not secrets themselves):

```typescript
// Prisma schema
model User {
  id String @id @default(cuid())
  email String @unique
  encrypted_phone String? // Encrypted phone number
  
  @@index([email])
}

// NestJS service with encryption/decryption
import * as crypto from 'crypto'

@Injectable()
export class EncryptionService {
  private cipher = (plaintext: string) => {
    const cipher = crypto.createCipher('aes-256-cbc', process.env.ENCRYPTION_KEY)
    return cipher.update(plaintext, 'utf8', 'hex') + cipher.final('hex')
  }
  
  private decipher = (encrypted: string) => {
    const decipher = crypto.createDecipher('aes-256-cbc', process.env.ENCRYPTION_KEY)
    return decipher.update(encrypted, 'hex', 'utf8') + decipher.final('utf8')
  }
}
```

---

## API Security

### Middleware

**Helmet** (security headers):

```typescript
import helmet from '@nestjs/helmet'

@Module({})
export class AppModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(helmet()).forRoutes('*')
  }
}
```

**CORS** (restrict origins):

```typescript
app.enableCors({
  origin: process.env.FRONTEND_URL,
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
})
```

**Rate Limiting**:

```typescript
import { ThrottlerModule } from '@nestjs/throttler'

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000, // 1 minute
        limit: 100, // 100 requests per minute
      },
    ]),
  ],
})
export class AppModule {}
```

### Input Validation

**Never trust user input.**

- Validate all request bodies (Zod, class-validator)
- Sanitize strings (prevent XSS, SQL injection)
- Limit request size (prevent DoS)

```typescript
import { IsEmail, IsString, MinLength } from 'class-validator'

export class CreateUserDto {
  @IsEmail()
  email: string

  @IsString()
  @MinLength(8)
  password: string
}

// NestJS auto-validates with global validation pipe
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  })
)
```

### HTTPS / TLS

- **Always use HTTPS** in production
- Redirect HTTP → HTTPS
- Use valid SSL/TLS certificates (Let's Encrypt for free)
- Set HSTS header (force browsers to use HTTPS)

---

## Evolution Path

### Phase 1: Basic Authentication (MVP)

- ✅ Basic auth (username/password) or mock auth for local dev
- ✅ Simple RBAC (admin/user roles)
- ✅ Audit trail for major actions

### Phase 2: Enterprise SSO

- ✅ SSO integration (Azure AD / Okta)
- ✅ Fine-grained RBAC (roles + permissions)
- ✅ Comprehensive audit logging

### Phase 3: Secrets Management

- ✅ External secrets manager integration
- ✅ Encrypted sensitive fields in database
- ✅ Secrets rotation automation

### Phase 4: Compliance & Reporting

- ✅ Audit log export and analysis
- ✅ Compliance reporting (SOC2, ISO27001)
- ✅ Data retention policies

---

## Using This Guide in Your Project

Reference this document from your project's `docs/architecture/09-security.md`:

```markdown
## Security Architecture

For complete guidance on authentication, authorization, audit, and secrets management,
see the organization-level [SECURITY-PATTERNS.md](../../../SECURITY-PATTERNS.md).

In our project specifically:
- Authentication method: {{YOUR METHOD}}
- Supported roles: {{YOUR ROLES}}
- Secrets manager: {{YOUR SECRETS MANAGER}}
- Audit retention: {{YOUR RETENTION POLICY}}
```

---

**Last Updated**: 2026-07-23
