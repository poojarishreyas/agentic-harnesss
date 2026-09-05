# DeepSeek Harness — Phase Scope (Rebuild Roadmap)

> **Purpose**: This document translates [`architecture.md`](architecture.md) into an **ordered, dependency-driven build sequence**. It defines, for every phase of rebuilding the DeepSeek Harness project from a blank tree, exactly what is in scope, what is explicitly out of scope, which packages and directories are touched, what the deliverables are, and what must be true before the phase is accepted. It is the build order companion to `architecture.md` (what the system is) and `AGENTS.md` (the standing rules of the project); this file answers *"in what order, and up to what boundary?"*.
>
> **Intended reader**: an AI agent or engineer rebuilding the project from scratch, phase by phase, running the named verification commands as each phase's acceptance gate.

---

## 1. How to Use This Document

1. **Follow the phases in order.** Each phase's scope is a strict subset of the next; earlier phases never depend on later ones. The ordering follows the workspace dependency graph, the capability-seam rule ("a seam is complete: Service Definition / Service Provider / Consumer"), and the two-TypeScript-program build architecture.
2. **A phase is done only when its exit criteria pass.** Every phase ends with a concrete, runnable verification. Do not start the next phase on a red gate.
3. **Keep the standing rules in force at all times.** The repo-wide conventions in `AGENTS.md` (ESM everywhere, `@deepseek-ai/dsh-*` naming, registrations as effects, runtime invariants per package, tests describing behavior, Agent Notes for non-trivial changes, bilingual docs) apply inside every phase, not only at these gates.
4. **Coverage is a phase-gate, not an afterthought.** The CI-mandated per-file 100% coverage on `packages/*/*/src` is enforced from Phase 1's first package onward (`pnpm run test:coverage`). New code ships with its tests in the same phase.
5. **This is the rebuild order, not the runtime topology.** After every phase the artifact tree must still satisfy the source-plane rule: tests resolve workspace imports through `tsconfig.base.json` `paths` to `src`; only gates that explicitly consume built `lib/` output may require a build.

### 1.1 Cross-cutting phase invariants

Every phase, without exception, maintains:

- All packages typed under `strict: true` with `noImplicitAny`; every remaining `any` carries an explanation.
- Every package registers its `./invariant` companion (or an explained empty rationale).
- Every capability added is a **complete seam** — never one role alone.
- Every upgradeable package boundary keeps an explicit `resolve(request): Spec` step rather than a hidden default inside `run()`.
- Every model-visible input lands in the session log (`SessionEventMap`), never as an unlogged side channel.
- Unit tests live under `packages/<group>/<pkg>/tests/`, not in `src/__tests__/`.
- Non-trivial behavior changes arrive with an Agent Note in `.agents/notes/` in the same change.

---

## 2. Phase Map (Overview)

| Phase | Name | Establishes | Depends on |
|---|---|---|---|
| P0 | Repository scaffolding and toolchain | Workspace, git, lint, hooks, build skeleton | — |
| P1 | Vendored framework layer | `vendor/*` (Cordis, Cosmokit, Schemastery, Loader, Include, Group, Timer, HMR, Logger) | P0 |
| P2 | Zero-dependency utilities | `packages/util/*`, `runtime-diagnostics/invariants` | P0 |
| P3 | Types and seals | Brand, identity, storage, workspace entity, scope | P2 |
| P4 | Session spine | `core/session`, `core/scope`, session persistence/JSONL/projection | P3 |
| P5 | LLM capability | `llm/llm`, adapters, token meter, retry | P2, P4 |
| P6 | Tools and system prompt | `core/tools`, `core/system-prompt`, presentation modes | P4 |
| P7 | Agent and agent loop | `core/agent`, `core/agent-loop`, default model | P4, P5, P6 |
| P8 | Execution seams | fs, subprocess, shell, terminal, sandbox, code-runtime + their model-facing tools | P2, P4, P6 |
| P9 | Context and safety | context plugins, guard plugins, compaction, spill, attachment | P4, P6 |
| P10 | Intelligence seams | web, skill, lsp, subagent, jobs, goal, workflow, webhook, todo, plan, schedule, feedback, mcp | P6, P7, P8 |
| P11 | Boot and base bundle | `boot/app-boot`, `boot/cmdline`, `bundle/base` | All host-seam phases |
| P12 | Typert and API gateway | `typert/*`, `api/gateway`, `api/remotes` | P4, P6, P7 |
| P13 | Host web half | `host/webserver`, `host/frontend-static`, `api/session-controller`, `api/settings-controller`, `api/workspace-controller` | P11, P12 |
| P14 | Client shell | `client/web`, `client/modules`, `client/connection`, `client/store`, `client/locale`, `client/hmr` | P12 |
| P15 | Client UI machinery | `client/ui-primitives`, `client/ui-slots`, `client/ui-renderer`, `client/ui-theme`, `client/ui-layout` | P14 |
| P16 | Client feature UI | all remaining `client/ui-*` plugin packages | P15 |
| P17 | Web profile composition | `bundle/web-app`, `apps/server`, `apps/web` end-to-end boot | P13–P16 |
| P18 | Test and snapshot hardening | snapshot fixtures, web replay, GUI test tiers, coverage partitions | P17 |
| P19 | Docs, website, i18n | `docs/`, `website/`, translation pairing | P18 |
| P20 | CI/CD and release | `.github/workflows/*`, release pipeline, native `landlock-run` | P19 |

---

## 3. Phase P0 — Repository Scaffolding and Toolchain

### Goal
Create a green, typed, linted monorepo skeleton that can later host every package in `architecture.md` §4.

### Scope — In

- Root files: `package.json` (`@deepseek-ai/dsh-root`, `private: true`, `"type": "module"`, `packageManager: pnpm@11.7.0`, engines `node ^22.19.0 || >=24.0.0`), `pnpm-workspace.yaml` (workspace globs `vendor/*`, `packages/*/*`, `native/landlock-run`, `native/landlock-run/packages/*`, `apps/*`, `website`), `.gitignore`, `.editorconfig`, `.gitattributes`, `LICENSE` (MIT), `README.md` (+ `.zh.md`), `SAFETY.md`, `CONTRIBUTING.md`.
- TypeScript layout: `tsconfig.base.json` (strict, es2024, ESNext modules, `paths` map — no `include`/`files`), `tsconfig.base.client.json`, `tsconfig.host.json`, `tsconfig.client.json`, solution `tsconfig.json` (`files: []`, references the two aggregates).
- Build toolchain: `tsdown.config.ts` (workspace bundler, `DSH_BUILD_FACE` host/client passes, `typertPlugin` wired once P12 lands), `scripts/build.ts` (five-phase orchestrator), `scripts/clean.ts`.
- Lint and hygiene: `.oxlintrc.json`, `.oxlintrc.staged.json`, `scripts/run-oxlint.ts`, `.jscpd.json`, `vitest.config.ts` + `vitest.shared.ts` (vitest 4, `vite-tsconfig-paths` against `tsconfig.base.json`), `pytest.ini`.
- Git hooks: `lefthook.yml` (pre-commit: translation pairing, archived notes, staged lint, third-party notices, whitespace, vendor manifest guard; pre-push: typecheck), `scripts/install-lefthook.mjs`, `postinstall` wiring.
- Dev dependencies: typescript ^6, tsx ^4, vitest ^4, oxlint 1.76, tsdown ^0.22, lefthook ^2, jscpd, publint.

### Scope — Out
- Any `packages/` content (starts P2).
- Any `vendor/` imports (starts P1).
- Application code (`apps/`, `website/`).

### Deliverables
- A working `pnpm install` with pinned pnpm, hooks installed.
- `pnpm run typecheck` green (empty but valid aggregate programs).
- `pnpm run lint` green against the empty tree.
- `pnpm run test` green (empty or seed test).

### Exit Criteria
- `pnpm install --frozen-lockfile` succeeds from a lockfile.
- `pnpm run typecheck && pnpm run lint && pnpm run test` all exit 0.
- `git commit` passes the pre-commit hook set.

---

## 4. Phase P1 — Vendored Framework Layer

### Goal
Own the framework layer: Cordis + foundation libraries as pinned, scoped, source-vendored packages with the documented local modifications.

### Scope — In
- Create `vendor/` with the nine packages from `vendor/README.md` (cosmokit, schemastery, cordis, loader, include, group, timer, hmr, logger-console), each renamed to `@deepseek-ai/*` scope.
- Apply the full "Local modifications" log from `vendor/README.md` (19 entries): fiber lifecycle hardening, JSDoc enrichment, transactional Loader/include reconciliation, lazy config resolution, `!!js` entry-`disabled` interpolation, internal-loader runtime shape detection, rescope, `src` publication, entry-patch extraction (`applyEntryPatches`, `entryListSchema`), serialized child-tree mutation, durable debounced writes, exact-config watching.
- `pnpm-workspace.yaml` `linkWorkspacePackages: true` + `overrides` mapping `@deepseek-ai/cosmokit` and `@deepseek-ai/schemastery` to `link:vendor/<pkg>`; `peerDependencyRules.allowedVersions.typescript` `>=5 <7`; `allowBuilds` for esbuild/lefthook/node-pty/koffi; `patchedDependencies` for node-pty; `minimumReleaseAgeExclude` for the reviewed runtime closure (pi-ai, node-addon-*, claude-agent-sdk-*, openai/codex).
- `vendor/README.md` manifest table with upstream SHAs; `scripts/check-vendor-manifest.sh`; `docs/rescope.md`.
- Root `tsconfig.base.json` `paths` entries resolving vendored scoped names to their `vendor/<pkg>/src`.

### Scope — Out
- Any harness package importing the vendor (starts P2).
- Modifying vendored `src/` beyond the logged local-modification list.

### Deliverables
- All nine vendored packages compile under the root base config.
- A passing `pnpm run test` at vendor level.
- The manifest gate (`scripts/check-vendor-manifest.sh`) green.

### Exit Criteria
- `pnpm run typecheck` (Host aggregate) compiles every `vendor/*/src`.
- `pnpm run test` green.
- `pnpm run hygеne` / `verify-vendored-links` confirms every vendored name resolves to a workspace `link:` with no registry copy.

---

## 5. Phase P2 — Zero-Dependency Utilities

### Goal
Lay the leaf layer of the dependency graph: small, well-tested, zero-dependency utility packages plus the runtime-invariant infrastructure every package will need.

### Scope — In
- `packages/util/`: `brand` (`Branded<B>`), `values` (deep-freeze, value utils), `time`, `timeout`, `deque`, `atomic-write`, `home-paths` (`resolveDshHome`), `launch-environment`, `native-command`, `output-retention`, `crypto`, `workspace-path`.
- `packages/runtime-diagnostics/invariants`: the invariant registry and reporters (`ctx.invariants`), plus `@deepseek-ai/dsh-invariants` path wiring in `tsconfig.base.json`.
- Package skeletons per `packages/AGENTS.md`: `package.json` (`@deepseek-ai/dsh-<name>`, ESM, exports `.`, `./invariant`, `./src/*`, `./package.json`), `tsconfig.json` (extends base, `rootDir: src`, project references), `src/types.ts` (types only), `src/invariant.ts`, `tests/`.
- Per-package JSDoc contracts (`@param`/`@returns`), `verify-export-jsdoc` compatible.

### Scope — Out
- Anything importing another package group (no `core/`, `llm/`, etc. dependencies).
- Duplicating functionality the vendored layer already provides (check vendor first).

### Deliverables
- Complete leaf utility layer with unit tests at 100% per-file coverage.
- The invariant infrastructure usable by P4+.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- `pnpm run test:coverage` reports 100% on `packages/util/*/src` and `packages/runtime-diagnostics/invariants/src`.
- `pnpm run constraints` (workspace constraints) green.

---

## 6. Phase P3 — Types and Seals

### Goal
Establish the shared entity types, branded IDs, storage domains, and the per-agent scoped-registration primitive that later phases build on.

### Scope — In
- `packages/storage/`: `storage` (hub), `storage-domain` (domain form), `storage-json` (JSON file backend), `storage-sqlite` (SQLite backend) — the durable non-session storage seam.
- `packages/workspace/`: the `workspace` entity.
- `packages/identity/`: `anonymous-user-id` (anonymous identity).
- `packages/attachment/`: `attachment` (seam) + `attachment-local` (content-addressed storage) — durable attachment identity and validation.
- `packages/settings/`: `settings` (seam) + `settings-file` (file-backed provider).
- `packages/credentials/`: `credentials` + `credentials-local` (env-over-`.env` provider) + `authorization` (ask-a-human flows).
- `packages/core/scope`: the per-agent scoped-registration primitive (library, no `ctx` key).
- Wire these into `tsconfig.host.json` aggregate references.

### Scope — Out
- Session log (P4), LLM (P5), tools (P6), agent (P7).
- Any debugger, file, or adapter beyond the domain entities.

### Deliverables
- Branded, type-safe domain entities usable by the session spine.
- Storage backends with CRUD + durability tests.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- Coveraged 100% per-file on all scoped packages.
- `pnpm run constraints` green.

---

## 7. Phase P4 — Session Spine

### Goal
Create the durable conversation log: append-only `SessionEvent` log, live store, persistence seam + JSONL backend, checkpoint policy, projections, titles, and telemetry — the data plane the entire agent system renders from.

### Scope — In
- `packages/core/session`: `Session` service (`ctx.sessions`), `SessionEvent` types, `SessionEventMap` (merge-extensible, declaration merged into `@deepseek-ai/cordis`), `deriveMessages()`, fork/resume, `SESSION_FORMAT_VERSION` (0, no compatibility promise).
- `packages/session/`: `session-persistence` (seam + write coordination), `session-persistence-jsonl` (append-only JSONL, optional Zstandard), `session-checkpoint-policy` (durable before next action), `session-log-deepseek` (canonical-log upload metadata), `session-projection` + `session-projection-cache`, `session-stats`, `session-turn-outline`, `session-title` + `session-title-llm` + `session-title-first-prompt-llm` + `session-title-all-prompts-llm`, `session-telemetry` + `session-telemetry-otel` (FULL / FEEDBACK_ONLY / DISABLED).
- `packages/session-query/`: `session-query` (logical corpus, bounded reads, lineage) + `session-query-sqlite` (full-text) — the read/tool surface can consume persistence independently.
- `packages/attachment/` integration where session events reference durable attachments.

### Scope — Out
- Model calls (P5), tool registry (P6), agent driver (P7).
- Anything that writes a session event from a tool or prompt (later phases).

### Deliverables
- A complete durable session log with replay, fork, resume, titles, projections, and telemetry.
- Invariant: model-visible ⟺ logged (runtime invariant asserting it).

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- 100% per-file coverage on all scoped packages.
- A seed integration test writes a session, persists it to JSONL, reloads, and derives identical message history.

---

## 8. Phase P5 — LLM Capability

### Goal
Stand up the provider-neutral model-call service and the DeepSeek provider: the one vocabulary the loop, session log, titles, and compaction all speak.

### Scope — In
- `packages/llm/llm`: service (`ctx.llm`), message/content-block/stream-chunk vocabulary, adapter seam, request deep-freeze, per-request retry-policy capture, model metadata resolution.
- `packages/llm/llm-deepseek`: DeepSeek API adapter (SSE streaming, tool calling, reasoning-effort/max-tokens defaults).
- `packages/llm/deepseek-llm-api-extensions`: DeepSeek wire extensions; `packages/llm/plugin-package-inventory-deepseek` (model catalog).
- `packages/llm/llm-pi-ai` (optional pi-ai backend), `packages/llm/llm-retry` (durable step-boundary retry), `packages/llm/token-meter` (token/KV-cache budgets).
- The `llm/*` events (waterfall `llm/stream`) and codec for the session log (`request/header` etc. can come with P7).

### Scope — Out
- Agent loop integration (P7).
- Web fetch/search providers (P10).

### Deliverables
- A callable LLM service with at least one working adapter, replayable through `llm-replay`/`llm-mock-server` fixtures (test-support packages may be seeded here).

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test:gui` green.
- Mock-server streaming test proves request → chunks → assistant message round trip.
- Coverage 100% per-file on `llm/llm` and the shipped adapters.

---

## 9. Phase P6 — Tools and System Prompt

### Goal
Build the scoped tool registry with its guarded execution pipeline and the prompt-section/tool-schema assembly.

### Scope — In
- `packages/core/tools`: `defineTool`, `ctx.tools` registry, native function-calling + PTC presentation modes (and `presentAs` per-agent shadowing), execution pipeline: allow/deny/ask policy, monotonic guards, around-dispatch wrappers, result inspection, definition-owned content finalization, observe-only notification; `tools/*` events (waterfalls `tools/pre-execute`, `tools/execute`, `tools/post-execute`).
- `packages/core/system-prompt`: prompt-section registration, tool-schema assembly, `{{var}}` templating.
- `packages/core/agent-tool-presentation`: presentation-mode selection.
- `packages/guard/`: `timeout-policy` (tool-call deadlines) and `repeat-tool-reminder` (advisory).
- `packages/interaction/`: `commands` (dispatch without model turn), `permission-presets`, `tool-ask-user`, `user-approval`, `user-questions` — the human-collaboration plane tools need.
- Model-facing tool contracts document Model Experience (prompts, schemas, results) per package README rules.

### Scope — Out
- Concrete execution providers (fs/shell/web/etc. — P8/P10).
- Agent loop driving the pipeline (P7).

### Deliverables
- A registry where any future tool registers and becomes model-visible automatically; schemas flow into prompt assembly.
- Policy hooks for sandbox/approval enforcement.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- Pipeline test proves deny/allow/ask paths through the executor (not just the registry).
- Coverage 100% per-file on `core/tools`, `core/system-prompt`, `guard/*`.

---

## 10. Phase P7 — Agent and Agent Loop

### Goal
Drive sessions: the `Agent` interface, live registry, per-agent scope, and the default `ReactLoopAgent` implementing turns and steps over the session log + LLM + tools.

### Scope — In
- `packages/core/agent`: `Agent` interface, `AgentRegistry.create()` (factory with scoped setup commit, session/agent/session-start events), initiator scope (`ctx.agents.withInitiator()`), inbox, consumed-work, model-selection, `agent/*` events (`pre-step`, `request`, `turn-stopping`, status, continuation), runtime invariant companions.
- `packages/core/agent-loop`: `ReactLoopAgent` — phase machine (idle/maintenance/running), turn/step boundaries, deriveMessages → prompt assembly → `llm/stream` → tool pipeline, step-end decisions (completed/max-tokens), cancellation and error recovery; `dsh-agent-loop` is swappable.
- `packages/core/agent-default-model`: default model selection per agent.
- Session-log codecs for `turn/*`, `step/*`, `user/message`, `assistant/*`, `tool/*`, `request/header` events (extend `SessionEventMap`).

### Scope — Out
- Concrete capability tools beyond a smoke `todo_write`-style example (P8/P10).
- Composition/bundling (P11).

### Deliverables
- A working conversational loop: `turn/start` → claim → `agent/pre-step` → stream → tool calls → `turn/end`, all durable in the session log.
- The agent event surfaces UI and hooks can observe.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- `agent-loop-testkit` (test-support) drives one full turn with a mock LLM, asserting the session log order and event emission.
- Coverage 100% per-file on `core/agent`, `core/agent-loop`, `core/agent-default-model`.

---

## 11. Phase P8 — Execution Seams

### Goal
Add the capability seams that give the agent real execution power: every seam complete with Service Definition / Provider / Consumer.

### Scope — In
- **Filesystem**: `fs/fs` (seam), `fs/fs-local`, `fs/fs-sandbox`, `fs/fs-observation-policy`, `fs/tool-fs`, `fs/tool-fs-search`, `fs/tool-str-replace-editor`.
- **Subprocess**: `subprocess/subprocess` (seam), `subprocess/subprocess-local` (local process-tree provider), `subprocess/win32-process` (shared Win32 library).
- **Sandbox**: `sandbox/sandbox` (seam), `sandbox/sandbox-local` (bwrap/Landlock/Seatbelt), `sandbox/sandbox-policy`, `sandbox/sandbox-windows-acl`.
- **Shell**: `shell/shell` (seam), `shell/bash-local`, `shell/bash-sandbox`, `shell/pwsh-local`, `shell/pwsh-sandbox`, `shell/shell-env`, `shell/tool-bash`, `shell/tool-bash-persistent`, `shell/tool-pwsh`, `shell/tool-pwsh-persistent` (exactly one shell stack per platform).
- **Terminal**: `terminal/terminal` (owner-scoped PTY seam), `terminal/terminal-bash`, `terminal/tool-terminal`.
- **Code runtime**: `code-runtime/code-runtime` (seam), `code-runtime/code-runtime-python`, `code-runtime/code-runtime-worker-thread` (PTC-mode consumer).
- Wire `fs`/`subprocess` providers to share one execution world (a remote sandbox moves bash/PTY/LSP together).

### Scope — Out
- Web/skill/LSP seams (P10) — same pattern, deferred.
- Agent presets that scope these per-session (P17).

### Deliverables
- Run bash, read/write files, spawn processes, and run confined commands through model-facing tools, all behind sandbox/approval policy.
- Tool results spill-able (careful: `spill` is P9) — keep results inline until then.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green; real provider suites self-skip appropriately per platform (Windows excludes bash-requiring suites via vitest config).
- A behavioral tool test executes `bash -c` and reads a file through the seam, asserting the model-facing result JSON.
- Coverage 100% per-file on all scoped seam packages.

---

## 12. Phase P9 — Context and Safety

### Goal
Add model-visible request context and the guards/compaction/spill/attachment machinery that keep sessions healthy.

### Scope — In
- **Context**: `context/agent-instructions`, `context/time-context`, `context/session-reference`, `context/file-reference` + `file-reference-local`, `context/tmux-context`.
- **Compaction**: `compaction/compaction` (seam), `compaction/compaction-basic`, `compaction/compaction-tool-result-pruner`, `compaction/command-compact`.
- **Spill**: `spill/spill` (seam), `spill/spill-local`, `spill/spill-policy` — tool-result spill policies (belongs here; consumers in P8 are updated to retain/persist results).
- **Attachment** integration (durable attachment identity from P3 now consumed by tools).
- Model Experience documentation for each new model-visible section/tool.

### Scope — Out
- Agent-team/experimental coordination (P20 / excluded).
- UI for these (P16).

### Deliverables
- Richer context (time, references, instructions) with KV-cache-aware ordering.
- Compaction/spill keeping long sessions within token budgets.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- Compaction path proves a compacted session re-derives equivalent model context.
- Coverage 100% per-file on all scoped packages.

---

## 13. Phase P10 — Intelligence Seams

### Goal
Add the remaining capability families: web, skill, LSP, subagents, background jobs, goals, workflows, webhooks, todo, plan, schedule, feedback, MCP.

### Scope — In
- **Web**: `web/web` (seam), `web/web-fetch-http`, `web/web-search-deepseek`, `web/web-search-exa`, `web/web-search-perplexity`, `web/tool-web` (`web_search`/`web_fetch`).
- **Skill**: `skill/skill` (provider registry + catalog/loader), `skill/skill-badge`, `skill/skill-filesystem`, `skill/tool-skill`.
- **LSP**: `lsp/lsp` (seam), `lsp/lsp-stdio`, `lsp/tool-lsp`.
- **Subagent**: `subagent/subagent` (provider registry, process singleton), `subagent/subagent-fork-in-process`, `subagent/subagent-spawn-in-process`, `subagent/subagent-in-process-driver`, `subagent/subagent-acp`, `subagent/subagent-claude-code`, `subagent/subagent-codex`, `subagent/tool-subagent`, `subagent/tool-subagent-control`, `subagent/tool-subagent-report`.
- **Jobs**: `jobs/jobs` (background-job runtime), `jobs/jobs-local`, `jobs/tool-jobs`.
- **Goal**: `goal/goal`, `goal/goal-round-driver`, `goal/command-goal`, `goal/tool-goal` (same-session goal lifecycle).
- **Workflow**: `workflow/workflow` (seam), `workflow/workflow-worker-thread`, `workflow/tool-workflow`, `workflow/tool-ralph` (fresh-agent loop).
- **Webhook**: `webhook/webhook` (authenticated dispatch), `webhook/webhook-github`.
- **Todo**: `todo/tool-todo`. **Plan**: `plan/plan-mode`. **Schedule**: `schedule/schedule`. **Feedback**: `feedback/command-feedback`, `feedback/message-feedback`. **MCP**: `mcp/mcp-client`.
- **Hooks**: `hooks/hook-protocol`, `hooks/hooks-claude-code`, `hooks/hooks-codex` (external hook bridges).

### Scope — Out
- Host API remotes exposing these to the browser (P12/P13).
- UI for these (P16).
- `experimental/*` (P20 / excluded from official builds).

### Deliverables
- The complete tool catalog (in the shape `docs/tool-catalog.md` will document): web, skill, subagents, goals, workflows, jobs, webhook-driven sessions.
- Tool-result spilling and timeout policies applied uniformly.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green.
- A `subagent` delegation test proves child-agent create → report → collect round trip with a mock LLM.
- A `workflow` test proves multi-agent fan-out with structured results.
- Coverage 100% per-file across the phase; the process-singleton rules from `packages/AGENTS.md` hold under HMR-safety tests.

---

## 14. Phase P11 — Boot and Base Bundle

### Goal
Make the whole host-side tree bootable: profile loading, patch composition, environment, fail-loud, and the base bundle that assembles the default runtime.

### Scope — In
- `packages/boot/app-boot`: profile discovery/initialization, bundle resolution, patch parsing/composition, `boot()`, `installFailLoud`, activation audit, config dump, live patch watching, snapshot replay (`cordis.snapshot.yml` swap), module fallback, harness-source section.
- `packages/boot/cmdline`: launcher-to-app argv handoff, bounded exit, readiness signal.
- `packages/preset/`: `agent-presets` (per-session composition from preset `cordis.yml`), `persona`.
- `packages/bundle/base`: `dsh-base` bundle — the first patch layer inserting core rows over the empty profile root (agents, models, tools, persistence, sandbox/approval policy, settings, credentials, telemetry, session-query, spill, subagent registry, guard, plan, goal, commands, ask-user, time-context, agent-instructions, web tools, skill registry, workflow, jobs, token-meter).
- `packages/guard/` composition, `packages/context/*` mounting, and profile patch-reload semantics (`patchReload: live | startup`).
- Ship `packages/bundle/base/cordis.patch.yml` mirroring the current composition (base rows, including the platform-gated shell rows and telemetry switch).

### Scope — Out
- HTTP server and browser surface (P13–P17).
- External bundles and `--patch` overlays beyond base.

### Deliverables
- `dsh --dump-config` prints the composed entry list with provenance comments.
- A headless boot of the base tree activates every row; failures produce one labelled line with the failing plugin and nonzero exit.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test:gui` green.
- `apps/server`-style boot test with a fake app mounts/unmounts the base bundle cleanly (via `app-boot` testkit).
- Live patch reload test: edit profile patch → tree recomposes without restart; invalid edit → last good app keeps running.
- Coverage 100% per-file on `boot/*` and `bundle/base`.

---

## 15. Phase P12 — Typert and API Gateway

### Goal
Provide typed Client→Host calls: the build-time type-graph generator, runtime registry, and the two-sided RPC endpoint.

### Scope — In
- `packages/typert/generator`: build-time source-type analysis → reflection, schemas, Remote descriptors; `typertPlugin` wired into `tsdown.config.ts` Host pass; fixture corpus + tests.
- `packages/typert/loader`: auto-registers generated artifacts from Loader compositions.
- `packages/typert/protocol`: `@Remote`/`@RemoteScope` decorators, wire descriptors, codecs, provider contracts.
- `packages/typert/registry`: `ctx.typert` — package reflection + Zod schemas, lookup + Context provider registries.
- `packages/api/gateway`: `ctx.typertGateway` (Host), `ctx.remote` (Client), shared `InvocationDescriptor` contract, strict/SRC modes, cancellation-aware + streaming remotes, WebSocket mux heartbeat, `registerRemoteEvents`.
- `packages/api/remotes`: business Remote service declarations (generated Host-for-Client projection).

### Scope — Out
- HTTP route mounts on a real server (P13).
- Feature plugins crossing the wire (P13/P16).

### Deliverables
- A generated contract through which a Client can call a Host business method with validation on both sides.
- Streams (SSE/WS) and cancellation with reconnect recovery.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test` green (Typert generator spec corpus passes; generated fixtures compile under both aggregates).
- `packages/api/gateway/tests` prove unary + stream + cancellation + reconnect recovery paths.
- Coverage 100% per-file on typert and api.

---

## 16. Phase P13 — Host Web Half

### Goal
Serve HTTP: the webserver, SPA dist serving, and the API controllers the browser will call.

### Scope — In
- `packages/host/webserver`: `node:http` server with named exact/prefix routes, upgrade routes, index injection/transforms, single fallback seat; loopback-only default posture (`127.0.0.1`), optional `0.0.0.0`, gzip compression.
- `packages/host/frontend-static`: SPA dist server owning the fallback seat, `renderIndex` on index responses.
- `packages/host/plugin-inventory`: read-only projection of Loader entries over trusted RPC.
- `packages/host/directory-picker*`: `directory-picker` seam + `-auto` / `-browse` / `-native` providers.
- `packages/api/session-controller`, `packages/api/settings-controller`, `packages/api/workspace-controller`: session/config/workspace commands and streams over Typert Remote.
- `packages/session/session-log-export` (`/export` command), `packages/session-query/session-log-export` browser controller.
- Client-side transport counterpart preparation: `client/connection` node half binds the gateway under `/api` (full pairing lands P14).

### Scope — Out
- Browser shell and React (P14–P16).
- The `web` profile bundle (P17).

### Deliverables
- A browser-reachable HTTP surface: SPA served at `/`, RPC at `/api`, WS mux at `/api/remote.mux`, HMR stream, plugin bundles at `/plugins/<id>/client.js`.

### Exit Criteria
- `pnpm run typecheck && pnpm run lint && pnpm run test:gui` green.
- `host/webserver/tests/webserver.spec.ts`-style suite proves route matching, fallback, index transforms, upgrade handling, and failure containment.
- Coverage 100% per-file on the host packages.

---

## 17. Phase P14 — Client Shell

### Goal
Boot the browser shell: module system, transport, store engine, locale, and HMR — everything that must exist before a single React component renders.

### Scope — In
- `packages/client/web`: boot kernel (`AppWebEntry`), boot page, `base.css`, `window.__DSH_BOOT__` consumption, loader status rendering, static module seeding (`getStaticModules`).
- `packages/client/modules`: client module system — dynamic plugin loading, module table, boot manifest, `/plugins/<id>/client.js` fetch, preload tiers, `dsh.client` manifest semantics.
- `packages/client/connection`: browser↔server transport — fetch/SSE client, WebSocket mux (Remote streams), generations, reconnect backoff, `connection/reset` events, the `__DSH_TRANSPORT__` preview hook.
- `packages/client/store`: snapshot-store engine (`defineStore`, `createSnapshotStore`, `shallowEqual`) — bare observable sources, no hook members.
- `packages/client/locale`: typed locale dictionaries + `t` seat.
- `packages/client/hmr`: client-side HMR receiver.
- Client build pipeline: package-local `tsdown.config.ts` (`clientBundle(...)`), `lib/client.js` artifacts, `verify-client-packages` alignment.
- `apps/web/vite.config.ts` adjustments (dedupe react/react-dom, `node:module` stub, boot-manifest-free shell rejection, preview page emission) and `apps/web/src/main.ts`.

### Scope — Out
- UI composition machinery (P15) and feature plugins (P16).
- Server-side pairing beyond what P13 prepared.

### Deliverables
- A booting browser shell that constructs the Cordis `Context`, loads every client plugin from `window.__DSH_BOOT__`, and connects to the Host.
- The clean error surface when bundles fail (boot page shows plugin failures).

### Exit Criteria
- `pnpm run test:gui` green (client suites, jsdom per-file pragmas, node-env default).
- Browser shell test: `AppWebEntry` boots against a driven manifest and mounts after all entries activate.
- `pnpm run build:web` produces `dist/` with the two-entry layout (`index` + `preview`/`bootstrap`).

---

## 18. Phase P15 — Client UI Machinery

### Goal
Stand up the composition + rendering machinery that every feature plugin registers into: slots, primitives, renderer, theme, layout.

### Scope — In
- `packages/client/ui-primitives`: shared React primitives (markdown/math rendering, highlight, typography shells) — vendor-chunk-aware (katex/shiki/micromark in Vite `vendor` chunk).
- `packages/client/ui-slots`: the slot system — one API (`ctx.slots.register({ name, children?, store?, inject? }, Component)`), `SlotMap` declaration merging, slot-name paths (`<domain>.<entry>.<hole>`), children = declaration + authorization.
- `packages/client/ui-renderer`: slot renderer/outlets, `SessionProvider`, `useSession`/`useSessions`/`useWorkspaces`/`useStore`/`renderSlot` framework hooks, uSES adapter; the only ctx→React integration point.
- `packages/client/ui-theme`: `--dsw-*` design tokens, global stylesheets, dark/light themes.
- `packages/client/ui-layout`: sidebar + content layout shell, `'root'` slot occupancy.
- Props discipline: components receive four shares (`PropsRuntime`, `PropsRenderSlots`, `PropsStore`, inject face) — never `ctx`, never hand-rolled hooks.

### Scope — Out
- Feature UI (P16).
- Conversation-node machinery beyond renderer basics (P16 `ui-conversation`).

### Deliverables
- A working slot tree: a fixture plugin registers into `'root'` and renders through the renderer with the share types.

### Exit Criteria
- `pnpm run test:gui` green; component specs feed props directly and assert user-visible behavior.
- `pnpm run verify-client-packages` and `verify-client-domain-graph` green.
- Coverage 100% per-file on client machinery packages.

---

## 19. Phase P16 — Client Feature UI

### Goal
Ship the feature UI plugins: chat, conversation, sessions, settings, tools, subagents, goals, jobs, workflows, etc.

### Scope — In
- `client/ui-chat`, `client/ui-conversation` (ConversationNodeDefinition + keyed renderers), `client/ui-session`, `client/ui-sidebar`.
- `client/ui-settings`, `client/ui-settings-general`, `client/ui-settings-models`, `client/ui-settings-plugins`, `client/ui-settings-plugin-inventory`.
- `client/ui-model-selection`, `client/ui-approval`, `client/ui-attachment`, `client/ui-brand-official`, `client/ui-commands`, `client/ui-deliverables`, `client/ui-goal`, `client/ui-input-trigger`, `client/ui-jobs`, `client/ui-message-feedback`, `client/ui-permission-presets`, `client/ui-plan`, `client/ui-reference`, `client/ui-schedule`, `client/ui-skill`, `client/ui-subagent`, `client/ui-tool`, `client/ui-trajectory`, `client/ui-user-questions`, `client/ui-workflow-run`, `client/ui-workspace`, `client/ui-agent-preset`.
- `client/ui-cordis` (runtime self-modification UI) + `extensions/cordis-client-runner`, `extensions/cordis-host-runner`, extensions tooling (`tool-cordis`) — the self-modification seam.
- Styling per `docs/web-styling.md`; product copy via typed locale dictionaries only.

### Scope — Out
- The `web` profile bundle wiring them all (P17).
- `experimental/*` UI (P20 / excluded).

### Deliverables
- The complete browser GUI: conversation, model/settings management, session history, tool call trees, approvals, goals, jobs, subagents, plans.

### Exit Criteria
- `pnpm run test:gui` green across all UI packages.
- `pnpm run verify-client-ui-i18n` green (no hardcoded copy).
- Browser smoke: a fixture session renders chat, tool cards, and settings panels without a live backend (replay mode).
- Coverage 100% per-file on all client feature packages.

---

## 20. Phase P17 — Web Profile Composition

### Goal
Compose the shipped product: the `web` profile bundle over `dsh-base`, the `apps/server` entry, and the `apps/web` build, end-to-end.

### Scope — In
- `packages/bundle/web-app`: `dsh-web-app` patch layer — browser roster (`dsh.client` rows → `window.__DSH_BOOT__`), host transport rows (webserver, web-runtime, client-hmr, connection, modules), API remotes (session/settings/workspace controllers), UI plugin roster, agent-preset rows (`dsh-agent-presets` with `default: standard`), per-surface config restatements (persona prompt, `:memory:` SQLite `openAt: never`, tools mode env seam, disabled native DeepSeek adapter), host-plane vs agent-plane split (which rows stay host-owned vs move behind presets).
- `apps/server`: `src/index.ts`, `src/args.ts`, `src/profile-boot.ts`, `src/dump-config.ts`, `src/process-shutdown.ts`, `package.json` full dependency closure, `config/examples/*` overlays.
- `apps/web`: final `index.html`, `vite.config.ts`, `src/*`, `dist/` served through `frontend-static`.
- Boot: `$DSH_HOME/profiles/web` initialization, module fallback healing, live patch reload, telemetry switch (`DSH_TELEMETRY_DISABLED`).
- The Host-plane/agent-plane ownership rules (which rows stay host-owned: `shell-env`, jobs registry, skill registry, goals service, token meter, subagent registry) exactly as documented in the bundle patch.

### Scope — Out
- CI packaging (P20).
- Experimental agent-team/profile bundles.

### Deliverables
- `pnpm start` boots the complete product: browser opens to a working chat with model access, tools, durable sessions, approvals, settings, and saved sessions.

### Exit Criteria
- `pnpm run build && pnpm start` → browser connects, agent completes a turn with a mock LLM.
- `pnpm run test:web` (rebuild + Playwright smoke pair) green in replay mode.
- `pnpm run test:snapshot` green: recorded sessions replay identically through shipped profiles.
- `--dump-config` shows the composed profile tree with all rows and patch provenance.

---

## 21. Phase P18 — Test and Snapshot Hardening

### Goal
Make the test matrix exhaustive and deterministic: snapshots, replay, GUI tiers, coverage partitions, stress/perf.

### Scope — In
- Snapshot infrastructure: `test-support/session-snapshot`, `test-support/llm-replay`, `test-support/llm-mock-server`, `test-support/client-runtime`, `test-support/loader-smoke`, `test-support/agent-loop-testkit`.
- `snapshots/` trees: `session/`, `web/`, `acp/`, `sdk/`, `AGENTS.md` ownership rules; `pnpm run test:snapshot:record` (needs key), `test:snapshot:refresh`, `test:snapshot`.
- Web replay + perf + stress: `vitest.web.config.ts`, `vitest.web.perf.config.ts`, `vitest.web-stress.config.ts`, `scripts/run-web-snapshots.ts`.
- Coverage partitioning: `scripts/coverage-partitions.ts`, `scripts/coverage-exempt.ts`, `scripts/run-coverage-partitions.ts`, `.coverage-times.json` weighting.
- Expected-filenames policy (`test:expected`), packed-session fixture migration.

### Scope — Out
- New feature work (beyond snapshot updates for changed behavior).

### Deliverables
- A CI-equivalent green run locally: `pnpm run test`, `test:coverage`, `test:snapshot`, `test:web`, `test:e2e` (with key), `test:expected`.

### Exit Criteria
- `pnpm run check:all` green on a clean tree.
- Every top-level `snapshots/` tree has an owned AGENTS policy and freshness gate.

---

## 22. Phase P19 — Docs, Website, i18n

### Goal
Complete the documentation tier: architecture, development, testing, catalogs, subsystem pages, cookbook, user guides, and the VitePress site, bilingual.

### Scope — In
- `docs/` per the tier taxonomy in `docs/AGENTS.md`: `architecture.md`, `development.md`, `testing.md`, `glossary.md`, `cordis-primer.md`, `defensive-patterns.md`, `capability-seams.md`, `event-producer-consumer.md`, `tool-execution-pipeline.md`, `agent-lifecycle.md`, `api-gateway.md`, `config-catalog.md`, `tool-catalog.md`, `persistence-catalog.md`, `module-graph.md`, `rescope.md`, `web-styling.md`, `subsystems/*`, `cookbook/*`, `cordis-api/*`, `cordis-tutorial/*`, `user/*`, `i18n/*`, `postmortem/*`.
- Generated catalogs wired to generators: `gen-module-graph`, `gen-tool-catalog`, `gen-config-catalog`, `gen-cordis-catalog`, `gen-persistence-catalog`, `gen-cordis-api`, `gen-client-catalog`, `gen-doc-graphs`, `gen-third-party-notices`; every `verify-*` counterpart.
- Website: `website/` VitePress adaptor, `docs.ts` publication manifest, `build.ts` (raw-Markdown twins + `llms.txt`), `.vitepress` config, `docs-pages.yml` deployment.
- i18n: `.i18n.yaml` pairing records, translation merge driver, `verify-translation-pairing`, `verify-md-wrap`, `verify-md-links`, `verify-doc-refs`, `verify-doc-budgets`, `verify-mermaid`, doc-typecheck (`ts` type-equiv/public-api manifest).
- Package READMEs with Model Experience sections (done incrementally per package, verified here).

### Scope — Out
- New product features.

### Deliverables
- The full documentation site with every page, catalog, and the Chinese mirror.

### Exit Criteria
- `pnpm run doc-sync` green (all doc gates).
- `pnpm run website:build` + `docs:build` green (dead-link check included).
- `pnpm run test:docs` (doc-quick aggregate) green.

---

## 23. Phase P20 — CI/CD and Release

### Goal
Ship the pipeline: CI matrix, failover, release automation, and the native `landlock-run` binary family.

### Scope — In
- `.github/workflows/` mirroring the current set: `ci.yml` (9 PR jobs + `all-checks-passed` verdict), `ci-master.yml` (serial standbys, wine-apt-cache seeder, benchmarks), `release.yml`, `release-vendor.yml`, `release-vendor-publish.yml`, `landlock-run.yml`, `landlock-run-release.yml`, `e2e.yml`, `pi-ai-provider-e2e.yml`, `docs-pages.yml`, `sandbox.yml`, `issue-lifecycle.yml`, `issue-policy.yml`, `expected-filenames.yml`, `build-preview-cloudflare.yml`.
- Scripts: `scripts/run-gates.ts` aggregates, `prepare-ci-bubblewrap.sh`, `wine-windows-gates.sh`, `check-expected-filenames.sh`, release scripts (`release/bump.ts`, `release/verify.ts`, `release/pack.ts`, `release/publish.ts`), `publint-all.ts`, `verify-built-package-invariants.mjs`.
- Native: `native/landlock-run` C launcher (~300 lines C11 over raw kernel UAPI, musl static), entry package (`launcherPath`/`probe`/`grantArgs`), linux-x64/arm64 prebuilds, pack-time gates (`verify-launcher-binary.mjs`, `verify-entry-lib.mjs`, `verify-packed-install.mjs`), `npm pack` (never `pnpm pack`) for platform tarballs, GitHub matrix from `prebuilds.json`.
- `experimental/*` handled as excluded from official artifacts (agent-team, webworker runtime/packer, inspector, etc.).
- Engine-floor compatibility: `check:node-compat` across 22.19 / 24 / 26.

### Scope — Out
- Post-release product features.

### Deliverables
- A green CI run equivalent to the current matrix on a real pull request, plus a dry-run release pack for dsh and vendor families and the landlock-run family.

### Exit Criteria
- `pnpm run check:all` green.
- A staging PR passes `all-checks-passed` (Linux + Windows + Wine + node-compat).
- `pnpm run release:pack` produces valid tarballs; `verify-packed-install` passes for landlock-run.

---

## 24. Dependency and Risk Notes

1. **P12 (Typert) gates P13–P16**: the Client cannot call Host methods until the generated contract exists. If P12 is delayed, keep P13's HTTP surface with stubbed controllers and re-expose through Remote later.
2. **P8 and P9 interlock**: tool-result spill lands in P9, so P8 tools initially keep inline results; revisit P8 consumers when `spill` exists.
3. **P11's base bundle expects P4–P10 rows**: do not attempt the bundle before the seams exist; composition is a thin layer over a complete tree.
4. **Platform divergence**: bash suites are Linux/macOS-only; pwsh suites run on Windows (`vitest.config.ts` enforces the split). Do not "fix" a suite by deleting it from the platform lists.
5. **Coverage partitions** weight by `.coverage-times.json`; an empty history falls back to file-count split, so a cold run is slower but valid.
6. **Sources vs built artifacts**: every phase's gates run against `src`; only P17+ gates and publint/release consume built `lib/`.
7. **Ordering of generator `verify-*` gates**: generated catalogs are evidence for CI freshness, not build inputs; they never block `pnpm run build`.
8. **Experimental scope**: P20 excludes `experimental/*` from official artifacts; if rebuilt at all, keep behind `build:official` boundaries and the `verify-application-entrypoints` gate.