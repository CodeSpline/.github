# CodeSpline Organizational Guidelines

Shared reference documents for all CodeSpline projects using the standardized tech stack.

These files define how we approach architecture, frontend design, security, observability, and bootstrapping across all projects in the organization. They are intended to be **referenced** from individual project documentation rather than duplicated.

---

## Core References

### [TECH-STACK-REFERENCE.md](./TECH-STACK-REFERENCE.md)
**Why we chose each core technology and the trade-offs involved.**

- Turborepo, Next.js, Astryx, Lucide, TanStack Query/Table
- React Hook Form + Zod, NestJS, PostgreSQL + Prisma
- Redis, Vitest + Playwright, Jest + Supertest, Docker + Kubernetes
- Each section explains: "Where it fits", "Why it fits", trade-offs, and alternatives

**Usage**: Reference from your project's `docs/architecture/11-technology-choices.md`. Document project-specific extensions (e.g., Elasticsearch, RabbitMQ, BullMQ, Framer Motion) separately.

---

### [FSD-ARCHITECTURE-GUIDE.md](./FSD-ARCHITECTURE-GUIDE.md)
**Complete Feature-Sliced Design guidance for Next.js frontend applications.**

- 6-layer architecture (`app`, `pages`, `widgets`, `features`, `entities`, `shared`)
- Dependency rules and visual graphs
- Folder organization best practices
- Common mistakes and migration strategies
- Directory depth guidelines and naming conventions

**Usage**: Reference from your project's `docs/architecture/08-frontend-architecture.md` (or `06-frontend-architecture.md` in some projects). Customize layer examples for your domain.

---

### [SECURITY-PATTERNS.md](./SECURITY-PATTERNS.md)
**Authentication, authorization, audit trails, and secrets management patterns.**

- SSO / OAuth 2.0 / OIDC with identity providers (Azure AD, Okta, Google)
- JWT configuration and session management
- RBAC with role definitions and database schema examples
- Append-only audit trails for compliance
- Secrets management for local, staging, and production
- API security (Helmet, CORS, rate limiting, input validation)
- Evolution path from basic auth to enterprise SSO

**Usage**: Reference from your project's `docs/architecture/09-security.md`. Customize roles, IdP choice, and secrets manager for your deployment environment.

---

### [OBSERVABILITY-STACK.md](./OBSERVABILITY-STACK.md)
**Metrics collection, structured logging, distributed tracing, dashboards, and alerting.**

- Prometheus metrics architecture and instrumentation patterns
- Structured JSON logging with standard fields
- W3C Trace Context for distributed tracing
- Grafana dashboards (YAML-as-code, provisioned dashboards)
- Alert rules in YAML format
- Evolution path from basic metrics to ML-driven insights

**Usage**: Reference from your project's `docs/architecture/10-observability.md`. Customize metrics, dashboard names, and alert thresholds based on your service characteristics.

---

### [BOOTSTRAP-TEMPLATE.md](./BOOTSTRAP-TEMPLATE.md)
**Step-by-step template for bootstrapping new projects from an empty repository.**

- 22 sections covering monorepo setup, framework bootstrap, backend/frontend stacks, database, Docker, CI/CD, and more
- 22 placeholder variables (`{{PROJECT_NAME}}`, `{{REPOSITORY_NAME}}`, etc.) for project-specific customization
- Same structure across all CodeSpline projects ensures consistency

**Usage**: 
1. Copy to your new project: `cp BOOTSTRAP-TEMPLATE.md your-project/docs/bootstrap-reference.md`
2. Replace all `{{PLACEHOLDER}}` values with project-specific details
3. Follow the step-by-step guide to set up your monorepo
4. Delete or preserve placeholders for optional components (workers, services, integrations)

---

## How to Use These Guidelines

### For Existing Projects

Update your project docs to reference the shared guidelines:

**In `docs/architecture/11-technology-choices.md`:**
```markdown
## Shared Tech Stack Reference

For detailed rationale on all core technologies (Turborepo, Next.js, NestJS, PostgreSQL, etc.),
see the organization-level [TECH-STACK-REFERENCE.md](../../../../.github/guidelines/TECH-STACK-REFERENCE.md).

Project-specific extensions:

### RabbitMQ (Event Messaging)
...

### Elasticsearch (Full-Text Search)
...
```

**In `docs/architecture/08-frontend-architecture.md` (or `06-frontend-architecture.md`):**
```markdown
## Frontend Architecture — Feature-Sliced Design

For complete FSD guidance including layer definitions, dependency rules, and folder organization,
see the organization-level [FSD-ARCHITECTURE-GUIDE.md](../../../../.github/guidelines/FSD-ARCHITECTURE-GUIDE.md).

Project-specific customizations:
...
```

**In `docs/architecture/09-security.md`:**
```markdown
## Security Patterns

For comprehensive security patterns including authentication, RBAC, audit, and secrets management,
see the organization-level [SECURITY-PATTERNS.md](../../../../.github/guidelines/SECURITY-PATTERNS.md).

Our identity provider: {{YOUR_IDP}}  
Secrets manager: {{YOUR_SECRETS_MANAGER}}
```

### For New Projects

1. Create a new repository with the CodeSpline tech stack
2. Copy `BOOTSTRAP-TEMPLATE.md` to `docs/bootstrap-reference.md`
3. Replace all `{{PLACEHOLDER}}` values
4. Reference the other 4 guides in your architecture documentation:
   - `docs/architecture/11-technology-choices.md` → links to TECH-STACK-REFERENCE.md
   - `docs/architecture/08-frontend-architecture.md` → links to FSD-ARCHITECTURE-GUIDE.md
   - `docs/architecture/09-security.md` → links to SECURITY-PATTERNS.md
   - `docs/architecture/10-observability.md` → links to OBSERVABILITY-STACK.md

---

## Projects Using These Standards

- **Hearth & Lens** — Editorial storytelling platform
- **Release Intelligence Portal** — Internal release tracking
- *{{Future projects here}}*

---

## Maintenance

When updating these shared guidelines:

1. Update the file in `.github/guidelines/`
2. Notify all projects using these standards (via issue or announcement)
3. Projects should review the changes and merge them at their convenience
4. No breaking changes without coordination across all projects

---

## Contributing

To suggest improvements or report issues with these guidelines:

1. Open an issue in the organization `.github` repository
2. Reference the specific guide and section
3. Provide context on why the change is needed
4. Wait for team approval before making changes

---

**Last Updated**: 2026-07-23
