# Paperclip Code Analysis

> Last updated: 2026-03-28

This document catalogs code quality issues, architectural concerns, and improvement opportunities found across the Paperclip codebase.

---

## Table of Contents

1. [Critical: Oversized Files](#1-critical-oversized-files)
2. [High: Adapter Code Duplication](#2-high-adapter-code-duplication)
3. [High: Race Conditions and Concurrency](#3-high-race-conditions-and-concurrency)
4. [High: Security Concerns](#4-high-security-concerns)
5. [Medium: Type Safety Issues](#5-medium-type-safety-issues)
6. [Medium: Error Handling Inconsistencies](#6-medium-error-handling-inconsistencies)
7. [Medium: Dead and Legacy Code](#7-medium-dead-and-legacy-code)
8. [Medium: Performance Concerns](#8-medium-performance-concerns)
9. [Low: Code Smells and Maintenance](#9-low-code-smells-and-maintenance)
10. [Recommendations Summary](#10-recommendations-summary)

---

## 1. Critical: Oversized Files

Several files have grown far beyond maintainable size and should be decomposed.

| File | Lines | Concern |
|------|-------|---------|
| `server/src/services/company-portability.ts` | 4,247 | `exportBundle()` is 576 lines; `importBundle()` is 553 lines; `buildPreview()` is 344 lines |
| `ui/src/pages/AgentDetail.tsx` | 4,052 | Single React component handling runtime state, config editing, run transcripts, skills, budgets |
| `server/src/services/heartbeat.ts` | 3,862 | `executeRun()` is 883 lines — the single largest function in the codebase |
| `server/src/routes/access.ts` | 2,887 | Route handler with deeply nested conditionals spanning 800+ lines |
| `packages/adapters/openclaw-gateway/src/server/execute.ts` | 1,434 | WebSocket protocol, encryption, device identity all in one file |
| `ui/src/components/NewIssueDialog.tsx` | 1,473 | Dialog component doing too much |
| `ui/src/components/AgentConfigForm.tsx` | 1,467 | Complex form with many inline subcomponents |
| `server/src/services/plugin-host-services.ts` | 1,131 | 16 instances of `as any` in a single file |

**Recommendation:** Break each into focused sub-modules. For example, `heartbeat.ts` could become `heartbeat-scheduler.ts`, `heartbeat-executor.ts`, `heartbeat-workspace.ts`, and `heartbeat-recovery.ts`. `AgentDetail.tsx` should be split into `AgentRuntimeView`, `AgentConfigView`, `AgentSkillsView`, and `AgentBudgetView`.

---

## 2. High: Adapter Code Duplication

The 6 local adapters (claude, codex, cursor, gemini, opencode, pi) share ~85% identical code. Total adapter code is ~14,500 lines; an estimated 8,000+ lines are duplicated.

### Duplicated Patterns

| Pattern | Files Affected | Approx Duplicated Lines |
|---------|---------------|------------------------|
| Execution loop (env setup, process spawn, streaming) | All 6 `execute.ts` files | ~400 lines x 6 |
| Skill management (list, sync, symlink) | All 6 `skills.ts` files | ~150 lines x 6 |
| Output parsing (tokens, costs, errors) | All 6 `parse.ts` files | ~120 lines x 6 |
| Environment testing checklist | All 6 `test.ts` files | ~80 lines x 6 |
| CLI event formatting | All 6 `format-event.ts` files | ~60 lines x 6 |
| Claude Code nesting guard env stripping | Multiple files | ~10 lines x 4+ |

### Inconsistencies Between Adapters

- **Session ID naming:** `sessionId` (claude) vs `sessionID` (codex, opencode) — could cause silent session resumption failures
- **Token field names:** `usage.input` vs `inputTokens` vs `usage.inputTokens` — no canonical format
- **Cost tracking:** Some adapters have it, others return 0 — no clear contract
- **Error detection:** Mix of regex patterns, exit codes, and type-based detection

**Recommendation:** Extract a `@paperclipai/adapter-local-base` package containing the shared execution loop, skill sync, env setup, and parsing logic. Each adapter would only implement its unique CLI invocation and output format mapping. This could reduce total adapter code from ~14,500 to ~5,000 lines.

---

## 3. High: Race Conditions and Concurrency

### 3.1 WebSocket LiveEvents Map Access

**File:** `server/src/realtime/live-events-ws.ts`

The `cleanupByClient` and `aliveByClient` Maps are accessed without synchronization. If a socket closes and pings simultaneously, map operations could interleave, causing memory leaks or exceptions.

### 3.2 Agent Start Lock Leaks

**File:** `server/src/services/heartbeat.ts:66`

```typescript
const startLocksByAgent = new Map<string, Promise<void>>();
```

This in-memory Map prevents concurrent runs per agent, but entries are never removed when agents are deleted. Over time, phantom locks accumulate.

### 3.3 Backup Flag Race

**File:** `server/src/index.ts:613`

```typescript
let backupInFlight = false;
```

The check-then-set pattern on this boolean is not atomic. Under high timer frequency, two backup operations could start simultaneously.

### 3.4 Plugin Worker Timer Corruption

**File:** `server/src/services/plugin-worker-manager.ts:197`

Multiple async operations manipulate per-worker timers without synchronization. Timer references could be overwritten before previous timers fire.

### 3.5 Plugin Dev Watcher Debounce

**File:** `server/src/services/plugin-dev-watcher.ts:165`

The `debounceTimers` Map is modified from file watcher callbacks without guards. Rapid file changes could cause events to be processed multiple times or skipped.

### 3.6 Instance Settings Update

**File:** `server/src/routes/instance-settings.ts:36-53`

Updates all companies' activity logs in parallel with `Promise.all()`. If a company is deleted during this operation, partial failures occur with no rollback.

### 3.7 Inconsistent Transaction Usage

Throughout services, some multi-step database operations use `db.transaction()` while others perform sequential queries without transactions. No documented pattern for when transactions are required. Examples:
- Agent creation + permissions setup: no transaction
- Company creation + membership: no transaction
- Issue reassignment + status update: no transaction

---

## 4. High: Security Concerns

### 4.1 No CSRF Tokens for Board Mutations

**File:** `server/src/middleware/board-mutation-guard.ts`

The guard checks Origin/Referer headers but does not use CSRF tokens. These headers can be spoofed in some browser contexts.

### 4.2 No Rate Limiting on Auth Endpoints

No visible rate limiting on `/api/auth/*` routes. Brute force attacks are possible on password endpoints in authenticated mode.

### 4.3 Secrets Strict Mode Bypass

**File:** `server/src/services/secrets.ts:115-118`

Strict mode validates new plain-text secrets but does not retroactively check existing secrets or env bindings from imports.

### 4.4 Weak Session Revocation

Revoked API keys are soft-deleted (`revokedAt` timestamp) rather than hard-deleted. Queries depend on `isNull(agentApiKeys.revokedAt)` — if the index is missing, this becomes a full table scan.

### 4.5 Hardcoded Embedded Postgres Credentials

**File:** `server/src/index.ts:362-365`

```typescript
user: "paperclip",
password: "paperclip",
```

Acceptable for embedded dev mode but could leak to logs if verbose logging is enabled.

---

## 5. Medium: Type Safety Issues

### 5.1 Widespread `as any` Usage

**50+ instances in production code** (not counting tests):

| File | Instances | Context |
|------|-----------|---------|
| `server/src/services/plugin-host-services.ts` | 16 | Service bridge typing |
| `server/src/services/company-portability.ts` | 12 | Import/export serialization |
| `server/src/index.ts` | 9 | Database instance passing |
| `server/src/middleware/logger.ts` | 6 | Log serialization |
| `server/src/services/issues.ts` | 5 | Query result casting |

Test files contain 100+ additional `as any` instances, primarily for mock object construction.

### 5.2 JSONB Columns Without Runtime Validation

Several tables use `jsonb().$type<Record<string, unknown>>()`:
- `agents.adapterConfig`
- `issues.metadata`
- `plugins.configSchema`
- `heartbeat_runs.resultJson`

Data is typed at the Drizzle layer but not validated at runtime. Corrupted JSON could cause runtime crashes.

### 5.3 Loose Adapter Configuration

`Record<string, unknown>` is used for `adapterConfig` throughout the system, preventing compile-time validation of adapter-specific settings.

### 5.4 Plugin Capability Checking is String-Based

Capabilities are checked via `capabilities.includes("events.subscribe")` — plain strings with no type safety. A typo in a capability name would silently fail.

### 5.5 Suspicious Ternary

**File:** `cli/src/client/context.ts:83`

```typescript
const version = record.version === 1 ? 1 : 1;  // Always returns 1
```

This is either dead logic or a bug — both branches return the same value.

---

## 6. Medium: Error Handling Inconsistencies

### 6.1 Silent Background Task Failures

**File:** `server/src/index.ts`

Multiple critical background operations log errors but don't escalate:
- Heartbeat recovery (line 574): logged, not surfaced
- Runtime service reconciliation (line 562): logged, not surfaced
- Database backup failures (line 640): logged, not surfaced
- Plugin tool dispatcher init (line 291): logged, not surfaced

No alerting mechanism exists for any of these.

### 6.2 Mixed Error Patterns

The codebase uses at least 4 different error handling approaches:

```typescript
// Pattern 1: .catch(() => null) — loses error info
await fs.stat(path).catch(() => null);

// Pattern 2: .catch(() => {}) — completely silent
await something().catch(() => {});

// Pattern 3: .catch(err => logger.error(...)) — logged but not propagated
await task().catch(err => logger.error({ err }, "failed"));

// Pattern 4: try/catch with re-throw — proper but inconsistent
try { ... } catch (err) { throw new AppError(..., { cause: err }); }
```

### 6.3 Unvalidated Request Query Parameters

**File:** `server/src/routes/costs.ts:95-113`

`req.query` is typed as `Record<string, unknown>` but accessed as strings via `String()` without validation. Could produce `"undefined"` strings or `NaN` numbers.

### 6.4 Validation Middleware Error Propagation

**File:** `server/src/middleware/validate.ts`

The validation middleware calls `schema.parse()` but relies on Express error handler to catch ZodErrors. Validation failures are not logged with request context.

---

## 7. Medium: Dead and Legacy Code

### 7.1 TODOs Left in Code

- `cli/src/commands/client/company.ts:362` — `TODO: replace this temporary claude_local fallback with adapter selection in the import TUI`
- `ui/src/pages/AgentDetail.tsx:771` — Commented-out skills tab: `// TODO: bring back later`
- `ui/src/adapters/runtime-json-fields.tsx:5` — `TODO(issue-worktree-support): re-enable this UI once the workflow is ready to ship`

### 7.2 Missing Adapter Type

`@paperclipai/shared` constants include `"hermes_local"` in `AGENT_ADAPTER_TYPES`, but no corresponding adapter implementation exists in `packages/adapters/`.

### 7.3 Deprecated Patterns Still Active

- Bootstrap prompt template system marked deprecated but still used in agent instructions
- Old board claim challenge code path still present
- Legacy `pglite` to `embedded-postgres` migration code exists in both `cli/src/config/store.ts` and `packages/db/src/runtime-config.ts` — duplicated migration logic

### 7.4 Type/Validator Dual Maintenance

Every type in `packages/shared/src/types/` (35 files) has a corresponding Zod schema in `packages/shared/src/validators/` (35 files). Changes must be synchronized manually across both locations. Consider using `z.infer<>` to derive types from validators.

---

## 8. Medium: Performance Concerns

### 8.1 N+1 Query in Auth Middleware

**File:** `server/src/middleware/auth.ts:45-60`

Every authenticated request runs two parallel database queries (instance roles + company memberships). This could be cached per session with a short TTL.

### 8.2 Memory-Based Company Filtering

**File:** `server/src/routes/companies.ts:60-68`

Fetches ALL companies from the database, then filters in JavaScript using `Set.has()`. Should use a WHERE clause. Scales poorly with company count.

### 8.3 Unselective Field Loading

Throughout services, `db.select()` loads entire rows including large text/JSONB fields when only IDs or status fields are needed. Particularly impactful for `heartbeat_runs` with full event data.

### 8.4 No Query Result Pagination

Some list endpoints return unbounded results. While `MAX_ISSUE_COMMENT_LIMIT = 500` exists for comments, other queries (agents, projects, activity logs) have no limits.

### 8.5 Command Palette Full Data Fetch

**File:** `ui/src/components/CommandPalette.tsx:60-82`

Fetches ALL issues, agents, and projects when the palette opens. Should implement server-side search with debouncing.

### 8.6 Missing React Memoization

- `PropertyRow` and `PropertyPicker` components re-render on every parent update
- Callback props passed to child components without `useCallback`
- Heavy list components (`IssuesList`, `KanbanBoard`) lack `React.memo` on row components

---

## 9. Low: Code Smells and Maintenance

### 9.1 Duplicated UI Components

`PropertyRow` is defined identically in 4 files:
- `ui/src/components/IssueProperties.tsx`
- `ui/src/components/AgentProperties.tsx`
- `ui/src/components/GoalProperties.tsx`
- `ui/src/components/ProjectProperties.tsx`

~300+ lines of duplicated component code. Should be extracted to a shared `PropertyPanel` component.

### 9.2 server-utils.ts is a Utility Dumping Ground

**File:** `packages/adapter-utils/src/server-utils.ts` (842 lines)

Contains unrelated utilities: process execution, path resolution, JSON parsing, skill management, template rendering, and log redaction. Should be split into focused modules.

### 9.3 Magic Numbers

Hardcoded thresholds scattered without named constants:
- `server/src/app.ts:54` — Vite HMR port offset: `serverPort + 10_000`
- `ui/src/components/Layout.tsx:163-165` — Swipe thresholds: `EDGE_ZONE=30, MIN_DISTANCE=50, MAX_VERTICAL=75`
- `ui/src/components/KanbanBoard.tsx:67` — Column width: `260px`
- Various timeout values: `10_000`, `3000`, `30000` across files

### 9.4 Inconsistent Service Initialization

Services use at least 3 patterns:
```typescript
// Pattern 1: Function returning object
export function heartbeatService(db: Db) { return { ... }; }

// Pattern 2: Singleton with db parameter
export const costService = { recordCost(db, ...) { ... } };

// Pattern 3: Factory with dependencies
export function createPluginJobScheduler(deps: { db, jobStore, workerManager }) { ... }
```

No documented convention for when to use which pattern.

### 9.5 Console Usage in Server Code

**File:** `server/src/app.ts:252`

```typescript
console.warn("[paperclip] UI dist not found; running in API-only mode");
```

Should use the structured `logger` like the rest of the server. There are ~319 `console.*` calls across the codebase — acceptable in CLI/scripts but a few exist in server code.

### 9.6 eslint-disable Comments

12 `eslint-disable` directives found, mostly `react-hooks/exhaustive-deps` in UI components:
- `ui/src/pages/IssueDetail.tsx:593,602`
- `ui/src/pages/GoalDetail.tsx:111`
- `ui/src/pages/CompanyExport.tsx:671,758`
- `ui/src/pages/NewAgent.tsx:117`
- `ui/src/hooks/useDateRange.ts:104`
- `ui/src/components/AgentConfigForm.tsx:217`

These suggest dependency arrays may be incorrect and effects could fire unexpectedly.

### 9.7 Inconsistent Null Handling in UI

Three different patterns used interchangeably:
```typescript
const value = data?.field ?? defaultValue;    // Correct for null/undefined
const label = agent?.name || shortId;          // Catches falsy (empty string, 0)
const shown = x ? agents?.find(...) : null;    // Ternary with nested optional chain
```

### 9.8 No Observability Infrastructure

- No request correlation IDs for tracing
- No Prometheus metrics or performance monitoring
- No visibility into queue depths, connection pool usage, or latencies
- Errors logged without request context, making multi-step flow debugging difficult

---

## 10. Recommendations Summary

### Priority 1 — High Impact, Architecture

1. **Extract adapter base package** — Consolidate ~8,000 duplicated lines across 6 local adapters into `@paperclipai/adapter-local-base`
2. **Decompose oversized files** — Split `heartbeat.ts`, `company-portability.ts`, `AgentDetail.tsx`, and `access.ts` into focused modules
3. **Standardize adapter interfaces** — Define canonical session ID, token usage, and cost structures
4. **Add database transactions** — Wrap multi-step operations (agent creation, company setup, issue reassignment) in transactions

### Priority 2 — Security and Reliability

5. **Add CSRF tokens** for board mutations in authenticated mode
6. **Add rate limiting** on authentication endpoints
7. **Fix race conditions** — Use atomic operations for backup flag, clean up agent start locks, protect WebSocket maps
8. **Add request correlation IDs** for distributed tracing

### Priority 3 — Code Quality

9. **Reduce `as any` usage** — Add proper types to plugin host services and database passing
10. **Derive types from Zod schemas** — Eliminate dual maintenance of types/ and validators/
11. **Extract shared UI components** — PropertyRow, PropertyPicker into reusable modules
12. **Split server-utils.ts** into focused utility modules
13. **Replace magic numbers** with named constants

### Priority 4 — Performance

14. **Cache auth middleware queries** with short TTL
15. **Add server-side search** for command palette
16. **Use selective field loading** in database queries
17. **Add pagination** to unbounded list endpoints
18. **Add `React.memo`** to frequently-rendered list item components
