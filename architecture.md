# DeepSeek Harness — Complete Architecture Reference

> **Purpose**: This document is a self-contained, exhaustive architectural reference for the DeepSeek Harness (DSH) project. It is written so that an AI agent with zero prior knowledge of the project could understand every structural decision and rebuild the exact same system from scratch.

---

## 1. What Is DeepSeek Harness?

DeepSeek Harness is an **open-source Web GUI and agent backend** developed by DeepSeek AI. It provides an interactive browser-based interface for conversing with AI agents, backed by a Node.js server runtime. The agent can execute code, search the web, read and write files, manage tasks, delegate to sub-agents, and interact with the filesystem—all through a unified plugin architecture.

**Key facts**:
- **License**: MIT
- **Version**: 0.1.2-alpha.3 (developer preview; breaking changes are expected)
- **Repository**: `https://github.com/deepseek-ai/deepseek-harness`
- **Documentation site**: `https://deepseek-harness.github.io/deepseek-harness/`

**Safety**: The project is experimental. It can execute model-generated code and commands, load third-party plugins, and access the network, processes, credentials, and files. Sandboxing, approval prompts, and permission controls reduce risk but do not guarantee isolation. It must not be treated as production-ready or secure.

---

## 2. Technology Stack

### Languages and Runtimes
| Technology | Version | Role |
|---|---|---|
| **TypeScript** (strict, ESM everywhere) | ^6.0.3 | Primary language for all packages, apps, and scripts |
| **JavaScript (ESM)** | — | Vendored framework, scripts, tests |
| **C11** (statically linked against musl) | — | `landlock-run` Linux sandbox launcher (native binary) |
| **React** | ^18.2.0 | Browser GUI framework |
| **HTML / CSS** | — | Browser shell and UI styling (CSS Modules + clsx) |
| **YAML** | — | Cordis configuration (`cordis.yml`, `cordis.patch.yml`), i18n translation files |
| **Python** (optional) | — | SDK runtime (referenced in `.gitignore` and `pytest.ini` but not present in checkout) |

### Build Toolchain
| Tool | Version | Role |
|---|---|---|
| **Node.js** | ^22.19.0 or >=24.0.0 | Runtime (CI covers 22.19, 24, and 26) |
| **pnpm** | 11.7.0 (pinned via `packageManager`) | Package manager and workspace orchestrator |
| **Corepack** | (built into Node) | Enables pinned pnpm |
| **TypeScript** | ^6.0.3 | Compiler (`tsc -b` for composite projects) |
| **tsdown** | ^0.22.2 | Runtime bundle bundler (ESM, workspace-wide, with Typert plugin) |
| **Vite** | ^6.0.0 | Browser application bundler (React + Vite for `apps/web`) |
| **tsx** | ^4.22.4 | TypeScript execution for scripts (`node --import tsx/esm`) |
| **LightningCSS** | ^1.32.0 | CSS bundling/optimization |
| **oxlint** | 1.76.0 | Linter (replaces ESLint for most checks) |
| **vitest** | ^4.1.8 | Test runner (unit, e2e, snapshot, web, coverage, perf, stress) |
| **lefthook** | ^2.1.9 | Git hooks (pre-commit, pre-merge-commit, pre-push) |
| **jscpd** | ^5.0.12 | Cross-file TypeScript clone detection |
| **publint** | ^0.3.21 | Package publication correctness |
| **Playwright** | ^1.49.0 | Browser testing (Chromium) |
| **mermaid** | 11.16.0 | Diagram rendering in docs |

### Framework (Vendored)
The Cordis framework and its foundation libraries are **source-vendored** (not npm-installed) under `vendor/`. They are renamed into the `@deepseek-ai` scope to prevent registry squatting. The project is described in the paper [_A Programming Paradigm for Spatiotemporal Composability_](https://arxiv.org/abs/2608.25512).

| Vendored Package | npm Name | Upstream Name | Version |
|---|---|---|---|
| `vendor/cordis/` | `@deepseek-ai/cordis` | `cordis` | 4.0.0-rc.7 |
| `vendor/cosmokit/` | `@deepseek-ai/cosmokit` | `cosmokit` | 1.8.1 |
| `vendor/schemastery/` | `@deepseek-ai/schemastery` | `schemastery` | 3.18.0 |
| `vendor/loader/` | `@deepseek-ai/cordis-plugin-loader` | `@cordisjs/plugin-loader` | 1.0.0-rc.5 |
| `vendor/include/` | `@deepseek-ai/cordis-plugin-include` | `@cordisjs/plugin-include` | 1.0.4 |
| `vendor/group/` | `@deepseek-ai/cordis-plugin-group` | `@cordisjs/plugin-group` | 1.0.0 |
| `vendor/timer/` | `@deepseek-ai/cordis-plugin-timer` | `@cordisjs/plugin-timer` | 1.1.2 |
| `vendor/hmr/` | `@deepseek-ai/cordis-plugin-hmr` | `@cordisjs/plugin-hmr` | 1.0.15 |
| `vendor/logger-console/` | `@deepseek-ai/cordis-plugin-logger-console` | `@cordisjs/plugin-logger-console` | 1.0.0 |

The vendored packages carry **19 documented local modifications** (lifecycle hardening, JSDoc enrichment, transactional config reconciliation, lazy Loader config resolution, etc.) tracked exhaustively in `vendor/README.md`. Upstream sync follows a documented procedure there.

---

## 3. What Is Cordis?

Cordis is an **all-plugin runtime framework**. Every part of the DSH product is a Cordis plugin—including the model adapter, tool registry, session log, agent loop, browser UI, and HTTP server. Key Cordis concepts:

- **Context** (`ctx`): A shared mutable registry. Plugins contribute services, typed events, and reversible effects through `ctx`. Every service has a string key (e.g., `ctx.sessions`, `ctx.tools`, `ctx.llm`).
- **Plugin**: A unit that registers services and effects on a Context. Plugins can be loaded/unloaded at runtime; their registrations are **effects** that unwind on disposal.
- **Fiber**: The lifecycle unit of a plugin. Fibers have states (PENDING → LOADING → ACTIVE → UNLOADING), manage disposal, and support HMR (hot module replacement).
- **Loader**: The vendored Cordis Loader resolves plugin entries from `cordis.yml` configuration, handles dependencies, and manages plugin lifecycle.
- **Events**: Typed, extensible event maps. Plugins use `ctx.on()` to subscribe. Events use **waterfall** semantics (listeners must call `next()` to delegate) or **serial** semantics.
- **Service**: A singleton object registered on `ctx` under a string key. Services are resolved via `ctx.get(name)` or the typed `ctx.<name>` proxy.
- **`!!js` expressions**: YAML `!!js` tags in `cordis.yml` config allow runtime expressions that reference `ctx` and `process.env` at boot time.

---

## 4. Repository Layout

```
deepseek-harness/
├── .agents/                  # Agent workflows, skills, and decision notes
│   ├── notes/                # Architecture, process, and feature decision records
│   │   ├── implemented/      # Active decision records (present-tense authority)
│   │   ├── archived/         # Frozen historical decisions (read-only)
│   │   └── proposed/         # Decisions under discussion
│   └── skills/               # Reusable agent task instructions (dsh-doc, dsh-prose-standard, etc.)
│
├── .github/                  # GitHub Actions, issue templates, issue lifecycle policy
│   ├── workflows/
│   │   ├── ci.yml            # Pull-request CI (9 jobs: Linux static/coverage/consumers, Windows build/coverage/native/observational, Wine, node-compat)
│   │   ├── ci-master.yml     # Master-push CI (serial Windows/Linux standbys, wine-apt-cache seeder, benchmarks)
│   │   ├── release.yml       # DSH release workflow
│   │   ├── release-vendor.yml / release-vendor-publish.yml  # Vendor release
│   │   ├── landlock-run.yml / landlock-run-release.yml      # Native binary build+release
│   │   ├── e2e.yml           # Real-API end-to-end tests
│   │   ├── docs-pages.yml    # Documentation site deployment
│   │   └── (others)          # E2B e2e, expected filenames, issue management, sandbox, pi-ai e2e
│   ├── ISSUE_TEMPLATE/       # Bug report, feature request templates
│   └── issue-management/     # Automated issue lifecycle policy + tests
│
├── apps/
│   ├── server/               # Web backend entry point (Node.js, `pnpm start`)
│   │   ├── src/
│   │   │   ├── index.ts      # Process entry: parse args, load env, boot profile
│   │   │   ├── profile-boot.ts  # Compose profile patches, mount Cordis tree, manage shutdown
│   │   │   ├── args.ts       # CLI argument parsing (commander)
│   │   │   ├── dump-config.ts  # `--dump-config` mode
│   │   │   └── process-shutdown.ts  # SIGINT/SIGTERM handling
│   │   ├── config/examples/  # Opt-in overlay examples (GitHub webhooks, MCP memory, schedule, cordis tools)
│   │   ├── reference/        # Backend reference documentation
│   │   ├── tests/            # Server e2e tests
│   │   ├── lib/              # Built output (gitignored in dev)
│   │   ├── package.json      # `@deepseek-ai/dsh` — all runtime dependencies
│   │   └── cordis.patch.yml  # (profile-level patches live here at $DSH_HOME, not in repo)
│   │
│   └── web/                  # Browser application (Vite + React)
│       ├── index.html        # SPA shell with `<div id="root">`
│       ├── src/
│       │   ├── main.ts       # Browser entry: `new AppWebEntry(el).run()`
│       │   ├── preview.ts    # Worker-preview bootstrap (experimental)
│       │   └── node-module-stub.ts  # Stubs Node builtins for browser
│       ├── vite.config.ts    # Vite config: vendor chunks, font routing, preview page emission
│       ├── dist/             # Built browser assets (served by apps/server)
│       ├── tests/            # Playwright browser tests
│       └── stress-tests/     # Stress tests
│
├── native/
│   └── landlock-run/         # `@deepseek-ai/node-addon-landlock-run` — Landlock self-restrict-then-exec launcher
│       ├── packages/
│       │   ├── entry/        # Published entry package (JS API: `launcherPath`, `probe`, `grantArgs`)
│       │   ├── linux-x64/    # Prebuilt static binary (Linux x86_64, musl-linked)
│       │   └── linux-arm64/  # Prebuilt static binary (Linux ARM64, musl-linked)
│       ├── scripts/          # Build, matrix, release scripts
│       ├── test/             # Behavioral tests
│       └── docs/             # Architecture, CLI contract, release procedure, support matrix
│
├── packages/                 # ~45 package groups, ~200+ individual packages
│   ├── core/                 # Product API spine
│   │   ├── agent/            # Agent service: registry, factory, initiator scope
│   │   ├── agent-loop/       # Default driver implementing the Agent interface
│   │   ├── agent-default-model/  # Default model selection per agent
│   │   ├── agent-tool-presentation/  # Tool presentation mode (native function calling / PTC / both)
│   │   ├── session/          # Append-only SessionEvent log and in-memory store
│   │   ├── system-prompt/    # Prompt-section and tool-schema assembly
│   │   ├── tools/            # Scoped tool registry and guarded execution pipeline
│   │   └── scope/            # Per-agent scoped-registration primitive
│   │
│   ├── api/                  # Remote BFF assembly and Typert RPC gateway
│   │   ├── gateway/          # Two-sided Typert RPC: Host gateway + Client remote
│   │   ├── remotes/          # Business Remote service declarations (generated)
│   │   ├── session-controller/  # Session commands, cold reads, live control
│   │   ├── settings-controller/  # Configuration reads/writes over Remote
│   │   └── workspace-controller/  # Workspace commands and projection
│   │
│   ├── typert/               # Type graph generation, loading, and runtime registry
│   │   ├── generator/        # Build-time: analyzes source types → reflection, schemas, Remote descriptors
│   │   ├── loader/           # Auto-registers generated artifacts from Loader compositions
│   │   ├── protocol/         # Remote decorators, wire descriptors, codecs, provider contracts
│   │   └── registry/         # Runtime: stores generated package reflection and Zod schemas
│   │
│   ├── llm/                  # LLM capability family
│   │   ├── llm/              # Provider-neutral model-call service (vocabulary, streaming, adapter seam)
│   │   ├── llm-deepseek/     # DeepSeek API adapter (SSE streaming, tool calling)
│   │   ├── llm-pi-ai/        # Optional pi-ai LLM API backend
│   │   ├── llm-retry/        # Retry execution at durable step boundaries
│   │   ├── deepseek-llm-api-extensions/  # DeepSeek-specific wire extensions
│   │   ├── token-meter/      # Token counting and KV-cache budget tracking
│   │   └── plugin-package-inventory-deepseek/  # Model catalog for DeepSeek
│   │
│   ├── shell/                # Bash capability family
│   │   ├── shell/            # Service Definition
│   │   ├── bash-local/       # Local POSIX shell provider
│   │   ├── bash-sandbox/     # Sandboxed bash provider
│   │   ├── pwsh-local/       # Local PowerShell provider (Windows primary)
│   │   ├── pwsh-sandbox/     # Sandboxed PowerShell provider
│   │   ├── shell-env/        # Shell environment variable management
│   │   ├── tool-bash/        # Model-facing `bash` tool
│   │   ├── tool-bash-persistent/  # Persistent bash session tool
│   │   ├── tool-pwsh/        # Model-facing `pwsh` tool
│   │   └── tool-pwsh-persistent/  # Persistent PowerShell session tool
│   │
│   ├── subprocess/           # Subprocess capability
│   │   ├── subprocess/       # Service Definition
│   │   ├── subprocess-local/ # Local process-tree provider (spawns via `child_process`)
│   │   └── win32-process/    # Shared Win32 process management library
│   │
│   ├── terminal/             # Persistent PTY sessions
│   │   ├── terminal/         # Service Definition (owner-scoped sessions)
│   │   ├── terminal-bash/    # Bash-based terminal provider
│   │   └── tool-terminal/    # Model-facing `terminal` tool
│   │
│   ├── fs/                   # Filesystem capability
│   │   ├── fs/               # Service Definition
│   │   ├── fs-local/         # Local filesystem provider
│   │   ├── fs-observation-policy/  # File access policy (read/write restrictions)
│   │   ├── fs-sandbox/       # Sandboxed filesystem provider
│   │   ├── tool-fs/          # Model-facing `read`/`write`/`edit` tools
│   │   ├── tool-fs-search/   # Model-facing `glob`/`grep` tools
│   │   └── tool-str-replace-editor/  # Model-facing string-replace edit tool
│   │
│   ├── sandbox/              # Process confinement
│   │   ├── sandbox/          # Service Definition
│   │   ├── sandbox-local/    # Local sandbox provider (bwrap/Landlock/Seatbelt)
│   │   ├── sandbox-policy/   # Sandbox policy evaluation
│   │   └── sandbox-windows-acl/  # Windows ACL-based sandbox
│   │
│   ├── skill/                # Skill capability
│   │   ├── skill/            # Service Definition (skill registry)
│   │   ├── skill-badge/      # Badge decoration for skills
│   │   ├── skill-filesystem/ # Filesystem-based skill discovery
│   │   └── tool-skill/       # Model-facing `skill` tool (load/install)
│   │
│   ├── web/                  # Web capability (search, fetch, model-facing tools)
│   │   ├── web/              # Service Definition
│   │   ├── web-fetch-http/   # HTTP fetch provider
│   │   ├── web-search-deepseek/  # DeepSeek web search provider
│   │   ├── web-search-exa/   # Exa web search provider
│   │   ├── web-search-perplexity/  # Perplexity web search provider
│   │   └── tool-web/         # Model-facing `web_search`/`web_fetch` tools
│   │
│   ├── subagent/             # Subagent delegation
│   │   ├── subagent/         # Service Definition (provider registry)
│   │   ├── subagent-fork-in-process/  # Fork-based in-process subagent
│   │   ├── subagent-spawn-in-process/ # Spawn-based in-process subagent
│   │   ├── subagent-in-process-driver/ # Shared in-process driver logic
│   │   ├── subagent-acp/     # ACP (Agent Communication Protocol) subagent
│   │   ├── subagent-claude-code/  # Claude Code subagent
│   │   ├── subagent-codex/   # Codex subagent
│   │   ├── tool-subagent/    # Model-facing `subagent` tool
│   │   ├── tool-subagent-control/  # Model-facing `subagent_fork`/`send_message`/`interrupt_agent` tools
│   │   └── tool-subagent-report/  # Continuable child-agent reporting
│   │
│   ├── compaction/           # Context compaction
│   │   ├── compaction/       # Service Definition
│   │   ├── compaction-basic/ # Basic compaction provider
│   │   ├── compaction-tool-result-pruner/  # Tool result pruning
│   │   └── command-compact/  # `/compact` user command
│   │
│   ├── session/              # Durable session data plane
│   │   ├── session-persistence/  # Persistence seam (write coordination)
│   │   ├── session-persistence-jsonl/  # JSONL log backend (per-session append-only files, optional Zstandard)
│   │   ├── session-checkpoint-policy/  # Makes requests + tool side effects durable before next action
│   │   ├── session-log-deepseek/  # Uploads log as optional official DeepSeek request metadata
│   │   ├── session-projection/  # Projection seam (folds events → current values)
│   │   ├── session-projection-cache/  # Persists projection checkpoints
│   │   ├── session-stats/    # Whole-log conversation counts and wall times
│   │   ├── session-turn-outline/  # Whole-log turn outline for navigation
│   │   ├── session-title/    # Log-backed session titles with fallback
│   │   ├── session-title-llm/  # Shared model-backed title generation policy
│   │   ├── session-title-first-prompt-llm/  # Titles from first eligible human message
│   │   ├── session-title-all-prompts-llm/  # Titles from all eligible messages
│   │   ├── session-telemetry/  # Capture + reporting backend
│   │   └── session-telemetry-otel/  # OpenTelemetry delivery (FULL/FEEDBACK_ONLY/DISABLED)
│   │
│   ├── session-query/        # Session retrieval
│   │   ├── session-query/    # Logical corpus, bounded reads, lineage
│   │   ├── session-query-sqlite/  # SQLite full-text search
│   │   ├── session-log-export/  # Session log export/download
│   │   └── tool-session-query/  # Model-facing session query tools
│   │
│   ├── goal/                 # Same-session goal persistence
│   │   ├── goal/             # Goal service
│   │   ├── goal-round-driver/  # Goal continuation round driver
│   │   ├── command-goal/     # `/goal` user command
│   │   └── tool-goal/        # Model-facing `create_goal`/`update_goal`/`get_goal` tools
│   │
│   ├── plan/                 # Plan mode (logged collaborative planning state)
│   │   └── plan-mode/        # Plan mode plugin with direct entry command
│   │
│   ├── todo/                 # Todo tracking
│   │   └── tool-todo/        # Model-facing `todo_write` tool
│   │
│   ├── context/              # Model-visible request context
│   │   ├── agent-instructions/  # Agent instructions injection
│   │   ├── time-context/     # Current time context
│   │   ├── session-reference/  # Session reference context
│   │   ├── file-reference/   # File reference seam
│   │   ├── file-reference-local/  # Local file reference provider
│   │   └── tmux-context/     # tmux context (session multiplexer awareness)
│   │
│   ├── workflow/             # Workflow capability
│   │   ├── workflow/         # Service Definition
│   │   ├── workflow-worker-thread/  # Worker-thread execution engine
│   │   ├── tool-workflow/    # Model-facing `workflow` tool
│   │   └── tool-ralph/       # Model-facing `ralph` tool (fresh-agent iteration)
│   │
│   ├── jobs/                 # Background job runtime
│   │   ├── jobs/             # Job registry
│   │   ├── jobs-local/       # Local job provider
│   │   └── tool-jobs/        # Model-facing `job_list`/`job_output`/`job_kill` tools
│   │
│   ├── webhook/              # Webhook ingress
│   │   ├── webhook/          # Authenticated delivery dispatch
│   │   └── webhook-github/   # GitHub webhook provider
│   │
│   ├── hook/                 # External hook bridges
│   │   └── hooks-claude-code/  # Claude Code hook bridge
│   │
│   ├── hooks/                # Hook protocol library
│   │   ├── hook-protocol/    # Shared wire-protocol library
│   │   ├── hooks-claude-code/  # Claude Code bridge
│   │   └── hooks-codex/      # Codex bridge
│   │
│   ├── mcp/                  # Model Context Protocol
│   │   └── mcp-client/       # MCP client integration
│   │
│   ├── bundle/               # Profile patch-layer bundles
│   │   ├── base/             # `dsh-base`: agents, models, tools, persistence, sandbox, approval, settings, credentials, telemetry
│   │   └── web-app/          # `dsh-web-app`: HTTP backend, browser app, all UI plugins, agent presets
│   │
│   ├── boot/                 # Application boot glue
│   │   ├── app-boot/         # Profile loading, patch composition, boot helpers, live-reload
│   │   └── cmdline/          # CLI argument resolution, bounded exit, readiness signal
│   │
│   ├── preset/               # Agent composition presets
│   │   ├── agent-presets/    # Preset roster from `cordis.yml` files
│   │   └── persona/          # System prompt persona management
│   │
│   ├── guard/                # Loop-hygiene guards
│   │   ├── repeat-tool-reminder/  # Advisory repeat-call reminders
│   │   └── timeout-policy/   # Tool-call deadline enforcement
│   │
│   ├── extensions/           # Runtime self-modification
│   │   ├── cordis-host-runner/  # Host-side Cordis runtime tools
│   │   ├── cordis-client-runner/  # Client-side Cordis runtime tools
│   │   ├── tool-cordis/      # Model-facing `inspect`/`mount`/`unmount` tools
│   │   └── ui-cordis/        # Client UI for Cordis inspection
│   │
│   ├── host/                 # Web-GUI host half
│   │   ├── webserver/        # HTTP server: route registration, upgrade, index transforms, fallback
│   │   ├── frontend-static/  # SPA dist server (fallback seat)
│   │   ├── plugin-inventory/ # Read-only projection of Loader entries
│   │   ├── directory-picker/ # Directory picker seam
│   │   ├── directory-picker-auto/  # Auto-selects native or browse
│   │   ├── directory-picker-browse/  # Browse-mode picker
│   │   └── directory-picker-native/  # Native OS dialog picker
│   │
│   ├── client/               # Web-GUI browser half
│   │   ├── web/              # Shell boot kernel, CSS, boot page
│   │   ├── modules/          # Client module system (dynamic loading)
│   │   ├── connection/       # Browser↔server transport (fetch, SSE, WebSocket)
│   │   ├── hmr/              # Client-side HMR receiver
│   │   ├── locale/           # Client i18n
│   │   ├── store/            # Snapshot-store engine (`defineStore`, `createSnapshotStore`)
│   │   ├── ui-primitives/    # Shared UI primitives (React components, highlight, math rendering)
│   │   ├── ui-slots/         # Slot system (plugin-composable UI composition)
│   │   ├── ui-renderer/      # Slot renderer/outlets, SessionProvider, uSES adapter
│   │   ├── ui-theme/         # Design tokens, global styles, dark/light themes
│   │   ├── ui-layout/        # Application layout (sidebar + content)
│   │   ├── ui-chat/          # Chat interface
│   │   ├── ui-conversation/  # Conversation node system
│   │   ├── ui-session/       # Session management UI
│   │   ├── ui-sidebar/       # Sidebar navigation
│   │   ├── ui-settings/      # Settings panel root
│   │   ├── ui-settings-general/  # General settings
│   │   ├── ui-settings-models/  # Model settings
│   │   ├── ui-settings-plugins/  # Plugin inventory settings
│   │   ├── ui-settings-plugin-inventory/  # Plugin package inventory
│   │   ├── ui-model-selection/  # Model selector popup
│   │   ├── ui-approval/      # Approval prompt UI
│   │   ├── ui-attachment/    # File attachment UI
│   │   ├── ui-brand-official/  # Official brand slot occupant
│   │   ├── ui-commands/      # Command surface (/ commands)
│   │   ├── ui-deliverables/  # Produced-files row
│   │   ├── ui-goal/          # Goal bar in input dock
│   │   ├── ui-input-trigger/  # `/` and `@` input pipeline
│   │   ├── ui-jobs/          # Background jobs list
│   │   ├── ui-message-feedback/  # Like/Dislike per message
│   │   ├── ui-permission-presets/  # Permission preset UI
│   │   ├── ui-plan/          # Plan control
│   │   ├── ui-reference/     # Reference sources
│   │   ├── ui-schedule/      # Schedule UI
│   │   ├── ui-skill/         # Skill catalog UI
│   │   ├── ui-subagent/      # Subagent UI
│   │   ├── ui-tool/          # Tool call tree / views
│   │   ├── ui-trajectory/    # Agent trajectory display
│   │   ├── ui-user-questions/  # User question prompt UI
│   │   ├── ui-workflow-run/  # Workflow lifecycle UI
│   │   ├── ui-workspace/     # Workspace picker UI
│   │   └── ui-agent-preset/  # Agent preset selection UI
│   │
│   ├── settings/             # User settings
│   │   ├── settings/         # Settings seam
│   │   └── settings-file/    # File-backed settings provider
│   │
│   ├── credentials/          # Credential management
│   │   ├── credentials/      # Credential seam
│   │   ├── credentials-local/  # Env/.env credential provider
│   │   └── authorization/    # Authorization flow (ask a human)
│   │
│   ├── interaction/          # Human collaboration
│   │   ├── commands/         # Command registration and dispatch
│   │   ├── permission-presets/  # Permission preset definitions
│   │   ├── tool-ask-user/    # `ask_user_question` tool
│   │   ├── user-approval/    # Approval prompt seam
│   │   └── user-questions/   # User question seam
│   │
│   ├── identity/             # User identity
│   │   └── anonymous-user-id/  # Anonymous user identity
│   │
│   ├── storage/              # Non-session storage
│   │   ├── storage/          # Storage hub
│   │   ├── storage-domain/   # Domain storage
│   │   ├── storage-json/     # JSON file backend
│   │   └── storage-sqlite/   # SQLite backend
│   │
│   ├── spill/                # Tool-result spilling
│   │   ├── spill/            # Spill seam
│   │   ├── spill-local/      # Local spill provider
│   │   └── spill-policy/     # Spill policy
│   │
│   ├── attachment/           # Durable attachment identity
│   │   ├── attachment/       # Attachment seam
│   │   └── attachment-local/ # Local content-addressed storage
│   │
│   ├── code-runtime/         # Code execution
│   │   ├── code-runtime/     # Service Definition
│   │   ├── code-runtime-python/  # Python execution provider
│   │   └── code-runtime-worker-thread/  # Worker-thread execution provider
│   │
│   ├── feedback/             # User feedback
│   │   ├── command-feedback/  # Feedback command
│   │   └── message-feedback/  # Per-message feedback
│   │
│   ├── schedule/             # Session-local scheduled follow-ups
│   │   └── schedule/         # Schedule service
│   │
│   ├── workspace/            # Workspace entity
│   │   └── workspace/        # Workspace management
│   │
│   ├── lsp/                  # Language Server Protocol
│   │   ├── lsp/              # Service Definition
│   │   ├── lsp-stdio/        # Stdio provider
│   │   └── tool-lsp/         # Model-facing `lsp` tool
│   │
│   ├── e2b/                  # E2B remote sandbox (POC)
│   │   ├── e2b/              # E2B service
│   │   ├── fs-e2b/           # E2B filesystem provider
│   │   └── subprocess-e2b/   # E2B subprocess provider
│   │
│   ├── experimental/         # Private prototypes (excluded from official releases)
│   │   ├── agent-team/       # Agent team coordination
│   │   ├── agent-team-profile/  # Team profile management
│   │   ├── agent-team-web-profile/  # Team web profile
│   │   ├── client-ui-agent-team/  # Team UI
│   │   ├── inspector/        # Cordis inspector
│   │   ├── tool-agent-team/  # Team tools
│   │   ├── webworker-packer/ # Web worker image packing
│   │   └── webworker-runtime/  # Web worker Cordis runtime
│   │
│   ├── util/                 # Zero-dependency utilities
│   │   ├── atomic-write/     # Atomic file writes
│   │   ├── brand/            # Branded types (`Branded<B>`)
│   │   ├── crypto/           # Cryptographic utilities
│   │   ├── deque/            # Double-ended queue
│   │   ├── home-paths/       # Harness home resolver (`resolveDshHome`)
│   │   ├── launch-environment/  # Launch-time environment snapshot
│   │   ├── native-command/   # Native command resolution
│   │   ├── output-retention/ # Output retention logic
│   │   ├── time/             # Time utilities
│   │   ├── timeout/          # Timeout utilities
│   │   ├── values/           # Deep-freeze, value utilities
│   │   └── workspace-path/   # Workspace path resolution
│   │
│   ├── runtime-diagnostics/  # Runtime invariant checks
│   │   └── invariants/       # Invariant registry and reporters
│   │
│   └── test-support/         # Dev/test infrastructure
│       ├── agent-loop-testkit/  # Agent loop testing utilities
│       ├── client-runtime/   # Client runtime test support
│       ├── llm-mock-server/  # Mock LLM API server (runs via `pnpm run mock:llm`)
│       ├── llm-replay/       # LLM replay for snapshot tests
│       ├── loader-smoke/     # Loader smoke tests
│       └── session-snapshot/ # Session snapshot test harness
│
├── vendor/                   # Vendored Cordis framework (see §2)
│
├── docs/                     # Documentation tree
│   ├── architecture.md       # This project's architecture (the authoritative internal version)
│   ├── development.md        # Contributor setup and daily workflow
│   ├── testing.md            # Testing policy and execution model
│   ├── glossary.md           # Project terminology
│   ├── cordis-primer.md      # Introduction to Cordis
│   ├── defensive-patterns.md # Lifecycle, concurrency, subprocess, teardown patterns
│   ├── capability-seams.md   # Capability seam graph
│   ├── event-producer-consumer.md  # Event map (producers/consumers)
│   ├── tool-execution-pipeline.md  # Tool pipeline reference
│   ├── agent-lifecycle.md    # Agent lifecycle sequence diagram
│   ├── api-gateway.md        # API gateway reference
│   ├── config-catalog.md     # Generated: all configuration fields
│   ├── tool-catalog.md       # Generated: all model-facing tools
│   ├── persistence-catalog.md  # Generated: all persistence backends
│   ├── module-graph.md       # Generated: peer dependency graph (Mermaid)
│   ├── rescope.md            # Vendored package name mapping
│   ├── web-styling.md        # CSS token and styling guide
│   ├── deepseek-llm-api-wire-extensions.md  # DeepSeek wire protocol
│   ├── subsystems/           # One reference page per subsystem (~40 pages, bilingual)
│   ├── cookbook/              # Step-by-step guides (adding a package, tool, adapter, settings card, etc.)
│   ├── cordis-api/           # Generated Cordis core API reference
│   ├── cordis-tutorial/      # Cordis tutorial
│   ├── user/                 # Product-facing user guides
│   │   ├── guide/            # User guides
│   │   ├── develop/          # Developer guides
│   │   └── skills/           # Skill documentation
│   ├── i18n/                 # Internationalization workflow
│   └── postmortem/           # Incident stories
│
├── website/                  # VitePress documentation site
│   ├── .vitepress/           # VitePress configuration
│   ├── .generated/           # Disposable projection of docs/ into Markdown (gitignored)
│   ├── .dist/                # Build output
│   ├── docs.ts               # Publication manifest (routes, pages, locales)
│   ├── build.ts              # Production build script
│   ├── package.json          # `@deepseek-ai/website`
│   └── public/               # Static assets
│
├── scripts/                  # ~173 repository scripts (gates, generators, release tools)
│   ├── build.ts              # Full repository build orchestrator
│   ├── run-gates.ts          # CI gate runner (check:ci, check:all, hygiene, doc-sync)
│   ├── run-oxlint.ts         # Lint runner wrapper
│   ├── gen-module-graph.ts   # Module dependency graph generator
│   ├── gen-cordis-catalog.ts  # Cordis catalog generator
│   ├── gen-tool-catalog.ts   # Tool catalog generator
│   ├── gen-config-catalog.ts  # Config catalog generator
│   ├── gen-third-party-notices.ts  # Third-party license notice generator
│   ├── clean.ts              # Build output cleaner
│   ├── verify-*.ts           # ~30 verification scripts (doc-budgets, translation-pairing, package-invariants, etc.)
│   ├── release/              # Release scripts (bump, verify, pack, publish)
│   └── dev-web.ts            # Web HMR watcher for client development
│
├── snapshots/                # Test fixtures
│   ├── session/              # Recorded session snapshots
│   ├── web/                  # Web UI snapshots
│   ├── acp/                  # ACP snapshots
│   └── sdk/                  # SDK snapshots
│
├── patches/                  # pnpm patch files (e.g., node-pty patch)
│
└── .dsh-build/               # Build record (gitignored)
```

---

## 5. Application Architecture

### 5.1 The Web Backend (Server)

The backend is the **only shipped application entry point**. There is no standalone CLI task runner, public TypeScript or Python SDK, or ACP server application.

**Boot sequence**:
1. `apps/server/src/index.ts` parses CLI args, loads layered `.env` files (invocation directory + Harness home + inherited environment), and calls `runProfile()`.
2. `runProfile()` loads the `web` profile from `$DSH_HOME/profiles/web`, composes its patch layers in order:
   - Bundle patches (from `dsh-base` then `dsh-web-app`)
   - Profile's own `cordis.patch.yml`
   - Home-level `$DSH_HOME/cordis.patch.yml`
   - `--patch` overlay files
   - Telemetry opt-out switch
3. An empty root `cordis.yml` is written to the profile directory.
4. The Cordis Loader mounts the complete plugin tree over this empty root using the composed patch layers.
5. `boot()` resolves all plugin dependencies, activates fibers in topological order, and returns the running `Context`.
6. Live patch reload is installed: edits to `cordis.patch.yml` or the home patch re-compose and re-apply without restart. Module HMR is optional.
7. SIGINT/SIGTERM trigger bounded disposal and exit.

**Configuration model**: Every row in the composed entry list is a named Cordis plugin with an optional `config` object. Rows can be inserted, replaced, or disabled by `id` in patch layers. `!!js` expressions evaluate at boot time against `ctx` and `process.env`.

**Harness Home** (`$DSH_HOME`): Typically `~/.dsh/`. Contains `profiles/web/` (profile manifest, patches), `cordis.patch.yml` (user customization), `.env` (user environment), and `.agent-presets/` (user-authored presets).

### 5.2 The Web Frontend (Browser)

The browser application is a **React SPA** built with Vite. It is NOT a standalone application — it depends on `window.__DSH_BOOT__` injected by the Host server.

**Boot sequence**:
1. `apps/web/index.html` loads `src/main.ts` as an ES module.
2. `main.ts` creates `AppWebEntry` from `@deepseek-ai/dsh-client-web`.
3. The shell boot kernel reads `window.__DSH_BOOT__` (injected by Host's index transforms), creates a `ClientModuleSystem`, and fetches plugin bundles from `/plugins/<id>/client.js`.
4. A Cordis `Context` is created in the browser.
5. Each client plugin activates: registers React components into **slots**, subscribes to Remote streams, and connects to the Host via the connection transport.
6. The UI renderer mounts the slot tree into `<div id="root">`.

**Communication**: The browser communicates with the backend over:
- **HTTP API** (`/api`): Typert RPC (unary request/response)
- **WebSocket** (`/api/remote.mux`): Multiplexed streams (events, agent lifecycle)
- **SSE**: Server-Sent Events for real-time updates

### 5.3 Composition Architecture (Profiles + Bundles)

The product is assembled through **configuration composition**, not code imports:

1. **Profile** (`$DSH_HOME/profiles/web/`): Named composition template. Declares ordered `bundles` and `external` plugin dependencies.
2. **Bundle** (`dsh-base`, `dsh-web-app`): Installable patch layers. Each declares `dsh.bundle.patch` in its `package.json` pointing to a `cordis.patch.yml`.
3. **Patch** (`cordis.patch.yml`): YAML document that targets rows by `id` — replaces config or inserts new rows. Applied in layer order.
4. **Overlay** (`--patch`): Additional patches applied on top.

The **empty root config** (`[]`) is patched over by all layers to produce the final entry list. This means there is no monolithic config file — the entire composition is emergent from the patch stack.

**Bundles in order**:
1. `dsh-base`: Core runtime — agents, models, tools, persistence, sandbox, approval, settings, credentials, telemetry, session query, subagent registries
2. `dsh-web-app`: Web surface — HTTP server, browser app, all `ui-*` plugins, API remotes, agent presets

---

## 6. Core Architecture: The Agent System

### 6.1 Session Log (Event Source of Truth)

The session log is an **append-only sequence of `SessionEvent` objects**. It is the authoritative source for:
- Model context (messages the model sees)
- UI rendering (chat history)
- Fork/resume (conversation continuity)
- Telemetry and reporting

**Key rule**: "Model-visible means logged." Anything that reaches a model request must be reconstructable from the log. A new model-visible input requires a new session event type in `SessionEventMap`.

**Durable event types** include: `turn/start`, `turn/end`, `step/start`, `step/end`, `user/message`, `assistant/chunk`, `assistant/message`, `tool/call`, `tool/result`, `request/header`, and more.

### 6.2 Agent Lifecycle

An **Agent** is a live session worker. The lifecycle:

1. **Creation**: `AgentRegistry.create()` mints a `SessionId`, creates the session, creates the agent context (`agentCtx`), runs agent setup (scoped tool registrations, prompt sections), and publishes the agent.
2. **Turn flow**: A **turn** is zero or more **steps**. Each step is one model request plus the tools it calls.
3. **Step flow**:
   - `agent/pre-step`: Plugins decide what the model sees (may rewrite or reject messages)
   - Prompt sections + tool schemas are assembled
   - `agent/request` → LLM streaming → `assistant/chunk*` → `assistant/message`
   - `tool/call*` → `tools/pre-execute` → `tools/execute` → `tools/post-execute` → `tool/result*`
4. **Turn end**: `agent/turn-stopping` allows plugins to intervene.

### 6.3 The Default Agent Loop (`ReactLoopAgent`)

The `dsh-agent-loop` package provides `ReactLoopAgent`, the default driver:
- Reads input from an `Inbox`
- Manages phase state (idle/maintenance/running)
- Calls `deriveMessages()` to project model history from the session log
- Assembles prompts through `dsh-system-prompt`
- Streams requests through `dsh-llm`
- Executes tools through `dsh-tools`
- Manages step boundaries and turn lifecycle

### 6.4 Prompt Assembly

`dsh-system-prompt` owns prompt-section and tool-schema assembly:
- **Sections**: Named text blocks registered by plugins (workspace instructions, time context, harness source, web surface context)
- **Tool schemas**: JSON Schema definitions from registered tools, presented in native function-calling format or PTC (Prompt-Tool-Call) mode
- **Variables**: Template variables like `{{cwd}}`, `{{model}}`, `{{workspacePath}}`

### 6.5 Tool Pipeline

`dsh-tools` provides the scoped tool registry and guarded execution pipeline:
- **Registration**: `defineTool()` builds a typed tool definition (name, description, parameter schema, output schema, execute body)
- **Presentation mode**: Native function calling, PTC mode, or both (configurable per-session)
- **Execution pipeline**: Allow/deny/ask policy → monotonic guards → around-dispatch wrappers → result inspection → content finalization → observe-only notification
- **Scoping**: Tools can be scoped to a specific agent via `agent.ctx`

### 6.6 Capability Seams

A **capability seam** is a swappable capability with three roles:
1. **Service Definition**: Declares the interface (e.g., `dsh-fs`)
2. **Service Provider**: Implements it (e.g., `dsh-fs-local`)
3. **Consumer**: Model-facing tool that uses it (e.g., `dsh-tool-fs`)

One provider swap changes the whole product. Filesystem and subprocess providers share one execution world — pointing them at a remote sandbox moves Bash, PTY, and LSP with them.

Current capability seams: filesystem, subprocess, shell, terminal, sandbox, LLM, web, skill, compaction, subagent, workflow, session persistence, session projection, settings, credentials, interaction, approval, storage, spill, attachment, code-runtime, plan, goal, jobs, webhook, MCP.

---

## 7. Build System

### 7.1 TypeScript Project Layout

The repository uses **isolated Host and Client aggregates** under two separate TypeScript programs:

| File | Role |
|---|---|
| `tsconfig.base.json` | Shared compilerOptions + source `paths` map (no `include`/`files` — serves as resolution facade) |
| `tsconfig.base.client.json` | Browser compiler settings (jsx, DOM libs) |
| `tsconfig.json` | Solution root: `extends` base, `files: []`, references host + client aggregates (no program) |
| `tsconfig.host.json` | Host aggregate program (Host packages, scripts, tests, website) |
| `tsconfig.client.json` | Client aggregate program (packages/client/*, apps/web) |
| `tsconfig.host.tsbuildinfo` / `tsconfig.client.tsbuildinfo` | Incremental build caches |

**Why two aggregates**: Both sides declaration-merge the Cordis `Context` interface under the same keys with different services; one program seeing both merges reports a collision.

**Source plane vs artifact plane**: Static gates and tests resolve workspace imports through `tsconfig.paths` to `src` and pass on a clean tree; gates consuming built `lib/` output declare that dependency explicitly.

### 7.2 Build Phases

The root `pnpm run build` executes:

```
1. tsc -b tsconfig.host.json     →  lib/types/ (TypeScript declarations + JS for every Host package)
2. tsdown --env.DSH_BUILD_FACE host  →  lib/ (Runtime bundles for Host packages; runs Typert generator)
3. tsc -b tsconfig.client.json   →  lib/types/ (Client packages)
4. tsdown --env.DSH_BUILD_FACE client  →  lib/ + lib/client.js (Browser bundles for Client packages)
5. pnpm run build:web            →  apps/web/dist/ (Vite React build → static assets)
```

**tsdown** is the workspace-wide bundler. It consumes only the JavaScript emitted to `lib/types/` by the preceding tsc phase. Package-local `tsdown.config.ts` files select entries based on `DSH_BUILD_FACE`.

**Typert** (type graph generator) runs only during Host tsdown. It analyzes Host types and generates:
- Host reflection artifacts
- Host-for-Client Remote projections (typed Client-side method stubs)

### 7.3 Client Build Environment

`pnpm run build` embeds into the client artifacts:
- Root package version
- 7-character source commit
- Dirty marker (when Git reports local changes)
- User-supplied `DSH_CLIENT_*` values

A successful build writes a gitignored record binding exact public values to artifacts. Release packing and built Web tests reject stale records.

### 7.4 Vite Configuration (apps/web)

The Vite build:
- **Vendor chunks**: Heavy render libraries (katex, shiki, micromark/mdast) are grouped into a `vendor` chunk
- **Boot grammars**: TypeScript, ShellScript, JSON syntax highlighting grammars are in vendor
- **Lazy grammar chunks**: Other @shikijs/langs grammars get individual chunks under `assets/langs/`
- **Font assets**: KaTeX woff2/woff/ttf → `assets/fonts/`
- **Two entries**: `index.html` (main SPA) and `preview.ts` (worker-preview bootstrap)
- **Deduplication**: React and React-DOM are explicitly deduplicated to prevent identity splitting

### 7.5 Watch Mode (Development)

`pnpm run dev:web` starts a Vite build watcher that rebuilds client bundles on source changes. Client-plugin HMR is active: changes to `packages/client/*/src/client/` trigger automatic reload without full page refresh. Server-side changes require `pnpm start` restart.

---

## 8. Testing Strategy

### 8.1 Test Runner (Vitest)

The project uses **vitest** with multiple configuration profiles:

| Config | Purpose |
|---|---|
| `vitest.config.ts` | Default: unit tests across all packages |
| `vitest.e2e.config.ts` | Real-API end-to-end tests (self-skip without `DEEPSEEK_API_KEY`) |
| `vitest.expected.config.ts` | Owner-local process expectation tests |
| `vitest.snapshot.config.ts` | Keyless recorded-session replay tests |
| `vitest.web.config.ts` | Web browser smoke tests (Playwright + dist replay) |
| `vitest.web.perf.config.ts` | Performance regression tests |
| `vitest.web-stress.config.ts` | Stress tests |
| `vitest.shared.ts` | Shared vitest configuration helpers |

### 8.2 Test Categories

- **Unit tests** (`pnpm run test`): Every package under `packages/*/tests/`. Per-file 100% coverage gate on `packages/*/*/src`.
- **Coverage** (`pnpm run test:coverage`): CI gate with partitioned execution.
- **E2E** (`pnpm run test:e2e`): Real-API tests requiring `DEEPSEEK_API_KEY`.
- **Snapshot** (`pnpm run test:snapshot`): Keyless recorded-session replay through shipped profiles. Fixtures replay identically without an API key.
- **Expected** (`pnpm run test:expected`): Owner-local process expectation tests.
- **Web** (`pnpm run test:web`): Rebuilds frontend, then runs Playwright browser smoke tests with snapshot replay.
- **GUI** (`pnpm run test:gui`): Client suites + host-side GUI packages (no browser, fast).
- **Specs** run concurrently in forked workers beside other gate processes.

### 8.3 Testing Conventions

- Tests live at package level under `tests/`, not `src/__tests__/`.
- `// @vitest-environment jsdom` pragma per spec (not global config).
- Components render with realistic props; assert user-visible behavior, not internals.
- Product-visible changes MUST include a keyless recorded-session snapshot.
- Non-trivial changes MUST include an Agent Note in the same PR.

### 8.4 CI Gates

The `scripts/run-gates.ts` orchestrator runs gate aggregates:
- `check:ci:static` — lint, doc-sync, hygiene, package constraints, third-party notices
- `check:ci:coverage` — exhaustive per-file 100% coverage
- `check:ci:consumers` — compatibility, snapshot, artifact gates
- `check:ci:windows-blocking` — Windows build + site
- `check:ci:windows-observational` — Non-blocking Windows checks

---

## 9. CI/CD (GitHub Actions)

### 9.1 Pull-Request CI (`ci.yml`)

Nine blocking jobs + one verdict:

| Job | Runner | Content |
|---|---|---|
| `node-24` | Ubuntu (16-core) | Static gates (lint, doc-sync, constraints, etc.) |
| `node-24-coverage` | Ubuntu (16-core) | Exhaustive coverage (4 partitions, 6 workers) |
| `node-24-consumers` | Ubuntu (16-core) | Compatibility, snapshots, artifacts (Playwright) |
| `node-compat` | Ubuntu (matrix: 22.19, 24.9, 26) | Node version compatibility smokes |
| `windows` | Ubuntu (Wine) | Windows Node 24 under Wine (blocking) |
| `windows-build` | Windows (16-core) | Windows build + site |
| `windows-coverage` | Windows (16-core) | Windows coverage (4 partitions) |
| `windows-native-tests` | Windows (16-core) | Windows-specific native tests |
| `windows-observational` | Windows (16-core) | Non-blocking observational checks |
| `all-checks-passed` | Ubuntu | Aggregation verdict for branch protection |

**Failover**: Self-hosted VM pool available via repository variables `DSH_CI_FAILOVER_LINUX` and `DSH_CI_FAILOVER_WINDOWS`.

### 9.2 Master CI (`ci-master.yml`)

Runs on master push + workflow_dispatch:
- Serial Windows/Linux standby lanes (validate failover targets)
- Wine apt cache seeder
- Manual runner benchmarks

### 9.3 Other Workflows

- `release.yml` / `release-vendor.yml` / `release-vendor-publish.yml` — Package publishing
- `landlock-run.yml` / `landlock-run-release.yml` — Native binary build + npm release
- `e2e.yml` / `pi-ai-provider-e2e.yml` — Real-API end-to-end tests
- `docs-pages.yml` — Documentation site deployment to GitHub Pages
- `sandbox.yml` — Sandbox testing
- `issue-lifecycle.yml` / `issue-policy.yml` — Automated issue management

---

## 10. Key Conventions and Invariants

### 10.1 Module System
- **ESM everywhere**: `"type": "module"` in every `package.json`.
- **Import syntax**: `.ts` extensions in local relative imports (rewritten by tsc to `.js` in output).
- **Package names across packages**: `import { ... } from '@deepseek-ai/dsh-llm'`, never relative paths between packages.

### 10.2 Plugin Architecture
- **Registrations are effects**: Every contribution goes through `ctx.effect()` / `ctx.on()`.
- **Service Definition / Provider / Consumer**: Every capability seam is complete with all three roles.
- **Extension plugins depend on Service Definitions, never concrete providers**.
- **No hardcoded tunables**: Deployment-varying choices are validated `Config` fields changeable from `cordis.yml`.

### 10.3 Type Safety
- **`strict: true`** with `noImplicitAny`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`.
- **Branded types**: Cross-boundary opaque IDs use `Branded<B>` from `dsh-brand`, never bare `string`.
- **Switch on discriminant tags**: Closed unions end in `assertNever`; merge-extensible unions fall through a documented default.

### 10.4 Event and Logging Discipline
- **Model-visible = logged**: Anything reaching a model request is reconstructable from the log.
- **Waterfall listeners MUST call `next()`** to delegate.
- **Typed events use declaration merging** and merge-extensible maps.
- **`SessionEventMap` members are required-on-read** by default.

### 10.5 Documentation Discipline
- **Every non-trivial change includes an Agent Note** in the same PR.
- **Docs accompany every code change**: Update README and JSDoc contracts together.
- **Bilingual (English + Chinese)**: Every user-facing document has a Chinese counterpart.
- **Wordcount budgets** enforced by `verify-doc-budgets`.
- **`verify-md-wrap`**: One physical line per paragraph.
- **Generated catalogs** are English-only, regenerated from source, and freshness-gated.

### 10.6 Package Naming
- Every npm package: `@deepseek-ai/dsh-<name>`.
- Every vendored package: `@deepseek-ai/cordis[-plugin-<x>]`, `@deepseek-ai/cosmokit`, `@deepseek-ai/schemastery`.
- Client packages: `@deepseek-ai/dsh-client-<name>`.
- Tests: `tests/` directory at package level.

### 10.7 Git Hooks (Lefthook)
- **pre-commit**: Translation pairing, archived agent notes, lint (oxlint with autofix), third-party notices regeneration, whitespace check, vendor manifest guard.
- **pre-merge-commit**: Translation pairing, archived agent notes.
- **pre-push**: Typecheck.

### 10.8 Invariant System
- Every package owns `./invariant` (companion module).
- `pnpm run constraints` checks workspace constraints.
- `verify-package-invariants` enforces package-owned invariant contracts.

---

## 11. How to Rebuild From Scratch

### 11.1 Prerequisites
```
Node.js ^22.19.0 or >=24.0.0
Corepack (built into Node)
pnpm 11.7.0 (enabled via `corepack enable`)
Git 2.26+
Optional: DEEPSEEK_API_KEY for real-API tests
Optional: musl-tools (for landlock-run native build on Linux)
```

### 11.2 Clone and Install
```bash
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
corepack enable
pnpm install
```

### 11.3 Build
```bash
pnpm run build
```

This executes the full five-phase build:
1. Host TypeScript compilation (`tsc -b tsconfig.host.json`)
2. Host runtime bundling (`tsdown --env.DSH_BUILD_FACE host`) — includes Typert type-graph generation
3. Client TypeScript compilation (`tsc -b tsconfig.client.json`)
4. Client runtime bundling (`tsdown --env.DSH_BUILD_FACE client`)
5. Web frontend build (`vite build` in `apps/web`)

### 11.4 Run
```bash
pnpm start
```

Opens `http://127.0.0.1:3080` in the default browser. The backend boots the `web` profile, serves the browser SPA, and provides agents, tools, settings, and session persistence.

### 11.5 Development Workflow
```bash
# Typecheck (quick feedback)
pnpm run typecheck

# Unit tests
pnpm run test

# Lint
pnpm run lint

# Full test:web (rebuilds frontend + Playwright smoke tests)
pnpm run test:web

# Dev mode with HMR
pnpm run dev:web    # In one terminal (watches client bundles)
pnpm start          # In another terminal (starts server)

# Mock LLM server (for testing without API key)
pnpm run mock:llm
```

### 11.6 Available Scripts Summary
| Script | Purpose |
|---|---|
| `pnpm start` | Start the Web backend |
| `pnpm run build` | Full repository build |
| `pnpm run build:lib` | Build all package lib/ output |
| `pnpm run build:web` | Build browser frontend |
| `pnpm run clean` | Remove build outputs |
| `pnpm run typecheck` | TypeScript type checking |
| `pnpm run lint` | oxlint linting |
| `pnpm run test` | Unit tests |
| `pnpm run test:coverage` | Per-file 100% coverage gate |
| `pnpm run test:e2e` | Real-API e2e tests |
| `pnpm run test:snapshot` | Keyless session replay tests |
| `pnpm run test:web` | Web browser smoke tests |
| `pnpm run test:gui` | Fast client+host GUI tests |
| `pnpm run duplication` | Clone detection |
| `pnpm run check:all` | Full gate suite |
| `pnpm run check:ci` | CI primary gates |
| `pnpm run hygiene` | Package publication checks |
| `pnpm run dev:web` | Client HMR watcher |
| `pnpm run mock:llm` | Mock LLM API server |
| `pnpm run docs:dev` | Documentation site dev server |
| `pnpm run docs:build` | Documentation site build |

---

## 12. Key Files Reference

| File | Purpose |
|---|---|
| `package.json` | Root workspace: scripts, devDependencies, engine requirements |
| `pnpm-workspace.yaml` | Workspace definition, overrides, peerDependencyRules, patchAllowList |
| `tsconfig.base.json` | Shared TypeScript compiler options + source paths map |
| `tsconfig.host.json` | Host aggregate program |
| `tsconfig.client.json` | Client aggregate program |
| `tsdown.config.ts` | Workspace-wide bundler config |
| `vitest.config.ts` | Default test config |
| `lefthook.yml` | Git hooks configuration |
| `.oxlintrc.json` | Linting rules |
| `.jscpd.json` | Clone detection config |
| `apps/server/src/index.ts` | Application entry point |
| `apps/server/src/profile-boot.ts` | Profile composition and boot orchestration |
| `apps/server/package.json` | All server runtime dependencies |
| `apps/web/src/main.ts` | Browser entry point |
| `apps/web/vite.config.ts` | Vite build configuration |
| `packages/bundle/base/cordis.patch.yml` | Base composition layer |
| `packages/bundle/web-app/cordis.patch.yml` | Web-app composition layer (browser roster, API layer, host/guest split) |
| `vendor/README.md` | Vendored package manifest, local modifications, sync procedure |
| `docs/architecture.md` | Internal architecture reference |
| `AGENTS.md` | Standing orders for AI agents working on the project |
| `.github/workflows/ci.yml` | Pull-request CI pipeline |
| `.github/workflows/ci-master.yml` | Master-push CI pipeline |

---

## 13. Architectural Decision Records

The project maintains decision records as **Agent Notes** under `.agents/notes/`:

- `implemented/`: Active decisions in present-tense authority. Subdirectories: `architecture/`, `feature/`, `process/`.
- `archived/`: Frozen historical decisions (read-only).
- `proposed/`: Decisions under discussion.
- `rejected/`: Rejected proposals.

Every non-trivial code change must include an Agent Note in the same PR. Notes document the **why**, what was given up, and required verification.

---

## 14. Internationalization (i18n)

- **Bilingual documentation**: Every user-facing doc has an English `.md` and Chinese `.zh.md` counterpart.
- **Translation pairing**: `.i18n.yaml` files track English↔Chinese pairs. A Git merge driver and `verify-translation-pairing` keep them synchronized.
- **Client UI copy**: All product text lives in typed locale dictionaries. `t` function or localized props are used. `verify-client-ui-i18n` rejects hardcoded copy.
- **Website**: VitePress with `zh-CN` locale support.

---

## 15. Release Process

1. `pnpm run release:dsh` or `pnpm run release:vendor` — bump versions
2. `pnpm run release:verify` — verify release readiness
3. `pnpm run release:pack` — pack npm tarballs
4. `pnpm run release:publish` — publish to npm

The `landlock-run` native packages have a separate release workflow (`landlock-run-release.yml`) that builds on per-architecture runners and publishes the three-package npm family.

---

## 16. Summary: Key Architectural Principles

1. **Everything is a plugin**: No privileged core to patch. Extend by mounting plugins; registrations are effects that unwind on unload.
2. **Configuration as composition**: The running system emerges from ordered patch layers, not monolithic config files.
3. **Capability seams**: Swappable Service Definition / Provider / Consumer trios. One provider swap changes the whole product.
4. **Model-visible = logged**: Complete session log is the source of truth for model context, UI, fork/resume, and telemetry.
5. **Two TypeScript programs**: Host and Client aggregates are separate to avoid Context merge collisions.
6. **Source plane vs artifact plane**: Tests resolve to `src`; gates consuming `lib/` declare that dependency.
7. **Fail-loud at every boundary**: Misconfiguration, missing dependencies, and sandbox failures exit nonzero with clear diagnostics.
8. **Pre-release stance**: Prefer correct foundations to compatibility shims. Rename/repackage freely; backends reject old on-disk formats.
9. **Tests describe behavior, not correctness**: Change obsolete behavior with its tests; explain why in the PR.
10. **Documentation accompanies code**: Every change updates README, JSDoc, and subsystem pages.
