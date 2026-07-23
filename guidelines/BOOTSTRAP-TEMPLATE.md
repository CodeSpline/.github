# CodeSpline Bootstrap Template

**Status**: Organization-level template for bootstrapping new CodeSpline projects using the core Turborepo + Next.js + NestJS + PostgreSQL stack.

**Purpose**: Step-by-step guide from empty repository to running full-stack application. Copy this template and replace `{{PLACEHOLDER}}` values with your project-specific details.

**How to use**: Copy this file to your new project's `docs/bootstrap-reference.md` and replace all placeholders.

---

# {{PROJECT_NAME}} — Bootstrap Reference

Complete setup reference from an empty repository to a running full-stack baseline.

Project:

- **{{PROJECT_NAME}}**
- {{PROJECT_DESCRIPTION}}
- Organization: CodeSpline

Purpose:

- give new contributors a reusable bootstrap path
- capture base prerequisites and tooling decisions
- provide a reference architecture before feature implementation

---

## 1. Project Overview

{{PROJECT_OVERVIEW_PARAGRAPH}}

Key capabilities:

- {{CAPABILITY_1}}
- {{CAPABILITY_2}}
- {{CAPABILITY_3}}
- {{CAPABILITY_4}}
- {{CAPABILITY_5}}

The system uses:

- **Frontend**: Next.js (App Router) with React 19 and Feature-Sliced Design
- **Backend**: NestJS modular monolith with clear domain boundaries
- **Data**: PostgreSQL for relational data, Redis for caching
- **Infrastructure**: Docker and Kubernetes for local and production deployments

---

## 2. Required Development Tools

Install the following on your machine:

```bash
Node.js >= 22
pnpm >= 9
git
docker
docker-compose
VS Code (recommended editor)
```

Verification:

```bash
node -v          # Should print v22.x.x or higher
pnpm -v          # Should print 9.x.x or higher
git --version    # Should print git version X.X.X
docker --version # Should print Docker version X.X.X
```

---

## 3. Repository Creation

Create GitHub repository:

- **Organization**: CodeSpline
- **Repository name**: `{{REPOSITORY_NAME}}`
- **Visibility**: Internal (private)
- **Initialize**: Empty (no README, no .gitignore)

Clone locally:

```bash
git clone https://github.com/CodeSpline/{{REPOSITORY_NAME}}.git
cd {{REPOSITORY_NAME}}
```

---

## 4. Monorepo Architecture

{{PROJECT_NAME}} uses:

- **Turborepo** for task orchestration and remote caching
- **pnpm workspaces** for dependency management
- **TypeScript** for type-safe code across frontend and backend

Structure:

```
{{REPOSITORY_NAME}}/
├── apps/
│   ├── web/           # Next.js frontend (App Router)
│   ├── api/           # NestJS API gateway
│   └── worker/        # {{OPTIONAL: BullMQ background workers}}
├── packages/
│   ├── ui/            # {{OPTIONAL: Astryx + Lucide components}}
│   ├── types/         # TypeScript interfaces + Zod
│   └── config/        # Prisma schema + shared config
├── services/
│   ├── {{SERVICE_1}}/
│   ├── {{SERVICE_2}}/
│   └── ...
├── docs/
└── docker-compose.yml
```

---

## 5. Initialize Root Project

Create directory structure:

```bash
mkdir -p apps packages services docs
touch pnpm-workspace.yaml turbo.json docker-compose.yml
touch .gitignore README.md
```

**pnpm-workspace.yaml**:

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'services/*'
```

---

## 6. Configure Turborepo

**turbo.json**:

```json
{
  "$schema": "https://turborepo.org/schema.json",
  "globalDependencies": ["**/.env.local", ".env.local"],
  "tasks": {
    "dev": {
      "cache": false,
      "interactive": true
    },
    "build": {
      "outputs": [".next", "dist"],
      "dependsOn": ["^build"]
    },
    "lint": {
      "outputs": [],
      "cache": false
    },
    "type-check": {
      "outputs": [],
      "cache": false
    },
    "test": {
      "outputs": ["coverage"],
      "cache": false
    }
  }
}
```

---

## 7. Install Turborepo

```bash
pnpm add -w -D turbo
pnpm turbo --version
```

---

## 8. Bootstrap Frontend (Next.js)

Create Next.js application:

```bash
cd apps
npx create-next-app@latest web --typescript --tailwind
# Select these options:
# - App Router: Yes
# - ESLint: Yes
# - Tailwind CSS: Yes
# - src/ directory: Yes
# - Turbopack: No
# - TypeScript: Yes
cd ../
```

---

## 9. Frontend Stack

Add dependencies to `apps/web/package.json`:

```bash
cd apps/web
pnpm add @tanstack/react-query @tanstack/react-table zod react-hook-form axios
pnpm add -D @testing-library/react vitest @vitest/ui @playwright/test
cd ../../
```

**Key libraries**:

- `@tanstack/react-query`: Server state management
- `@tanstack/react-table`: Data tables and grids
- `zod`: Runtime type validation
- `react-hook-form`: Form state management
- `axios`: HTTP client

**Create `apps/web/src` folder structure** (Feature-Sliced Design):

```bash
cd apps/web/src
mkdir -p {app,pages,widgets,features,entities,shared}/{ui,api,utils,config,hooks,types}
cd ../../../
```

---

## 10. Bootstrap Backend (NestJS)

Create NestJS application:

```bash
cd apps
npm install -g @nestjs/cli  # Global NestJS CLI
nest new api --package-manager pnpm
cd ../../
```

---

## 11. Backend Stack

Add dependencies to `apps/api/package.json`:

```bash
cd apps/api
pnpm add @nestjs/config @nestjs/swagger class-transformer class-validator prisma @prisma/client
pnpm add -D @nestjs/testing jest supertest
cd ../../
```

**Key libraries**:

- `@nestjs/config`: Environment configuration
- `@nestjs/swagger`: OpenAPI documentation
- `prisma`: ORM for database access
- `class-validator`: Request validation

---

## 12. Database Setup (Prisma)

Initialize Prisma:

```bash
cd apps/api
pnpm prisma init
```

**Create `.env.local`** (git-ignored):

```bash
DATABASE_URL="postgresql://dev:dev@localhost:5432/{{PROJECT_NAME}}_dev"
REDIS_URL="redis://localhost:6379"
```

**Create initial `prisma/schema.prisma`**:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model {{ENTITY_1}} {
  id        String   @id @default(cuid())
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("{{entity_1s}}")
}

model {{ENTITY_2}} {
  id        String   @id @default(cuid())
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("{{entity_2s}}")
}
```

Initialize database:

```bash
pnpm prisma migrate dev --name init
```

Return to root:

```bash
cd ../../
```

---

## 13. Create Welcome Endpoint (Backend)

**`apps/api/src/app.controller.ts`**:

```typescript
import { Controller, Get } from '@nestjs/common'

@Controller()
export class AppController {
  @Get('/health')
  health() {
    return { status: 'ok', timestamp: new Date() }
  }

  @Get('/api')
  root() {
    return {
      message: 'Welcome to {{PROJECT_NAME}} API',
      version: '0.1.0',
    }
  }
}
```

---

## 14. Frontend-Backend Communication

**`apps/web/src/shared/api/client.ts`**:

```typescript
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001',
  headers: {
    'Content-Type': 'application/json',
  },
})
```

**`apps/web/src/pages/page.tsx`**:

```typescript
'use client'

import { useQuery } from '@tanstack/react-query'
import { apiClient } from '@/shared/api/client'

export default function HomePage() {
  const { data, isLoading } = useQuery({
    queryKey: ['health'],
    queryFn: () => apiClient.get('/health').then(r => r.data),
  })

  return (
    <div>
      <h1>Welcome to {{PROJECT_NAME}}</h1>
      {isLoading ? <p>Loading...</p> : <p>API: {data?.status}</p>}
    </div>
  )
}
```

---

## 15. Development Commands

From repository root:

```bash
# Start all services (frontend + backend)
pnpm turbo run dev

# Start frontend only
pnpm --filter web dev

# Start backend only
pnpm --filter api dev

# Build all
pnpm turbo run build

# Lint all
pnpm turbo run lint

# Type check all
pnpm turbo run type-check

# Test all
pnpm turbo run test
```

**Port mapping**:

- Frontend (Next.js): http://localhost:3000
- Backend (NestJS): http://localhost:3001
- {{OPTIONAL_WORKER}}: Runs in background
- Swagger docs: http://localhost:3001/api

---

## 16. Docker Setup (Local Development)

Create `docker-compose.yml` at repository root:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: {{PROJECT_NAME}}_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

Start services:

```bash
docker-compose up -d
```

Verify:

```bash
docker-compose ps
psql -h localhost -U dev -d {{PROJECT_NAME}}_dev  # Test PostgreSQL
redis-cli -h localhost                            # Test Redis
```

---

## 17. Initial Database Schema

Create your domain entities based on your project's core concepts:

{{DATABASE_SCHEMA_DESCRIPTION}}

Example for a release management system:

```prisma
model Release {
  id          String   @id @default(cuid())
  name        String
  status      String   @default("draft")
  version     String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model {{YOUR_ENTITY}} {
  id        String   @id @default(cuid())
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

Apply migration:

```bash
cd apps/api
pnpm prisma migrate dev --name initial_schema
pnpm prisma studio  # Visual DB browser
```

---

## 18. Git Workflow

Initialize git:

```bash
git add .
git commit -m "Initial commit: bootstrap {{PROJECT_NAME}}"
git branch -M main
git push -u origin main
```

**Branching strategy**:

```mermaid
gitGraph
    commit id: "Initial"
    branch feature/setup
    checkout feature/setup
    commit id: "Add frontend"
    commit id: "Add backend"
    checkout main
    merge feature/setup
    branch feature/entity-1
    checkout feature/entity-1
    commit id: "Implement {{ENTITY_1}}"
    checkout main
    merge feature/entity-1
```

---

## 19. Development Roadmap

**Bootstrap → MVP → Production**

- **Phase 1 (Now)**: Monorepo setup, basic endpoints, database schema
- **Phase 2 (Week 1-2)**: Core features, CRUD operations, basic UI
- **Phase 3 (Week 3-4)**: {{INTEGRATION_1}}, {{INTEGRATION_2}}, authentication
- **Phase 4 (Month 2)**: Testing, deployment pipeline, monitoring
- **Phase 5 (Month 3+)**: {{ADVANCED_FEATURES}}

---

## 20. Shared Packages

Create reusable packages:

### `packages/ui/` (Astryx + Lucide wrappers)

```
packages/ui/
├── button.tsx
├── input.tsx
├── card.tsx
├── lucide-icons/
│   └── index.ts
└── package.json
```

### `packages/types/` (TypeScript + Zod)

```
packages/types/
├── {{entity_1}}.ts
├── {{entity_2}}.ts
└── package.json
```

### `packages/config/` (Prisma schema + settings)

```
packages/config/
├── prisma/
├── constants/
└── package.json
```

---

## 21. Shared Architecture Reference

For patterns and best practices, refer to organization-level guides:

- [Tech Stack Reference](../../../TECH-STACK-REFERENCE.md) — Why each technology was chosen
- [FSD Architecture Guide](../../../FSD-ARCHITECTURE-GUIDE.md) — Frontend layer structure
- [Security Patterns](../../../SECURITY-PATTERNS.md) — Auth, RBAC, audit
- [Observability Stack](../../../OBSERVABILITY-STACK.md) — Metrics, logging, tracing

---

## 22. Bootstrap Completion Checklist

- ✅ Repository created and cloned
- ✅ Monorepo structure initialized (Turborepo + pnpm)
- ✅ Next.js frontend running on localhost:3000
- ✅ NestJS backend running on localhost:3001
- ✅ PostgreSQL and Redis running in Docker
- ✅ Prisma schema created and migrated
- ✅ Frontend-backend communication working
- ✅ Welcome/health endpoint responding
- ✅ `pnpm turbo run dev` starts all services
- ✅ Initial database schema created
- ✅ Git repository initialized and first commit pushed
- ✅ `.env.local` created and git-ignored
- ✅ Team onboarded and comfortable with setup
- ✅ CI/CD pipeline configured (GitHub Actions / Azure DevOps)
- ✅ Architecture documentation linked from README

**Next steps**:

1. Read `docs/architecture/readme.md` for deep architectural understanding
2. Read `docs/setup-architecture.md` for local development workflow
3. Start implementing core features (Phase 2)
4. Set up monitoring and observability
5. Plan authentication and security hardening

---

**Last Updated**: 2026-07-23

**Template Usage**: Copy this file to a new project's `docs/bootstrap-reference.md` and replace all `{{PLACEHOLDER}}` values with project-specific information. Keep the structure and section numbers consistent for consistency across projects.
