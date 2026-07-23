# Feature-Sliced Design (FSD) Architecture Guide

**Status**: Organization-level reference for Feature-Sliced Design adoption in CodeSpline Next.js applications.

**Purpose**: Provide consistent guidance on FSD layer structure, dependencies, and folder organization across all projects.

**Scope**: This guide applies to the `apps/web/src` folder in CodeSpline projects. Backend services use different boundaries (NestJS modules); shared packages use simplified structures.

---

## Overview

Feature-Sliced Design (FSD) is a frontend architecture pattern that organizes code by **user capability** rather than **file type**. Instead of `components/`, `hooks/`, `utils/`, code is organized in layers where each layer has specific responsibilities and strict dependency rules.

**Key principle**: Lower layers never import from higher layers. Data flows downward; dependencies flow downward.

---

## The Six Layers

### 1. `app/` — Application Bootstrap

**Purpose**: Application-wide setup, providers, routing, global state, styling.

**Contains**:

- Root layout and page wrapper
- Context providers (auth, theme, notifications)
- Global CSS and Tailwind config imports
- App-level error boundaries
- Entry point initialization

**Examples**:

```
app/
├── providers/
│   ├── auth-provider.tsx
│   ├── query-provider.tsx
│   ├── theme-provider.tsx
│   └── index.tsx
├── routes/
│   └── (authenticated)/
│       ├── layout.tsx
│       └── page.tsx
├── error.tsx
├── layout.tsx
├── page.tsx
└── globals.css
```

**Rules**:

- Never import from higher layers (there are none above)
- Import from: pages, shared
- No business logic; pure setup and wiring

**Red flags**:

- ❌ Feature business logic in app layer
- ❌ Hardcoded feature routes or pages
- ❌ Feature-specific styling or configuration

---

### 2. `pages/` — Route Entry Points

**Purpose**: Compose page-level layouts and connect to features/entities.

**Contains**:

- Route-level components (one per URL path)
- Page composition (which features/widgets to show)
- URL parameter extraction
- Page-level loading and error states

**Examples**:

```
pages/
├── (public)/
│   ├── layout.tsx
│   └── page.tsx
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx
│   └── [id]/
│       └── page.tsx
├── settings/
│   ├── layout.tsx
│   └── page.tsx
└── not-found.tsx
```

**Rules**:

- Never import from higher layers (app)
- Import from: widgets, features, entities, shared
- One component per route; no re-exporting widgets
- Keep pages thin — they compose, don't implement

**Red flags**:

- ❌ Business logic in page components
- ❌ Long conditionals or state management
- ❌ Direct API calls (use features instead)

---

### 3. `widgets/` — Feature Blocks

**Purpose**: Large, reusable interface blocks composed from features and entities.

**Contains**:

- Feature cards, hero sections, sidebars
- Dashboard panels, data visualizations
- Complex layout blocks
- Re-usable page sections

**Examples**:

```
widgets/
├── release-card/
│   ├── release-card.tsx
│   ├── release-card.module.css
│   └── index.ts
├── deployment-status/
│   ├── deployment-status.tsx
│   ├── use-deployment-status.ts
│   └── index.ts
├── analytics-panel/
│   ├── analytics-panel.tsx
│   ├── chart-utils.ts
│   └── index.ts
└── sidebar/
    ├── sidebar.tsx
    └── index.ts
```

**Rules**:

- Import from: features, entities, shared
- Never import from: pages, app
- Widgets are reusable across pages
- Each widget has a clear, single purpose

**Red flags**:

- ❌ Page-specific logic in widgets
- ❌ Importing from pages
- ❌ Hard-coded data or styling

---

### 4. `features/` — User Actions

**Purpose**: User-facing business logic and interactions.

**Contains**:

- Create, update, delete workflows
- Search, filter, sort implementations
- Form workflows (login, signup, etc.)
- State management for user actions
- API calls and async operations

**Examples**:

```
features/
├── create-release/
│   ├── create-release.tsx
│   ├── use-create-release.ts
│   ├── create-release.schema.ts
│   ├── dto/
│   └── index.ts
├── approve-release/
│   ├── approve-release.tsx
│   ├── use-approve-release.ts
│   └── index.ts
├── search-releases/
│   ├── search-releases.tsx
│   ├── use-search-releases.ts
│   └── index.ts
└── filters/
    ├── release-filters.tsx
    └── index.ts
```

**Rules**:

- Import from: entities, shared
- Never import from: pages, widgets, app
- Each feature is a user capability
- Features handle async operations (API calls, mutations)
- Features manage local state for that action

**Red flags**:

- ❌ Importing from pages or widgets
- ❌ Hard-coded entity logic (belong in entities)
- ❌ Shared UI components (belong in shared)

---

### 5. `entities/` — Domain Models

**Purpose**: Stable domain concepts and their associated UI and utilities.

**Contains**:

- Type definitions and interfaces
- API client mappings
- Domain-specific utilities and helpers
- Small UI fragments tied to that model
- Selectors and data transformations

**Examples**:

```
entities/
├── release/
│   ├── types.ts
│   ├── release.schema.ts
│   ├── api-client.ts
│   ├── use-release.ts
│   ├── release-item.tsx
│   └── index.ts
├── application/
│   ├── types.ts
│   ├── api-client.ts
│   └── index.ts
├── environment/
│   ├── types.ts
│   ├── constants.ts
│   └── index.ts
└── team/
    ├── types.ts
    └── index.ts
```

**Rules**:

- Import from: shared only
- Never import from: features, widgets, pages, app
- Entities are stable — change only when domain changes
- UI fragments in entities are small and non-interactive (usually just display)

**Red flags**:

- ❌ Importing from features
- ❌ Business logic or state management
- ❌ Feature-specific components

---

### 6. `shared/` — Cross-Cutting Utilities

**Purpose**: Genuinely reusable code not tied to one feature or entity.

**Contains**:

- UI primitives wrapping Astryx components
- Lucide icon wrappers and icon helpers
- API client base utilities and axios configuration
- Config constants and environment variables
- Date and formatting helpers
- Form adapters and validation utilities
- Custom hooks (useLocalStorage, useDebounce, etc.)

**Examples**:

```
shared/
├── ui/
│   ├── button.tsx
│   ├── input.tsx
│   ├── dialog.tsx
│   ├── card.tsx
│   ├── lucide-icons/
│   │   ├── icon-chevron.tsx
│   │   └── index.ts
│   └── index.ts
├── api/
│   ├── api-client.ts
│   ├── axios-config.ts
│   └── index.ts
├── utils/
│   ├── date-format.ts
│   ├── string-utils.ts
│   ├── array-utils.ts
│   └── index.ts
├── config/
│   ├── constants.ts
│   ├── env.ts
│   └── index.ts
├── hooks/
│   ├── use-local-storage.ts
│   ├── use-debounce.ts
│   └── index.ts
└── types/
    ├── common.ts
    └── index.ts
```

**Rules**:

- Import from: nothing (shared is the lowest layer)
- Only used by layers above
- No feature-specific logic
- Keep foundational, not a catch-all

**Red flags**:

- ❌ Feature-specific code in shared (move to features)
- ❌ Importing from other layers
- ❌ 20+ files in shared (sign it's a catch-all)

---

## Dependency Graph

```
app/
 ↓
pages/
 ↓
widgets ← features
 ↓         ↓
 └→ entities
      ↓
     shared
```

**Rules**:

- ✅ App imports pages
- ✅ Pages import widgets, features, entities, shared
- ✅ Widgets import features, entities, shared
- ✅ Features import entities, shared
- ✅ Entities import shared
- ✅ Shared imports nothing

**Never**:

- ❌ Features importing from widgets or pages
- ❌ Entities importing from features
- ❌ Pages importing from app
- ❌ Widgets importing from pages

---

## Folder Organization Best Practices

### Index Files

Every folder with exports should have an `index.ts`:

```typescript
// features/create-release/index.ts
export { CreateRelease } from './create-release'
export { useCreateRelease } from './use-create-release'
export type { CreateReleaseInput } from './create-release.schema'
```

**Benefit**: Clean imports from other layers.

```typescript
// pages/releases/page.tsx
import { CreateRelease, useCreateRelease } from '@/features/create-release'
```

### Naming Conventions

- **Components**: PascalCase (ReleaseCard.tsx)
- **Hooks**: camelCase with `use` prefix (useRelease.ts)
- **Types**: PascalCase (Release.ts, ReleaseType.ts)
- **Utilities**: camelCase (formatDate.ts, parseError.ts)
- **Constants**: UPPER_SNAKE_CASE (RELEASE_STATUSES.ts)
- **Styles**: `.module.css` co-located with component

### Directory Depth

- Shallow is better: `features/create-release/` (good)
- Avoid deep nesting: `features/release/workflows/create/steps/form/` (bad)
- Max 2-3 levels: `entities/release/` or `entities/release/types/` (OK)

---

## Suggested Folder Shape (by Layer)

```
apps/web/src/
├── app/
│   ├── providers/
│   │   ├── auth-provider.tsx
│   │   ├── query-provider.tsx
│   │   ├── theme-provider.tsx
│   │   └── index.tsx
│   ├── routes/
│   │   └── (authenticated)/
│   │       ├── layout.tsx
│   │       └── page.tsx
│   ├── error.tsx
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
│
├── pages/
│   ├── dashboard/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── [id]/
│   │       └── page.tsx
│   ├── settings/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   └── (public)/
│       ├── layout.tsx
│       └── page.tsx
│
├── widgets/
│   ├── release-card/
│   │   ├── release-card.tsx
│   │   ├── release-card.module.css
│   │   └── index.ts
│   ├── deployment-status/
│   │   ├── deployment-status.tsx
│   │   ├── use-deployment-status.ts
│   │   └── index.ts
│   └── analytics-panel/
│       ├── analytics-panel.tsx
│       ├── chart-utils.ts
│       └── index.ts
│
├── features/
│   ├── create-release/
│   │   ├── create-release.tsx
│   │   ├── use-create-release.ts
│   │   ├── create-release.schema.ts
│   │   ├── dto/
│   │   │   └── create-release.dto.ts
│   │   └── index.ts
│   ├── approve-release/
│   │   ├── approve-release.tsx
│   │   ├── use-approve-release.ts
│   │   └── index.ts
│   └── search-releases/
│       ├── search-releases.tsx
│       ├── use-search-releases.ts
│       └── index.ts
│
├── entities/
│   ├── release/
│   │   ├── types.ts
│   │   ├── release.schema.ts
│   │   ├── api-client.ts
│   │   ├── use-release.ts
│   │   ├── release-item.tsx
│   │   └── index.ts
│   ├── application/
│   │   ├── types.ts
│   │   ├── api-client.ts
│   │   └── index.ts
│   └── team/
│       ├── types.ts
│       └── index.ts
│
└── shared/
    ├── ui/
    │   ├── button.tsx
    │   ├── input.tsx
    │   ├── card.tsx
    │   ├── dialog.tsx
    │   ├── lucide-icons/
    │   │   ├── icon-chevron.tsx
    │   │   ├── icon-check.tsx
    │   │   └── index.ts
    │   └── index.ts
    ├── api/
    │   ├── api-client.ts
    │   ├── axios-config.ts
    │   └── index.ts
    ├── utils/
    │   ├── date-format.ts
    │   ├── string-utils.ts
    │   └── index.ts
    ├── config/
    │   ├── constants.ts
    │   ├── env.ts
    │   └── index.ts
    ├── hooks/
    │   ├── use-local-storage.ts
    │   ├── use-debounce.ts
    │   └── index.ts
    └── types/
        └── common.ts
```

---

## Common Mistakes & How to Avoid Them

| Mistake | Why It's Wrong | Solution |
|---------|----------------|----------|
| Importing from pages in widgets | Breaks layering; pages depend on widgets, not vice versa | Move logic to features, share via entities |
| Putting business logic in app/ | App layer is only for bootstrap | Move to features |
| Shared becoming a catch-all | Violates single responsibility; makes shared too large | Create new layers or move to entities |
| Deep nesting (3+ levels) | Hard to navigate and import | Keep max 2 levels; use flat structure |
| Feature-specific components in shared | Violates cross-cutting purpose | Move to features if not truly shared |
| Circular imports | Breaks dependency graph | Use index files, check import paths |
| No index files | Harder to import, more brittle | Add index.ts to every folder |

---

## Migration Strategy (for existing projects)

If converting from flat `components/` structure to FSD:

1. **Identify features** (user actions): Create, update, delete, search, filter, etc.
2. **Identify entities** (domain models): The nouns — user, release, application, team, etc.
3. **Identify widgets** (reusable blocks): Cards, panels, sidebars that compose features
4. **Move cross-cutting utilities to shared**: UI wrappers, helpers, hooks
5. **Adopt incrementally**: New code in FSD structure, refactor old code over time

---

## Using This Guide in Your Project

Reference this document from your project's `docs/architecture/08-frontend-architecture.md`:

```markdown
## Feature-Sliced Design

For complete FSD guidance, layer definitions, and folder structure, 
see the organization-level [FSD-ARCHITECTURE-GUIDE.md](../../../FSD-ARCHITECTURE-GUIDE.md).

In our project specifically:
- Domain entities: {{LIST YOUR ENTITIES}}
- Primary features: {{LIST YOUR FEATURES}}
- Reusable widgets: {{LIST YOUR WIDGETS}}
```

---

## Resources

- [Feature-Sliced Design Official Docs](https://feature-sliced.design/)
- [FSD Cheat Sheet](https://feature-sliced.design/en/docs)
- [Best Practices](https://feature-sliced.design/en/docs/guides/examples/better-redux)

---

**Last Updated**: 2026-07-23
