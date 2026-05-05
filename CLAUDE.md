# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm install          # install dependencies
pnpm dev              # run CLI agent (tsx src/cli.ts)
pnpm build            # compile TypeScript → dist/
pnpm test             # run all tests (node --import tsx --test test/*.test.ts)
pnpm gateway          # start Gateway WebSocket server
pnpm gateway:connect  # connect to Gateway as client

# Examples
pnpm example:basic
pnpm example:custom-tools
pnpm example:gateway

# Build + publish checks
pnpm pack:check
```

Run a single test file:
```bash
node --import tsx --test test/session.test.ts
```

Requires Node.js ≥20, pnpm. Uses ESM (`"type": "module"`), `NodeNext` module resolution — all imports need explicit `.js` extensions even for `.ts` source files.

## Environment

Copy `.env.example` to `.env` and set at minimum one provider key:

```env
OPENCLAW_MINI_PROVIDER=anthropic
OPENCLAW_MINI_MODEL=claude-sonnet-4-20250514
ANTHROPIC_API_KEY=sk-ant-xxx
```

For providers without extended thinking support (e.g., GLM-4-Flash), set `OPENCLAW_MINI_REASONING=none`.

## Architecture

This is a teaching project — a minimal reproduction of the OpenClaw Agent system. It has four layers, described in order of reading priority:

### Core layer (`src/`)

**`agent-loop.ts` → `agent.ts` → `agent-events.ts`** — the execution spine.

- `agent-loop.ts`: pure function `runAgentLoop()` — dual-loop architecture:
  - **Outer loop**: handles follow-up messages after a turn completes
  - **Inner loop**: LLM call → tool execution (serial) → steering check after each tool
  - Returns an `EventStream<MiniAgentEvent, MiniAgentResult>` synchronously; async IIFE pushes events
- `agent-events.ts`: `MiniAgentEvent` discriminated union (~20 types) + `createMiniAgentStream()` factory
- `agent.ts`: `Agent` class wiring all 5 subsystems; exposes `subscribe(fn)` / `emit(event)` observer pattern, `run(sessionKey, message)`, `abort(runId?)`, `steer(sessionKey, text)`

**Steering**: messages injected mid-run (while tools are executing) go into `steeringQueues`. After each tool call, the loop checks and skips remaining tool calls if there's a pending steering message (using `skipToolCall()` placeholder results).

**`session.ts`**: `SessionManager` — JSONL persistence. Dual-write strategy: in-memory cache + disk append. The first assistant message triggers full file write; subsequent messages O(1) append. A write-lock prevents concurrent corruption.

**`context/`**: three-stage pipeline:
1. `loader.ts` (`ContextLoader`): loads bootstrap files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `MEMORY.md`) from `workspaceDir` into the system prompt
2. `pruning.ts` (`pruneContextMessages`): three-pass progressive pruning — drop old `tool_result` content → drop old `assistant` messages → keep most recent N
3. `compaction.ts` (`compactHistoryIfNeeded`): when context exceeds threshold, summarizes older messages into a single compaction summary message prepended to the pruned history

**`tools/`**: `Tool<TInput>` interface with `name`, `description`, `inputSchema` (JSON Schema), `execute(input, ctx)`. 10 builtin tools in `builtin.ts` (read, write, edit, exec, list, grep, memory_search, memory_get, memory_save, sessions_spawn). All paths validated against `workspaceDir` via `sandbox-paths.ts`.

**`provider/`**: thin wrapper over `@mariozechner/pi-ai` (22+ providers). `errors.ts` handles rate-limit retry detection and context-overflow detection for auto-compaction.

### Extended layer

- `memory.ts` (`MemoryManager`): keyword-based relevance search over persistent memory entries (no vector embeddings — intentional simplification)
- `skills.ts` (`SkillManager`): reads `SKILL.md` files by frontmatter trigger words; rewrites matching user messages to guide the model to load the right skill file
- `heartbeat.ts` (`HeartbeatManager`): two-layer architecture — `HeartbeatWake` (250ms merge window, double-buffer) feeds `HeartbeatRunner` (reads `HEARTBEAT.md`, deduplicates within 24h)

### Gateway layer (`src/gateway/`)

WebSocket RPC server that lets multiple clients share one Agent instance.

- `protocol.ts`: three frame types — `RequestFrame (req)`, `ResponseFrame (res)`, `EventFrame (event)` with monotonic `seq`
- `server.ts`: Challenge-Response handshake → method routing → Pub/Sub broadcast with `dropIfSlow` backpressure + 30s tick heartbeat
- `handlers.ts`: 6 RPC methods (`connect`, `chat.send`, `chat.history`, `sessions.list`, `sessions.clear`, `health`). `chat.send` uses ACK-then-stream: immediate response with `runId`, then async agent events broadcast as `EventFrame`s with 150ms delta throttle
- `client.ts`: Pending Map (UUID→Promise), exponential backoff reconnect (1s→30s), seq gap detection, 2-cycle tick watchdog

### Engineering layer

Safety and concurrency guards that are mostly transparent during development:
- `session-key.ts`: normalizes session keys to `agent:<id>:<session>` format
- `tool-policy.ts`: allow/deny lists for tool filtering before a run
- `command-queue.ts`: session lane (serial per session) + global lane (configurable concurrency across sessions)
- `session-tool-result-guard.ts`: auto-synthesizes missing `tool_result` blocks before LLM calls
- `context-window-guard.ts`: warns/blocks if configured context window is too small
- `tool-approval.ts`: runtime tool approval interceptor (separate from static policy filtering)

## workspace-templates/

Template files for the Agent's workspace context system. These are loaded by `ContextLoader` from `workspaceDir` at runtime:
- `AGENTS.md` — workspace rules loaded into every session's system prompt
- `SOUL.md`, `USER.md`, `IDENTITY.md` — persona/user context
- `HEARTBEAT.md` — active tasks for the heartbeat polling system
- `MEMORY.md` — persistent long-term memory index
- `TOOLS.md` — workspace-specific tool notes

## Key design patterns

| Pattern | Location |
|---------|----------|
| EventStream push/pull (IIFE + `for await`) | `agent-loop.ts` + `agent-events.ts` |
| Subscribe/emit observer | `agent.ts` |
| JSONL append log | `session.ts` |
| Three-stage context pruning + compaction | `context/pruning.ts`, `context/compaction.ts` |
| ACK-then-stream Gateway chat | `gateway/handlers.ts` |
| Timing-safe token comparison | `gateway/handlers.ts` (crypto.timingSafeEqual) |
| Exponential backoff reconnect | `gateway/client.ts` |
