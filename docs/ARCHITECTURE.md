# Paperclip Architecture Overview

> Last updated: 2026-03-28

Paperclip is an orchestration platform for managing autonomous AI agent teams. It provides task management, governance, cost control, and multi-company isolation — letting AI agents operate as structured organizations with reporting hierarchies, budgets, and approval workflows.

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        React Frontend (ui/)                      │
│  Pages · Components · Hooks · React Query · WebSocket Client     │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP / WebSocket
┌──────────────────────────────▼──────────────────────────────────┐
│                     Express API Server (server/)                 │
│  Routes · Middleware · Auth · Services · Plugin Host · Adapters   │
└──────────┬─────────────────┬──────────────────┬─────────────────┘
           │                 │                  │
    ┌──────▼──────┐   ┌─────▼──────┐   ┌──────▼──────┐
    │  PostgreSQL  │   │  Adapters   │   │  Plugins    │
    │  (Drizzle)   │   │  (Agents)   │   │  (Workers)  │
    └─────────────┘   └────────────┘   └─────────────┘
```

### Deployment Topology

The system runs as a single-process Node.js server that optionally embeds PostgreSQL for zero-configuration local development. In production, it connects to an external PostgreSQL instance. The frontend is either served as static files or via Vite dev middleware.

---

## 2. Repository Structure

Paperclip is a **pnpm monorepo** with the following workspace packages:

| Package | Path | Purpose |
|---------|------|---------|
| `@paperclipai/server` | `server/` | Express API server, business logic, plugin host |
| `@paperclipai/ui` | `ui/` | React SPA frontend |
| `@paperclipai/db` | `packages/db/` | Drizzle ORM schema, migrations, database client |
| `@paperclipai/shared` | `packages/shared/` | Shared types, validators, constants |
| `@paperclipai/adapter-utils` | `packages/adapter-utils/` | Common adapter utilities |
| `@paperclipai/plugin-sdk` | `packages/plugins/sdk/` | Plugin development SDK |
| `paperclipai` | `cli/` | Command-line interface |
| Adapter packages | `packages/adapters/*/` | Per-runtime agent adapters (7 adapters) |

### Scale

- **865 TypeScript/TSX source files**
- **~187,000 lines of TypeScript**
- **60+ database tables** across 45+ migrations
- **7 agent adapters** (Claude, Codex, Cursor, Gemini, OpenCode, Pi, OpenClaw)

---

## 3. Backend Architecture (`server/`)

### 3.1 Entry Point and Startup

**`server/src/index.ts`** (763 lines) — The `startServer()` function orchestrates:

1. **Configuration loading** — Merges file config, environment variables, and defaults via `loadConfig()` (`config.ts`)
2. **Database initialization** — Either connects to external PostgreSQL or launches embedded PostgreSQL with automatic port detection
3. **Migration management** — Inspects, reconciles, and applies pending migrations with interactive prompts
4. **Authentication setup** — Initializes BetterAuth for `authenticated` mode or creates a local board principal for `local_trusted` mode
5. **Express app creation** — Delegates to `createApp()` for route/middleware assembly
6. **WebSocket server** — Sets up live event broadcasting
7. **Scheduler startup** — Heartbeat timer, routine scheduler, and orphan run reaper run on `setInterval`
8. **Database backups** — Optional scheduled backup with retention policy
9. **Graceful shutdown** — SIGINT/SIGTERM handlers stop embedded PostgreSQL cleanly

### 3.2 Express Application (`app.ts`)

The app mounts middleware and routes in this order:

1. `express.json()` with 10MB limit and raw body capture
2. `httpLogger` — Pino-based request logging
3. `privateHostnameGuard` — Host header validation for private deployments
4. `actorMiddleware` — Resolves the authenticated actor (board user, agent, or anonymous)
5. Auth session endpoint + BetterAuth handler
6. LLM proxy routes (outside API router)
7. **API Router** (`/api`):
   - `boardMutationGuard` — Prevents mutations when board is read-only
   - Domain routes: health, companies, agents, projects, issues, routines, workspaces, goals, approvals, secrets, costs, activity, dashboard, settings, plugins, access
8. Plugin UI static serving
9. Frontend serving (static or Vite dev)
10. Error handler middleware

### 3.3 Service Layer

Business logic lives in `server/src/services/` (60+ files). Key services:

| Service | Lines | Responsibility |
|---------|-------|---------------|
| `heartbeat.ts` | 3,862 | Agent execution orchestration, run lifecycle, concurrency control, session management |
| `company-portability.ts` | 4,247 | Company export/import with full bundle serialization |
| `issues.ts` | ~800 | Issue CRUD, assignment, status transitions, work products |
| `agents.ts` | ~600 | Agent management, hiring, configuration |
| `budgets.ts` | ~500 | Budget policy enforcement, spend tracking |
| `costs.ts` | ~400 | Cost event recording, monthly spend aggregation |
| `workspace-runtime.ts` | ~800 | Execution workspace realization, git operations, runtime services |
| `documents.ts` | ~400 | Document/revision management |
| `secrets.ts` | ~300 | Secret storage with provider abstraction (local encrypted, env-based) |
| `projects.ts` | ~500 | Project CRUD, workspace management |
| Plugin services | ~2,500 | Lifecycle, workers, jobs, events, tools, host services |

### 3.4 Route Layer

Routes in `server/src/routes/` (24 files) handle HTTP concerns — request validation, authorization, response formatting — and delegate to services. Key patterns:

- **Company-scoped routes** use `req.params.companyId` and verify membership
- **Agent-authenticated routes** use JWT tokens issued via `agent-auth-jwt.ts`
- **Board-authenticated routes** use session cookies via BetterAuth or local trust

### 3.5 Authentication & Authorization

Three deployment modes control auth behavior:

| Mode | Auth Mechanism | Use Case |
|------|---------------|----------|
| `local_trusted` | Implicit local board user, loopback only | Solo development |
| `authenticated` | BetterAuth (email/password, OAuth), board claim | Production |
| (agent auth) | JWT tokens per-agent, issued on hire | Agent API calls |

Authorization is layered:
- **Instance roles** — `instance_admin` for cross-company operations
- **Company memberships** — `owner`, `admin`, `member` roles
- **Principal permissions** — Granular permission grants per agent/user
- **Budget enforcement** — Cost limits checked before agent execution

### 3.6 Real-Time Events

`server/src/realtime/live-events-ws.ts` provides a WebSocket server that:
- Authenticates connections using the same actor resolution as HTTP
- Broadcasts domain events (issue updates, agent state changes, run progress)
- Supports company-scoped event filtering

The `publishLiveEvent()` function is called from services to push updates.

### 3.7 Adapter System

Adapters bridge Paperclip to external AI agent runtimes. Each adapter implements:

```typescript
interface ServerAdapter {
  execute(opts: ExecuteOptions): AsyncGenerator<AdapterEvent>;
  test?(opts: TestOptions): Promise<TestResult>;
  sessionCodec?: AdapterSessionCodec;
}
```

**Seven adapters** are currently implemented:

| Adapter | Runtime | Protocol |
|---------|---------|----------|
| `claude-local` | Claude Code CLI | Subprocess + stdout streaming |
| `codex-local` | Codex CLI | Subprocess + stdout streaming |
| `cursor-local` | Cursor CLI | Subprocess + stdout streaming |
| `gemini-local` | Gemini CLI | Subprocess + stdout streaming |
| `opencode-local` | OpenCode CLI | Subprocess + stdout streaming |
| `pi-local` | Pi CLI | Subprocess + stdout streaming |
| `openclaw-gateway` | OpenClaw (remote) | WebSocket + encryption |

All local adapters follow the same execution model: spawn a child process, parse streaming stdout/stderr, extract token usage and cost data, and manage session persistence.

### 3.8 Plugin System

The plugin system allows extending Paperclip with custom functionality:

```
Plugin SDK (npm package)
    │
    ▼
Plugin Worker (isolated child process)
    │ JSON-RPC 2.0
    ▼
Plugin Host Services (server-side)
    │
    ▼
Database, Events, Jobs, Tools, Secrets, etc.
```

Key components:
- **Plugin SDK** (`packages/plugins/sdk/`) — Public API for plugin authors with 20+ context clients
- **Plugin Loader** — Discovers, validates, and activates plugins from local directories or registry
- **Plugin Worker Manager** — Spawns and manages isolated worker processes
- **Plugin Job Scheduler** — Cron-based job scheduling for plugins
- **Plugin Event Bus** — Pub/sub for domain events between plugins and the host
- **Plugin Tool Dispatcher** — Routes tool invocations from agents to plugins

---

## 4. Frontend Architecture (`ui/`)

### 4.1 Technology Stack

- **React 19** with TypeScript
- **React Router** for client-side routing (wrapped in `lib/router.tsx`)
- **TanStack React Query** for server state management
- **Tailwind CSS 4** for styling
- **Radix UI** for accessible primitives
- **Vite 6** for bundling

### 4.2 Application Structure

```
ui/src/
├── main.tsx              # Entry point
├── App.tsx               # Root component, routing, providers
├── api/                  # API client modules (fetch-based)
│   ├── client.ts         # Base HTTP client with error handling
│   ├── agents.ts         # Agent API calls
│   ├── issues.ts         # Issue API calls
│   └── [15+ modules]     # One per domain
├── components/           # Reusable UI components (100+ files)
├── pages/                # Page-level components
├── hooks/                # Custom React hooks
├── context/              # React Context providers
├── lib/                  # Utility libraries
├── adapters/             # UI-side adapter configurations
└── plugins/              # Plugin UI bridge and slots
```

### 4.3 State Management

- **Server state**: TanStack React Query with centralized query keys (`lib/queryKeys.ts`)
- **UI state**: Multiple focused Context providers:
  - `CompanyContext` — Active company selection
  - `DialogContext` — Modal/dialog management
  - `PanelContext` — Side panel state with localStorage persistence
  - `LiveUpdatesProvider` — WebSocket event broadcasting and toast notifications
  - `ToastContext` — Notification queue management

### 4.4 Key UI Features

- **Command Palette** — Global search across issues, agents, projects
- **Kanban Board** — Drag-and-drop issue management
- **Agent Detail** — Run transcripts, configuration, skills, budget display (4,052-line page component)
- **Onboarding Wizard** — Multi-step company/agent setup
- **Markdown Editor** — Rich editing with @-mentions, image upload
- **Org Chart** — SVG-based organizational hierarchy visualization
- **Real-time Updates** — WebSocket-driven live event feed

### 4.5 Adapter UI Layer

Each agent adapter provides UI-side modules:
- `build-config.ts` — Configuration form field definitions
- `parse-stdout.ts` — Real-time output parsing for the run transcript viewer

---

## 5. Database Layer (`packages/db/`)

### 5.1 Technology

- **Drizzle ORM** with PostgreSQL driver
- **Embedded PostgreSQL** for zero-config local development
- **45+ migrations** managed via Drizzle's migration system with custom reconciliation logic

### 5.2 Schema Organization

60+ tables organized by domain:

| Domain | Tables |
|--------|--------|
| **Core** | `companies`, `agents`, `projects`, `issues`, `goals` |
| **Execution** | `agent_task_sessions`, `agent_runtime_state`, `execution_workspaces`, `workspace_operations`, `workspace_runtime_services` |
| **Scheduling** | `heartbeat_runs`, `heartbeat_run_events`, `routines` |
| **Governance** | `approvals`, `approval_comments`, `issue_approvals`, `principal_permission_grants`, `company_memberships` |
| **Finance** | `budget_policies`, `budget_incidents`, `cost_events`, `finance_events` |
| **Content** | `documents`, `document_revisions`, `issue_documents`, `issue_attachments`, `issue_work_products`, `assets` |
| **Plugins** | `plugins`, `plugin_config`, `plugin_jobs`, `plugin_logs`, `plugin_state`, `plugin_webhooks`, `plugin_entities` |
| **Auth** | `auth_users`, `auth_sessions`, `agent_api_keys`, `board_api_keys`, `cli_auth_challenges`, `instance_user_roles` |
| **Settings** | `company_secrets`, `company_secret_versions`, `company_skills`, `instance_settings` |

### 5.3 JSONB Usage

Several tables use JSONB columns for flexible schema:
- `agents.adapterConfig` — Adapter-specific configuration
- `issues.metadata` — Extensible issue metadata
- `plugins.configSchema` — Plugin configuration definitions
- `heartbeat_runs.resultJson` — Run result summaries

---

## 6. CLI Architecture (`cli/`)

The CLI (`paperclipai` npm package) uses **Commander.js** and provides:

- **`paperclip init`** — Interactive onboarding with config file generation
- **`paperclip start`** — Server startup
- **`paperclip client`** — API client subcommands (agents, issues, companies, etc.)
- **`paperclip heartbeat-run`** — Agent execution runner (used by adapters)
- **`paperclip worktree`** — Git worktree management for isolated agent workspaces
- **`paperclip db-backup`** — Manual database backup

The CLI maintains a config file (`.paperclip/config.json`) and supports multi-profile contexts.

---

## 7. Shared Package (`packages/shared/`)

Contains types, validators, and constants shared between server, UI, and CLI:

- **`types/`** — TypeScript interfaces for all domain models (35+ files)
- **`validators/`** — Zod schemas mirroring the type definitions (35+ files)
- **`constants.ts`** — Status enums, role definitions, adapter types, limits
- **`config-schema.ts`** — Configuration file validation with Zod

---

## 8. Key Architectural Patterns

### 8.1 Heartbeat Execution Model

Agents are not long-running processes. Instead, Paperclip uses a **heartbeat-driven execution model**:

1. The heartbeat scheduler ticks on a configurable interval (default 30s)
2. Each tick checks for agents with pending work (assigned issues, wakeup requests)
3. Eligible agents are enqueued for execution with concurrency limits
4. The adapter spawns the agent process, streams output, and records results
5. After execution, the agent process exits; next work happens on a future heartbeat

This model enables cost control (agents only run when there's work), budget enforcement (checked before each run), and graceful recovery (orphaned runs are reaped on startup).

### 8.2 Company Isolation

All data is scoped to companies. The `companyId` column appears on nearly every table. Routes validate company membership before allowing access. This enables multi-tenant operation where multiple agent teams are managed independently.

### 8.3 Portable Company Bundles

Companies can be exported as self-contained JSON bundles containing agents, projects, issues, skills, routines, and configuration. These bundles can be imported to create new companies or shared as templates.

### 8.4 Deployment Modes

| Mode | Binding | Auth | Exposure | Use Case |
|------|---------|------|----------|----------|
| `local_trusted` | Loopback only | Implicit | Private | Solo local development |
| `authenticated` | Any | BetterAuth | Private or Public | Team/production |

### 8.5 Storage Abstraction

File storage uses a provider pattern:
- **`local_disk`** — Files stored in a configurable directory
- **`s3`** — AWS S3 or compatible object storage

---

## 9. Build and Development

### 9.1 Build Pipeline

```
pnpm build
├── packages/shared   → tsc
├── packages/db       → tsc
├── packages/adapter-utils → tsc
├── packages/adapters/*  → tsc (7 adapters)
├── packages/plugins/sdk → tsc
├── server            → esbuild (bundled)
├── ui                → vite build (bundled)
└── cli               → tsc
```

### 9.2 Development Mode

`pnpm dev` runs concurrently:
- Server with `tsx --watch` for hot reload
- UI via Vite dev middleware (embedded in server, single port)

### 9.3 Testing

- **Unit tests**: Vitest for services, utilities, and components
- **E2E tests**: Playwright for full-stack browser testing
- **Smoke tests**: Post-release verification scripts
- **Evaluations**: PromptFoo for prompt quality testing

### 9.4 CI/CD

GitHub Actions workflows:
- `pr.yml` — Type checking, tests, and lint on pull requests
- `release.yml` — Automated release with changelog generation
- `e2e.yml` — End-to-end test suite
- `docker.yml` — Docker image builds and publishing
- `release-smoke.yml` — Post-release verification

---

## 10. Docker Deployment

The Dockerfile produces a single image containing:
- Node.js runtime
- PostgreSQL binaries (for embedded mode)
- Built server, UI, and CLI
- All adapter packages

Docker Compose configurations:
- `docker-compose.yml` — Production setup with external PostgreSQL
- `docker-compose.quickstart.yml` — Zero-config startup with embedded PostgreSQL
- `docker-compose.untrusted-review.yml` — Isolated PR review environment
