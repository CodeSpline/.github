# CodeSpline Tech Stack Reference

**Status**: Organization-level reference document for all CodeSpline projects using the core Turborepo + Next.js + NestJS + PostgreSQL tech stack.

**Purpose**: Explain why each core technology was chosen, trade-offs, and alternatives considered. Use this as the foundation when documenting individual project architecture decisions.

**Scope**: This document covers the stable, proven core tech stack used across CodeSpline projects. Project-specific extensions (Elasticsearch, RabbitMQ, Framer Motion, etc.) are documented in individual project architecture folders.

---

## Monorepo Engine — Turborepo

**Where it fits:**

- Root orchestration for multiple apps (frontend, backend, workers)
- Shared package management (ui, types, config)
- Task graph execution (dev, build, lint, type-check, test) across the workspace
- Remote caching for CI/CD acceleration

**Why it fits:**

- **Lightweight and lean**: Unlike Nx, Turborepo has minimal overhead and no forced conventions. It stays out of your way.
- **pnpm integration**: Works seamlessly with pnpm workspaces (lightweight, disk-efficient).
- **Task orchestration**: Understands dependencies between tasks — can run builds in parallel where safe, sequentially where needed.
- **Remote caching**: Speeds up CI pipelines by caching build outputs across runs.
- **Low migration cost**: If you outgrow Turborepo, projects can run independently without major restructuring.

**Trade-offs:**

- Less opinionated than Nx (which can be a feature or a bug depending on team preference).
- Requires manual discipline with package boundaries (no built-in enforcement).

**Alternative considered:**

- Nx: More features (enforced architecture, generators), but heavier footprint and stronger opinions.

---

## Frontend Framework — Next.js (App Router)

**Where it fits:**

- Main user-facing application frontend
- Server-side rendering for SEO benefits
- API route handlers for lightweight backend operations
- TypeScript-first React development with Tailwind CSS

**Why it fits:**

- **App Router stability**: Mature enough for production use with rich features (layouts, streaming, suspense integration).
- **Built-in file routing**: Reduces boilerplate — routes map to the file system.
- **TypeScript-first**: Catches errors at build time.
- **Tailwind CSS v4**: Utility-first CSS with good DX and small bundle sizes.
- **Vercel ecosystem**: Optimized for deployment on Vercel or self-hosted via Docker.
- **React 19 compatibility**: Latest React features (Server Components, Actions, transitions).

**Trade-offs:**

- Steeper learning curve than Create React App if migrating teams.
- App Router is newer — some ecosystem libraries lag support.

**Alternative considered:**

- React + Vite: Lighter and faster dev server, but no file-based routing or server components.
- Remix: Similar capabilities, less ecosystem adoption.

---

## Component System — Astryx

**Where it fits:**

- Reusable React component primitives in `packages/ui`
- Button, Input, Dialog, Card, Layout, Icon wrappers
- Themeable UI building blocks shared across applications
- Foundation for admin interfaces and dashboards

**Why it fits:**

- **Designed for customization**: Not a rigid visual system; intended to be themed to your brand.
- **Accessible by default**: WCAG AA compliance baked in.
- **Production-tested**: Used by multiple large projects.
- **Good size**: Large enough library to reduce boilerplate, small enough to understand and customize.
- **React + TypeScript first**: Native support, no bundler gymnastics.

**Trade-offs:**

- Less pre-built domain components than Material-UI (fewer chart widgets, tables, forms out of the box).
- Maintenance overhead: keep Astryx up-to-date as React and dependencies evolve.

**Alternative considered:**

- shadcn/ui: Copy-paste components (less maintenance, more code duplication).
- Material-UI: Heavier, more rigid theming, larger bundle.

---

## Icon Library — Lucide React

**Where it fits:**

- Navigation and action buttons (menu, settings, search, filter, etc.)
- Status indicators and metadata UI
- UI affordances (chevrons, arrows, check marks)
- Application-specific icons and visual markers

**Why it fits:**

- **Tree-shakable**: Only imported icons are bundled — no unused SVGs in production.
- **Large library**: 500+ icons covering most common UI needs.
- **React-native**: Works naturally in JSX with props (size, color, stroke-width).
- **TypeScript support**: Type-safe icon names.
- **Simple system**: No complex theming — icons inherit color from CSS.

**Trade-offs:**

- Fixed icon set — custom icons must be built separately or sourced elsewhere.
- Less style variation than some icon libraries (all icons follow consistent stroke weight and design).

**Alternative considered:**

- React Icons: Larger, harder to tree-shake; multiple icon sets increase bundle.
- Custom SVG icons: Full control but high maintenance cost.

---

## Server-Side State & Data Fetching — TanStack Query

**Where it fits:**

- Fetching and caching API responses
- Synchronizing with backend when data changes
- Automatic request deduplication and stale-while-revalidate patterns
- Background refetches when browser regains focus

**Why it fits:**

- **Separates concerns**: Async server state handled by Query, UI state stays in React.
- **Built-in caching and sync**: No manual `useEffect` management or race conditions.
- **Devtools**: Query DevTools help debug data flow in development.
- **Suspension + streaming**: Integrates with Next.js App Router Suspense for streaming UI updates.

**Trade-offs:**

- Added dependency in the client (lightweight but still a choice).
- Requires discipline — easy to misuse if you treat server state like UI state.

**Alternative considered:**

- SWR (Vercel): Lighter, simpler, works well for simpler apps; less powerful than Query.
- Raw fetch + useState: Tempting but leads to race conditions and boilerplate over time.

---

## Data Tables & Grids — TanStack Table

**Where it fits:**

- Data lists (searchable, sortable, paginated)
- Dashboard tables and reporting views
- Audit log display
- Any tabular data that needs sorting, filtering, column hiding

**Why it fits:**

- **Headless design**: Brings logic (sorting, filtering, pagination) without imposing UI.
- **Works with Astryx**: Easy to build custom table UI using Astryx primitives.
- **Lightweight**: Core library is small; you pay for what you use.
- **TypeScript-first**: Columns and data are type-safe.

**Trade-offs:**

- Requires more setup than a pre-built Table component (you build the UI).
- Not a charting library — for complex data visualization, you'll need recharts or similar.

**Alternative considered:**

- ag-Grid: Full-featured but expensive for large-scale use, heavier bundle.
- Pre-built Material-UI DataGrid: Less flexible, harder to customize.

---

## Forms & Validation — React Hook Form + Zod

**Where it fits:**

- Any form (user creation, configurations, settings)
- Validation dialogs and workflows
- Search filters and input forms
- Admin configuration and management interfaces

**Why it fits:**

- **Performance**: React Hook Form minimizes re-renders, critical for large forms.
- **Zod for runtime validation**: Type-safe schema validation with automatic TypeScript inference.
- **Works with Next.js Actions**: Hook Form plays well with server actions for form submission.
- **Minimal boilerplate**: Less code than older form libraries.

**Trade-offs:**

- Learning curve if team is used to Formik or other libraries.
- Zod validation must run both client-side and server-side (no automatic server inheritance).

**Alternative considered:**

- Formik: More mature but heavier; more boilerplate.
- Server-side validation only: Slower feedback, poor UX.

---

## Backend Framework — NestJS

**Where it fits:**

- Core API gateway serving frontend applications
- Modular domain services with clear boundaries
- Dependency injection for testability
- Swagger/OpenAPI auto-generated documentation
- TypeScript-first backend development

**Why it fits:**

- **Modular by design**: Modules, providers, decorators enforce boundaries naturally.
- **Batteries included**: Built-in logging, validation, dependency injection, OpenAPI generation.
- **TypeScript-first**: Full type safety across backend, database, and frontend.
- **Testing**: Built-in testing utilities (Test module, Supertest integration).
- **Scalability path**: Modular structure supports evolution to microservices if needed.

**Trade-offs:**

- Larger footprint than Express or Fastify for simple APIs.
- Steeper learning curve for decorator-based architecture.
- Requires discipline to avoid "everything in every module" anti-patterns.

**Alternative considered:**

- Express + custom middleware: Lightweight but requires manual wiring for everything.
- Fastify: Faster, but less out-of-the-box structure; still requires many decisions.

---

## Database & ORM — Prisma + PostgreSQL

**Where it fits:**

- Relational data modeling and queries
- Type-safe data access from NestJS services
- Migrations and schema versioning
- Multi-tenant data scoping (if needed)

**Why it fits:**

- **TypeScript native**: Schema defines both database and TypeScript types; changes in one propagate to the other.
- **Migrations**: `prisma migrate dev` keeps schema and code in sync.
- **Query building**: Type-safe queries prevent SQL injection and typos.
- **PostgreSQL power**: JSON columns, array types, window functions, full-text search — all accessible via Prisma.
- **Good DX**: Prisma Studio for visual database inspection.

**Trade-offs:**

- N+1 query risk if not careful with eager loading.
- Complex queries may fall back to raw SQL (which is fine when needed).
- Lock-in: Migrating off Prisma requires rewriting queries.

**Alternative considered:**

- TypeORM: More traditional ORM, heavier API surface, steeper learning curve.
- Raw SQL (node-postgres): Full control, no abstractions, more code.
- Drizzle: Newer, more SQL-like, lighter than Prisma; trade-off is less polish.

---

## Cache Layer — Redis

**Where it fits:**

- BullMQ queue broker (if using async job processing)
- Dashboard query caching
- Session store
- Rate limiting counters
- Real-time metrics aggregation

**Why it fits:**

- **In-memory speed**: Sub-millisecond latency for cache hits.
- **Multi-use**: Can serve dual purpose (queue broker + cache), reducing infrastructure complexity.
- **Commands-based**: SET, GET, HSET, INCR, etc. are simple and fast.
- **Expiration support**: Automatic key expiration prevents cache bloat.
- **Monitoring**: Redis CLI and observability exporters provide visibility.

**Trade-offs:**

- Data is volatile — server restart loses cache (acceptable for non-critical caches; use AOF persistence if needed).
- Single-server Redis has no replication by default (Redis Cluster or Sentinel for HA).
- Memory-bound — cannot cache datasets larger than available RAM.

**Alternative considered:**

- Memcached: Simpler but fewer features (no expiration, no data structures).
- Application-level cache: Loses cache on process restart.

---

## Testing — Vitest + Playwright (Frontend), Jest + Supertest (Backend)

**Where it fits:**

- Frontend unit tests and component tests (Vitest)
- Frontend end-to-end tests (Playwright)
- Backend unit tests and service tests (Jest)
- Backend integration tests (Supertest)

**Why it fits:**

- **Vitest for frontend**: Vite-native, matches Next.js build setup, fast feedback loop.
- **Playwright for e2e**: Cross-browser testing (Chrome, Firefox, Safari), reliable, good debugging.
- **Jest for backend**: Standard for Node.js testing, excellent test runner, snapshot testing.
- **Supertest for API tests**: Lightweight HTTP assertion library, works with Express-like servers.

**Trade-offs:**

- Maintaining test suite requires discipline — tests rot if not kept in sync with code.
- Playwright tests run slower than unit tests; need careful trade-offs for CI/CD speed.

**Alternative considered:**

- Cypress for e2e: Heavier, fewer browsers; slower feedback.
- Testing Library alone: Good for component tests, overkill if using Vitest + Playwright combo.

---

## Containerisation & Orchestration — Docker + Kubernetes

**Where it fits:**

- Local development: `docker-compose` with PostgreSQL, Redis services
- Production deployment: Docker images for all services
- Kubernetes orchestration: Pod scaling, health checks, rolling updates, secrets management
- CI/CD: Build and push images, deploy to cluster

**Why it fits:**

- **Docker standard**: Consistent environment from dev to production.
- **Kubernetes for orchestration**: Industry standard for multi-service deployments.
- **Reproducibility**: Same Dockerfile runs identically on laptop and in production.
- **Scaling**: Kubernetes handles load balancing, health checks, auto-scaling.

**Trade-offs:**

- Operational complexity: Kubernetes has a steep learning curve and non-trivial ongoing management.
- Over-engineering for small teams: If project is dev → single server, Kubernetes is unnecessary.
- Debugging is harder: Container logs, pod lifecycle, network policies add complexity.

**Alternative considered:**

- Docker Swarm: Simpler but less powerful; less community support.
- Heroku: Easy deployment, higher cost, less control.
- Self-managed VMs: Full control, maximum operational burden.

---

## Tech Stack Philosophy

The technology choices in CodeSpline projects prioritize:

1. **TypeScript throughout**: Type safety from database queries to UI components.
2. **Modular boundaries**: Each service, package, and app can evolve independently.
3. **Open-source and self-hosted**: No vendor lock-in; deployable anywhere.
4. **Proven in production**: Each technology has 100+ large deployments; not experimental.
5. **Developer experience**: Tools that reduce boilerplate, give fast feedback, and are pleasant to use.
6. **Scalability path**: Monolithic starting point, but modular structure supports microservices extraction if needed.

---

## Using This Reference

When starting a new CodeSpline project:

1. **Keep core choices as-is** (Turborepo, Next.js, NestJS, PostgreSQL, Vitest/Jest) unless you have specific reasons to deviate.
2. **Extend with domain-specific tech** (Elasticsearch for search-heavy apps, RabbitMQ for event-driven flows, Prometheus/Grafana for observability, etc.).
3. **Document deviations** in your project's architecture docs — explain why you chose Fastify instead of NestJS, or Svelte instead of Next.js.
4. **Update versions** as needed, but test thoroughly when upgrading major versions.
5. **Reference this document** from your project's architecture README, pointing to this shared file for tech rationale.

Example reference in your project's `docs/architecture/11-technology-choices.md`:

```markdown
## Shared Tech Stack Rationale

For detailed "why" explanations on core technologies (Turborepo, Next.js, NestJS, PostgreSQL, Vitest, etc.), 
see the organization-level [TECH-STACK-REFERENCE.md](../../../TECH-STACK-REFERENCE.md).

Project-specific extensions and customizations are documented below:
```

---

## Version Reference (Last Updated: 2026-07-23)

- Node.js: >= 22
- pnpm: >= 9
- Turborepo: latest
- Next.js: 16+
- React: 19+
- NestJS: latest v10
- Prisma: latest
- TypeScript: 5.x
- Vitest: latest
- Jest: latest
- Playwright: latest
- Docker: latest
