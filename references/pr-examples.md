# PR Examples — Team Style Reference

---

## PR #7

**Title:** feat(telemetry): complete observability stack — correlation IDs, Langfuse tool tracing, config seeding

## Summary

Production-critical completion of the 3-system observability stack (correlation IDs, Langfuse LLM tracing, Langfuse infrastructure). The original implementation had working foundations but two integration gaps that left agent→backend correlation and tool-level Langfuse tracing inoperative in production. This PR closes both gaps across all 3 Go agents and the Rust backend, and seeds Langfuse configuration defaults for zero-guesswork admin setup.

### Correlation ID chain — agent operations now traced end-to-end

The Go agents had correlation ID generation and HTTP header forwarding code, but every `context.Background()` created bare contexts — the `X-Correlation-ID` header was always empty, making the entire propagation pipeline dead code. All 3 agents now generate fresh UUID v4 correlation IDs at every operation boundary:

- **ADFS Agent**: 11 operation contexts replaced — config fetch, heartbeat, alert checks, rescan triggers, reconnect loops, event flushes, startup checks
- **Scan Agent**: heartbeat ticks, poll/dispatch cycles, and WebSocket connections now carry unique IDs per operation
- **Updater Agent**: heartbeat and WebSocket contexts instrumented
- New `ContextWithTimeout()` / `ContextWithCancel()` helpers in all 3 telemetry packages — clean single-call API instead of verbose 3-line context composition
- 6 new unit tests (2 per agent) verifying UUID generation, deadline propagation, and cancellation

### Langfuse tool call tracing — every AI action now visible

`ToolCallTrace` was built and fully tested in `llm_trace.rs` but never imported or wired into the AI agent loop (`ai_chat.rs`). Langfuse showed LLM calls (prompt/response/tokens) but zero tool execution spans — meaning "which Cypher query ran and did it succeed" was invisible in the tracing dashboard. Now all 10 tool types emit Langfuse spans nested under each session trace:

| Tool | Captured |
|------|----------|
| `execute_cypher` | query input, row count, error status |
| `search_fulltext` | index, term, result preview |
| `web_search` | query, result count |
| `generate_artifact` | type, title, success/failure |
| `generate_report` | cypher, format, row count |
| `generate_pdf_document` | title, format |
| `create_dashboard_app` | title, format |
| `save_ownership_rule` | rule type, name |
| `lookup_feature_catalog` | query |
| `batch_queries` | query count, total rows, error status |

### Langfuse configuration seeding

Added `INSERT OR IGNORE` defaults for `langfuse_enabled`, `langfuse_host`, `langfuse_public_key`, `langfuse_secret_key` in `config_db_schema.sql` — admins can discover required keys without raw SQL guesswork. Idempotent on every startup, disabled by default.

## Changes by layer

### Go Agents (ADFS, Scan, Updater)
| File | Changes |
|------|---------|
| `*/internal/telemetry/telemetry.go` (×3) | `ContextWithTimeout()`, `ContextWithCancel()` helpers |
| `*/internal/telemetry/telemetry_test.go` (×3) | 6 new tests for context helpers |
| `ADFS_Agent/cmd/adfs-agent/main.go` | 11 bare contexts → correlation-ID-bearing contexts, telemetry import |
| `Scan_Agent/cmd/scan-agent/main.go` | Heartbeat/poll/WebSocket contexts instrumented, telemetry import |
| `Updater_Agent/cmd/updater-agent/main.go` | Heartbeat/WebSocket contexts instrumented, telemetry import |

### Backend (Rust)
| File | Changes |
|------|---------|
| `ai_chat.rs` | Import `ToolCallTrace`, wire into all 10 tool call match arms |
| `config_db_schema.sql` | Seed 4 Langfuse config defaults (disabled) |

## Verification

- All 27 Go telemetry tests pass (9 per agent, including 6 new)
- All 3 Go agents compile clean (`go build ./...`)
- Rust backend compiles clean (`cargo check`)
- Zero behavioral changes — all additive instrumentation
- Langfuse remains disabled by default until admin configures credentials

## Merge notes

- **No conflicts expected** — changes are isolated to telemetry infrastructure and the tool-call match block in `ai_chat.rs`
- **No deployment steps required** — Langfuse stays disabled until admin runs `infrastructure/langfuse/setup.ps1` and sets credentials
- **Safe to squash merge** — all changes are additive, no destructive refactors

---

## PR #8

**Title:** feat(ai): agent loop safety rails, orchestrator prompt fix, Langfuse exit tracking

## Summary

Production-critical hardening of the AI agent loop and prompt architecture. Three interconnected improvements that make the agent safe, observable, and correctly configured.

### 1. Orchestrator Prompt — Single Source of Truth

The orchestrator prompt (the short behavioral directive at the end of the system prompt) was defined in **3 places** — i18n hardcode, config DB, and prompts DB — but the chat UI only used the i18n one, making the admin config page's setting **completely inert**. The per-message textarea below the chat input kept resetting on every new tab and could never be permanently cleared (backend had a hardcoded fallback).

**Fix:** Removed the chat-level textarea entirely. The backend now reads the orchestrator prompt server-side with a clean priority chain: per-request override → admin config DB (`ai_orchestrator_prompt`) → seeded prompts DB default. The admin config page is now the single source of truth. Also fixed the format suffix (HTML/PDF/web-search instructions) being duplicated in both the user message AND the orchestrator prompt — it now only appears in the user message.

### 2. Agent Loop Safety Rails — 7 Systems

The agent loop (`run_agent_loop` in `ai_chat.rs`) had no context management, no budget enforcement, no timeout, and no circuit breaker. Complex queries could run for 50+ minutes with unbounded token consumption.

| Safety System | What It Does |
|---|---|
| **Reasoning block sanitization** | Strips `reasoning_content` from `choice` before it enters message history (o3/o4/gpt-5.x thinking blocks were silently inflating context) |
| **Wall-clock timeout** | Kills the loop after 10 min (configurable via `ai_agent_timeout_secs`) |
| **Token budget** | Stops when total tokens exceed limit (configurable via `ai_max_tokens_per_run`, default unlimited) |
| **Wrap-up warning** | At 80% of any limit, injects a "wrap it up" system message so the LLM produces a complete answer instead of getting cut off |
| **Per-tool call limits** | Caps each tool (e.g. `execute_cypher` max 30, `generate_artifact` max 3) — pushes error result and skips execution when exceeded |
| **Circuit breaker** | After 5 consecutive tool failures (configurable), injects "use existing data" message instead of hammering a dead service |
| **Context compaction** | New `compact_assistant_history()` truncates older assistant messages (head 300 + tail 150 chars) while preserving `tool_calls` for API pairing |

All systems are purely additive — the agent always works, tools/prompts/SSE contract unchanged. The only difference is optimization and safety.

### 3. Langfuse Exit Tracking

The Langfuse trace output was missing the `status_label`, so the dashboard couldn't distinguish why an agent run ended. Fixed:
- Added `status_label` and `exit_reason` to trace output JSON
- Added `agent_completion_quality` score: 1.0 for normal completion, 0.5 for safety-rail exits (timeout, token budget, max iterations)
- Bug fix: `choice` variable needed `mut` for reasoning block sanitization to compile

## Changes by file

| File | Changes |
|---|---|
| `backend/src/api/ai_chat.rs` | +338 lines: all 7 safety systems, reasoning sanitization, Langfuse exit tracking, orchestrator prompt resolution chain, `compact_assistant_history()` |
| `backend/src/api/ai_config.rs` | +42 lines: 3 new config fields (struct, default, load, save) |
| `frontend/src/pages/AiConfigPage.tsx` | +109 lines: 3 new admin UI controls (timeout, token budget, error limit) |
| `frontend/src/pages/AiAssistantPage.tsx` | -35/+12 lines: removed orchestrator textarea, added warning step rendering (amber AlertTriangle) |
| `frontend/src/lib/api.ts` | +8 lines: 3 new config fields in TypeScript interface, orchestratorPrompt made optional |

## New Admin Config Controls

Under "Agent-Limits" section in AI Config page:
- **Agent-Timeout (Sekunden)**: Default 600s (10 min), range 30-3600
- **Max Tokens pro Lauf**: Default unlimited, range 50K-2M
- **Fehler-Abbruchgrenze**: Default 5, range 2-20

## Verification

- `cargo check` — clean
- `npx tsc --noEmit` — clean
- All new fields backward compatible (`#[serde(default)]`, all default to 0 = "use code default")
- SSE contract unchanged — existing frontends work without update
- Langfuse remains disabled by default until admin configures credentials

## Commits

```
bf1f3ca fix(ai): fix reasoning sanitization mutability and add Langfuse exit tracking
b1405b6 feat(ai): add 7 agent loop safety rails
b71738c fix(ai): single source of truth for orchestrator prompt
```

**Safe to squash merge.**

---

## PR #10

**Title:** feat(ai): web search config fix, test infrastructure sweep, Langfuse tracing upgrade

## Summary

Three layers of production hardening: a user-facing web search config bug that silently hid features, a test infrastructure sweep that resolves all 11 failing tests, and a Langfuse tracing upgrade that adds full API field coverage with observation nesting.

### 1. Web Search Config — is_enabled vs is_ready

Selecting "Bing Search API" as the web search provider caused the web-search toggle to **silently vanish** from the chat UI. Root cause: `WebSearchConfig::is_enabled()` conflated admin intent (show the feature) with provider readiness (API key configured). When the Bing API key was missing, `is_enabled()` returned false, hiding the entire feature instead of showing a "not configured" warning.

**Fix:** Split into a 3-method contract:

| Method | Purpose | Consumers |
|---|---|---|
| `is_enabled()` | Admin turned it on + valid provider selected | Frontend UI visibility |
| `is_ready()` | Enabled AND provider prerequisites met (API key, etc.) | Runtime tool injection in `ai_chat.rs` |
| `config_error()` | User-facing diagnostic when enabled but not ready | Frontend warning banner |

The frontend now shows the web-search option with an **amber warning state** when enabled but not ready, with inline config error text and provider-specific setup hints (e.g. Credential Manager key name for Bing Search API). `bulk.rs` sends `aiWebSearchAvailable`, `aiWebSearchReady`, and `aiWebSearchConfigError` to the frontend.

### 2. Test Infrastructure — All 11 Failures Resolved

The test suite had accumulated 11 failures across 8 files from API signature changes, role model updates, and missing service registrations.

**API contract drift (6 files):**
- `contract_tests.rs`: enrollment token API changed to `consume_enrollment_token(id, agent_id, ip)` — test still used old signature
- `credentials.rs`, `workers.rs`, `system_architecture.rs`: `Role::Viewer` removed/renamed to `Role::Requester` — 7 permission-denied tests referenced a non-existent role
- `deep_report.rs`, `fs_cleanup.rs`: `{guid}` in format strings triggered compiler warnings (needed `{{guid}}`)

**Infrastructure gaps (8 files):**
- `TestApp` missing `WorkerRegistry` and `WsConnectionManager` — handlers crashed before permission checks could run (4 tests)
- `global_search`, `temporal`, `system_architecture`: required `Neo4jPool` even for permission-check tests — changed to `Option<Neo4jPool>` (5 tests)
- `provisioning`: test referenced non-existent `/templates/user` route
- `credentials`: test was environment-specific (domain-joined machines returned different credential names)
- `scheduler`: cron day-of-week convention wrong (1=Sun not 1=Mon)
- `config_db`: fresh-table assertion failed because AI safety rail defaults now seed rows at startup

### 3. Safety Rail Test Coverage

Unit tests for the compaction and config systems added in PR #8:

| Test area | Coverage |
|---|---|
| `compact_tool_history` | Noop within budget, truncates old results, skips small results (< 600 chars), preserves recent results |
| `compact_assistant_history` | Noop with few messages, truncates old assistant content, preserves `tool_calls` intact for API pairing |
| `ai_config` safety rails | Default values (0 = use code default), custom value loading, invalid value fallback to 0, JSON serde roundtrip (frontend ↔ backend contract) |
| `web_search` | `is_enabled` (UI visibility), `is_ready` (runtime gating), `config_error` (diagnostics), frontend warning state |

### 4. Langfuse Tracing Upgrade — Full API Coverage + Nesting

The Langfuse client was sending minimal trace/generation/span events, missing most of the API's fields. Upgraded to full coverage:

**New fields:** `tags`, `release`, `version`, `public`, `environment` on traces. `parent_observation_id`, `prompt_name`, `prompt_version` on generations and spans. Cost override fields on usage. New `EventEvent` struct for discrete point-in-time observations.

**Observation nesting:** Tool call spans are now nested under their parent LLM generation via `parent_observation_id` — the Langfuse dashboard shows the full call tree instead of flat spans.

**Wiring in `ai_chat.rs`:** Traces tagged with context/mode, release from build consts, environment from `langfuse_environment` config key. LLM messages passed as generation input for full prompt visibility in the dashboard.

## Changes by file

| Layer | File | Changes |
|---|---|---|
| Backend | `web_search.rs` | +162: `is_enabled`/`is_ready`/`config_error` 3-method split |
| Backend | `ai_chat.rs` | +273: web search `is_ready()` gating, safety rail tests, Langfuse nesting |
| Backend | `ai_config.rs` | +62: safety rail config test coverage |
| Backend | `bulk.rs` | +4: send `aiWebSearchReady` + `aiWebSearchConfigError` to frontend |
| Backend | `telemetry/langfuse.rs` | +383: full API field coverage, `EventEvent`, cost overrides |
| Backend | `telemetry/llm_trace.rs` | +357: builder upgrades, parent nesting, environment tagging |
| Backend | `main.rs` | +7: build release const for Langfuse trace tagging |
| Backend | `config_db_schema.sql` | +1: `langfuse_environment` seed |
| Backend | `test_helpers.rs` | +13: `WorkerRegistry` + `WsConnectionManager` in `TestApp` |
| Backend | 6 test fix files | +21/−19: role rename, API signature alignment, format string escaping |
| Backend | 4 infra fix files | +80/−27: `Option<Neo4jPool>`, cron fix, credential env-agnostic, config_db seed |
| Frontend | `OutputFormatPanel.tsx` | +69: amber warning state for unconfigured web search |
| Frontend | `WebSearchPopup.tsx` | +28: amber globe icon + diagnostic banner |
| Frontend | `OutputFormatPanel.test.tsx` | +32: warning state + clean state test cases |
| Frontend | `api.ts` | +6: `aiWebSearchReady`, `aiWebSearchConfigError` types |
| Frontend | `AiAssistantPage.tsx` | +5: pass new web search state props |
| Frontend | `AiConfigPage.tsx` | +21: provider-specific setup hints |

## Commits

```
b9e07d6 fix(ai): split WebSearchConfig::is_enabled into is_enabled/is_ready/config_error
caa8ec1 feat(ui): show web-search with warning state when provider not ready
5eeac14 test(ai): update web search tests for is_enabled/is_ready contract
2b9f032 fix(tests): align test assertions with updated API signatures and role model
abef0a8 test(ai): add unit tests for agent loop safety rails and config fields
6fb2620 fix(test): resolve all 11 test failures with infrastructure fixes
036e39b feat(telemetry): upgrade Langfuse tracing with full API fields, nesting, and event support
```

## Verification

- All 11 previously failing tests now pass
- Safety rail compaction + config roundtrip tests pass
- Web search warning state renders correctly in OutputFormatPanel and WebSearchPopup
- Langfuse events include full field set when tracing is enabled
- All new fields backward compatible (`#[serde(default)]`)
- Zero changes to SSE contract, tool definitions, or prompt system

**27 files changed, +1441, −116. Safe to squash merge.**

