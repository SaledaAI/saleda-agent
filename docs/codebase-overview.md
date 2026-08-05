# Hermes Agent Codebase Overview

> Orientation snapshot for maintainers and coding agents. Verified against
> `main` at commit `aec331899` on 2026-08-04. The filesystem and source remain
> authoritative when this document and the implementation differ.

## Executive summary

Hermes is a Python agent runtime presented through several clients:

- classic interactive CLI;
- React/Ink terminal UI;
- messaging gateway and platform adapters;
- browser dashboard with an embedded TUI;
- Electron desktop application with its own chat UI;
- ACP integrations and batch/research entry points.

The central architectural idea is a narrow agent core surrounded by tools,
skills, plugins, providers, platform adapters, and UI clients. A capability
should normally be added at one of those edges instead of becoming a new tool
included in every model request.

Two invariants dominate design and review:

1. A conversation's cached prompt prefix must stay stable. Do not rebuild the
   system prompt or mutate earlier context during a conversation except through
   the supported compression path.
2. Message roles must preserve valid alternation and tool-call structure. Do
   not inject synthetic user messages into the middle of an agent turn.

Read `AGENTS.md` before changing code. It contains the contribution policy,
footprint ladder, configuration rules, and surface-specific guidance.

## Runtime map

```text
                         User-facing surfaces

  classic CLI     Ink TUI     messaging     dashboard     desktop     ACP
      |              |           |              |             |         |
      |              +----- stdio JSON-RPC -----+             |         |
      |                          |               |             |         |
      |                    tui_gateway           +-- PTY/TUI   |         |
      |                          |               +-- WS RPC ---+         |
      +--------------------------+-----------------------------+---------+
                                 |
                              AIAgent
                                 |
                       agent conversation loop
                                 |
          providers -- tool registry -- sessions -- memory -- plugins
```

The diagram is conceptual: some surfaces call the core directly, while others
cross a JSON-RPC, WebSocket, subprocess, or PTY boundary.

## Primary entry points

| Surface | Entry point | Notes |
| --- | --- | --- |
| Main CLI | `hermes_cli.main:main` | Installed as `hermes`; argparse dispatches chat, gateway, setup, dashboard, tools, profiles, and other subcommands. |
| Direct agent | `run_agent:main` | Installed as `hermes-agent`; useful for programmatic and legacy direct runs. |
| ACP | `acp_adapter.entry:main` | Installed as `hermes-acp`. |
| Agent object | `run_agent.AIAgent` | Public programmatic interface. `chat()` wraps `run_conversation()`. |
| Conversation implementation | `agent.conversation_loop.run_conversation` | Executes a turn: preparation, model calls, tool dispatch, fallback/retry, compression, persistence, and post-turn work. |
| Messaging gateway | `gateway/run.py` | Orchestrates adapters, routing, active turns, delivery, recovery, and slash commands. |
| TUI/desktop backend | `tui_gateway/server.py` | JSON-RPC method router, session lifecycle, streaming events, prompts, commands, and agent ownership. |
| Dashboard/backend server | `hermes_cli/web_server.py` | FastAPI REST/WebSocket service, dashboard SPA, PTY bridge, and headless `serve` mode. |
| Ink client | `ui-tui/src/entry.tsx` | Terminal UI communicating with `tui_gateway`. |
| Desktop client | `apps/desktop/` | Electron + React + nanostores; independent chat surface using the JSON-RPC backend. |
| Web dashboard | `web/` | React dashboard. The primary chat transcript/composer is the embedded Ink TUI over PTY. |

Python console scripts are declared in `pyproject.toml` under
`[project.scripts]`. JavaScript packages are npm workspaces declared in the
root `package.json`.

## Agent turn lifecycle

`run_agent.AIAgent` remains the compatibility-facing class, but important
behavior has been extracted into `agent/` modules.

Approximate flow for a normal turn:

1. A surface resolves configuration, profile, provider, model, toolsets,
   session identity, callbacks, and credentials.
2. `AIAgent.run_conversation()` establishes per-turn accounting,
   observability, relay coordination, and context variables.
3. `agent.conversation_loop.run_conversation()` builds the stable request
   context and checks interruption and iteration budgets.
4. The selected provider adapter performs a model request.
5. Assistant tool calls are dispatched through
   `model_tools.handle_function_call()` and the central tool registry.
6. Tool results are appended with valid tool-call structure, then the model is
   called again.
7. Context pressure may invoke the supported compression path and rotate the
   persisted session segment without changing the logical conversation.
8. A terminal assistant answer is streamed/persisted, followed by accounting,
   lifecycle hooks, and optional memory/skill review work.

`IterationBudget` is shared with delegated subagents. Do not introduce a child
execution path that silently creates an unrelated unlimited budget.

Provider-specific protocol handling belongs in `agent/` provider and transport
modules rather than in UI surfaces.

## Tools and toolsets

### Registration

`tools/registry.py` is the dependency leaf for tools. Tool implementation files
call `registry.register(...)` at module scope. `discover_builtin_tools()` scans
top-level Python syntax before importing only self-registering modules.

`model_tools.py` triggers built-in discovery and exposes compatibility maps and
the main dispatcher. MCP discovery is intentionally performed by each entry
point instead of at `model_tools` import time, because blocking MCP startup on
an active async gateway loop can freeze platform heartbeats.

When adding capability, prefer the footprint ladder in `AGENTS.md`:

1. extend existing code;
2. CLI command plus skill;
3. service-gated tool;
4. standalone plugin;
5. MCP server;
6. new always-present core tool only as a last resort.

### Availability and schema size

`toolsets.py` defines category toolsets and platform bundles. A registry entry
may have a `check_fn`, so a configured service can contribute a structured tool
without adding schema weight for users who do not have that service.

Availability checks are cached briefly and retain a short last-known-good
window to avoid a transient external probe removing tools mid-session. Changes
here affect prompt schema stability and must be tested across profiles and
long-lived sessions.

Webhook-originated content receives an intentionally restricted safe toolset.
Do not broaden it casually: public webhook text is an untrusted prompt-injection
source.

### Plugins

Plugins are discovered from bundled, user, project, or installed-package
locations. They should integrate through generic registries, ABCs, and hooks.
A plugin must not special-case itself by modifying core files.

Third-party product integrations generally belong in standalone plugin
repositories, not under this repository's `plugins/` tree.

## Commands

`hermes_cli/commands.py` is the source of truth for interactive slash-command
metadata. `COMMAND_REGISTRY` feeds:

- classic CLI dispatch and help;
- aliases and autocomplete;
- gateway-known commands and help;
- Telegram command menus;
- Slack subcommand routing.

Add aliases to the existing `CommandDef`; do not duplicate alias handling in
individual clients.

Desktop intentionally curates built-in commands in
`apps/desktop/src/lib/desktop-slash-commands.ts`. That curation must continue to
allow non-built-in extension commands, including skills and user quick
commands, through both discovery and execution paths.

## Sessions, profiles, and persistence

`hermes_state.py` and related `hermes_state_*` modules own the SQLite session
database. Conversation search uses FTS5. The gateway adds routing and active
session state through `gateway/session.py`.

Important distinctions:

- A session segment is not always the same as a logical conversation. Context
  compression can rotate the current segment while preserving lineage.
- Gateway routing keys incorporate platform/chat/thread/profile context and
  must be sanitized before they touch paths or persistent indexes.
- SQLite is the durable source of truth for session history. Some gateway
  indexes and compatibility mirrors also exist and need crash-healing and stale
  entry pruning.
- Profiles are independent configuration islands. Do not introduce live
  inheritance from the default profile; cloning at creation is the supported
  sharing mechanism.
- `HERMES_HOME` resolution is profile-aware. Thread and async boundaries must
  propagate the active profile context explicitly.

Changes to session rotation, interruption, restart recovery, or profile routing
need real-path tests with a temporary Hermes home, not only mocks.

## Client surfaces

### Classic CLI

`hermes_cli/main.py` owns top-level argparse setup and command dispatch.
`cli.py` owns the long-lived interactive CLI orchestration and slash-command
handling. Rich handles formatted output and prompt_toolkit handles interactive
input.

### Ink TUI

`ui-tui/` is a React/Ink terminal application. It communicates with the Python
`tui_gateway` through newline-delimited JSON-RPC over stdio. Python owns model
calls, sessions, tools, and persistence; TypeScript owns rendering and input.

### Dashboard

`web/` supplies the dashboard shell and supporting structured panels. Its main
chat is the actual Ink TUI hosted through the PTY WebSocket in
`hermes_cli/web_server.py`. Extend Ink when changing the primary dashboard chat
experience; do not rebuild the transcript or composer in React.

### Desktop

`apps/desktop/` is a genuinely separate chat surface. Electron starts a
headless `hermes serve` backend and the renderer communicates over JSON-RPC.
The desktop does not depend on the dashboard frontend at build or runtime.

`apps/shared/` contains the framework-independent JSON-RPC client, WebSocket
URL helpers, and shared skin/transport definitions used by multiple clients.

## Messaging gateway

`gateway/run.py` is the main gateway orchestrator. `gateway/platforms/base.py`
defines the common platform contract. A smaller set of fundamental adapters is
built in; many platform integrations are supplied as plugins under
`plugins/platforms/`.

The gateway handles several sensitive boundaries simultaneously:

- sender authorization and pairing;
- routing to a stable session;
- concurrent-message interruption or queueing;
- platform-specific formatting and media;
- restart recovery and auto-continuation;
- tool progress and final delivery;
- untrusted platform metadata included in prompts.

Preserve neutralization, ID hashing, path safety, and platform-specific PII
rules in `gateway/session.py` when changing prompt context.

## Configuration and dependencies

User configuration lives in `~/.hermes/config.yaml`. `.env` is reserved for
credentials such as API keys and tokens. Do not add a user-facing `HERMES_*`
environment variable for behavioral configuration; expose it through the setup
and `hermes tools`/`hermes config` workflows instead.

Python supports 3.11 through 3.13. Core dependencies are exact-pinned in
`pyproject.toml`; provider-specific and heavyweight dependencies belong in
optional extras and are lazy-installed where appropriate. Update `uv.lock`
when changing a pin.

The JavaScript side is an npm workspace using Node 22+, React, TypeScript,
Vite, Vitest, and Electron/Ink depending on the package.

## Testing and validation

Use the repository harness for Python tests:

```bash
scripts/run_tests.sh tests/path/to/test_file.py -q
```

It selects an appropriate virtual environment and normally isolates test files
in separate Python processes to contain mutable module-level state.

Common checks:

```bash
# Focused Python behavior
scripts/run_tests.sh tests/test_toolsets.py -q

# Ink TUI
npm run check --workspace ui-tui

# Dashboard
npm run check --workspace web

# Desktop unit/type/lint checks
npm run typecheck --workspace apps/desktop
npm run lint --workspace apps/desktop
npm run test --workspace apps/desktop

# Desktop slash-command contract
cd apps/desktop
npx vitest run src/lib/desktop-slash-commands.test.ts
```

The Python suite excludes tests marked `integration` by default. Resolution
chains, config propagation, persistence, security boundaries, and remote I/O
need explicit E2E coverage in addition to unit tests.

At the time of this snapshot, a representative registry/toolset slice passed:
45 tests in `tests/test_toolsets.py`, `tests/test_toolset_distributions.py`,
`tests/providers/test_provider_registry.py`, and
`tests/gateway/test_multiplex_adapter_registry.py`.

## Complexity and risk map

The architecture is sound, but several orchestration modules remain large:

| Area | Primary risk |
| --- | --- |
| `gateway/run.py` | Platform routing, concurrency, delivery, and restart behavior interact in one large orchestrator. |
| `cli.py` | Long-lived interactive state and compatibility behavior have a wide regression radius. |
| `hermes_cli/web_server.py` | REST, WebSockets, PTY management, dashboard APIs, authentication, and headless serving share one module. |
| `tui_gateway/server.py` | JSON-RPC, sessions, commands, streaming, prompts, and lifecycle management are tightly coupled. |
| `agent/conversation_loop.py` | Retry, fallback, compression, tool calling, persistence, and cancellation meet in the hottest runtime path. |
| sync/async bridges | Persistent event loops, worker threads, subprocesses, PTYs, and cancellation can leak or deadlock if ownership is unclear. |
| protocol shapes | Python emits JSON-RPC payloads consumed by TypeScript without one generated cross-language schema. |
| mirrored state | SQLite plus routing/client compatibility state requires careful crash recovery and coherence tests. |

Large-file extraction is welcome when it is a declared, behavior-preserving
refactor. Preserve public imports and monkeypatch seams where tests or plugins
depend on them.

## Practical change checklist

Before implementing a change:

1. Reproduce the reported behavior on current `main`.
2. Trace the exact runtime path and inspect relevant history with
   `git log -p -S '<symbol>'` when an omission or restriction may be deliberate.
3. Choose the smallest footprint that solves the problem.
4. Identify all sibling surfaces that share the same command, session,
   provider, or transport contract.
5. Preserve prompt-cache stability, role alternation, tool-call structure,
   profile isolation, and credential boundaries.
6. Add behavior/invariant tests rather than snapshots of changing lists or
   version literals.
7. Exercise the real integration path with a temporary `HERMES_HOME` when the
   change crosses configuration, persistence, security, or I/O boundaries.
8. Run the smallest relevant checks first, then broaden validation in
   proportion to the regression risk.

## Fast orientation reading order

For a future task, read only the relevant slice after `AGENTS.md`:

- Agent behavior: `run_agent.py`, then `agent/conversation_loop.py` and the
  named helper module.
- Tools: `tools/registry.py`, `model_tools.py`, `toolsets.py`, then the tool
  implementation.
- CLI command: `hermes_cli/commands.py`, `hermes_cli/main.py`, and `cli.py`.
- Messaging: `gateway/session.py`, `gateway/platforms/base.py`, the adapter,
  then the relevant portion of `gateway/run.py`.
- TUI or desktop backend: `tui_gateway/server.py`, then the client call site.
- Dashboard chat: `ui-tui/` plus the PTY/sidecar sections of
  `hermes_cli/web_server.py`.
- Desktop UI: `apps/desktop/AGENTS.md`, then the feature's atoms, actions, and
  components.
- Persistence: `hermes_state.py` and the focused `hermes_state_*` module.
- Provider: `providers/base.py`, the model-provider plugin, and the relevant
  request adapter under `agent/`.

