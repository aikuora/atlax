# Codebase Intelligence

<!-- Keep this file concise. It is read at every session start. -->
<!-- Structural sections: populated by planner after /plan -->
<!-- Active Context: updated by engineer after every task -->

## Project Type & Stack

Open-source MIT monorepo — TypeScript SDK framework for agent-first operating systems.

- **Monorepo orchestrator:** moonrepo (`moon`) — `.moon/` workspace config, `moon.yml` per package
- **Toolchain:** prototools — `.prototools` pins Node.js LTS and Rust stable
- **Package manager:** pnpm — `pnpm-workspace.yaml` lists `packages/*`
- **TypeScript packages:** ESM-first, dual ESM+CJS output via `tsup`
- **Testing:** Vitest — `vitest.config.ts` per package, shared `vitest.workspace.ts` at root
- **Linting/formatting:** ESLint (flat config, `eslint.config.ts`) + Prettier (`.prettierrc`)
- **Git hooks:** lefthook (`lefthook.yml`) — pre-commit lint/format, commit-msg commitlint
- **Versioning:** release-please — manifest-based multi-package release PRs
- **Web renderer:** React + Vite (in consumer apps); `@atlax/neo/web` uses `react-dom`
- **Mobile renderer:** React Native + Expo; `@atlax/neo/native` uses `react-native`
- **Rust crate (Phase 2+):** `crates/atlax-core/` — compiled to WASM via `wasm-bindgen`

## Key Directories

```text
atlax/
  .moon/                   — moonrepo workspace config (workspace.yml, toolchain.yml)
  .prototools              — pinned Node.js, pnpm, Rust versions
  pnpm-workspace.yaml      — pnpm workspace root listing packages/*
  lefthook.yml             — git hooks config
  commitlint.config.ts     — conventional commits rules
  eslint.config.ts         — shared ESLint flat config (all TS packages inherit)
  .prettierrc              — shared Prettier config
  vitest.workspace.ts      — root vitest workspace (runs all package tests)
  packages/
    core/                  — @atlax/core (main SDK)
      src/
        domain/            — aggregates, value objects, ports (NO infrastructure imports)
          identity/        — AuthSession aggregate
          agent/           — Agent, Intent, Permission aggregates
          vault/           — Memory aggregate
          messaging/       — AXPMessage, AuditEntry aggregates
          llm/             — LLMProvider aggregate
          ui/              — AXUISchema aggregate
          marketplace/     — AgentPackage aggregate
          shared/          — shared value objects (Fingerprint, Ed25519Key, UUIDv7)
        application/       — use cases, one file per spec.yaml behavior
          identity/        — UserAuthenticationUseCase
          agent/           — AgentRegistrationUseCase, AgentExecutionUseCase, etc.
          vault/           — VaultUnlockUseCase, MemoryReadUseCase, MemoryWriteUseCase, etc.
          messaging/       — AxpSendUseCase, AxpBroadcastUseCase, AxpDelegateUseCase, etc.
          llm/             — LlmRoutingUseCase, BudgetManagementUseCase
          ui/              — AxuiRenderUseCase, DashboardPrioritizationUseCase, VoiceInputUseCase
          marketplace/     — AgentInstallUseCase, AgentPublishUseCase, AgentDeprecateUseCase, etc.
        infrastructure/    — adapters implementing domain ports
          crypto/          — Ed25519, AES-256-GCM, Argon2id, SHA-256 adapters
          persistence/     — SQLite/OPFS (episodic), HNSW (semantic), KV (preferences)
          sandbox/         — WorkerSandboxAdapter (Phase 0-1), WasmSandboxAdapter (Phase 2+)
          llm/             — AnthropicAdapter, OpenAIAdapter, LocalModelAdapter
          zion/            — ZionClient HTTP stubs and AgentPackage types
          otel/            — OpenTelemetry metrics and traces setup
          mcp/             — MCP protocol adapter (used by @atlax/bridge)
        presentation/      — public SDK API surface
          decorators/      — @Agent, @Intent, @Hook, @Guard, @Tool
          functions/       — defineAgent(), defineConfig(), defineRouter(), defineDashboard()
          context/         — AgentContext, ctx.memory, ctx.axp, ctx.llm, ctx.ui, ctx.tools
      src/index.ts         — @atlax/core public exports
      vitest.config.ts
      moon.yml
      package.json
    neo/                   — @atlax/neo (AXUI renderer)
      src/
        shared/            — AXUI schema validation, node registry, theme tokens
        web/               — React + react-dom renderer (entry: src/web/index.ts)
        native/            — React Native renderer (entry: src/native/index.ts)
      package.json         — conditional exports: ./web and ./native
      moon.yml
    prompt-guard/          — @atlax/prompt-guard (standalone)
      src/
        detector/          — prompt injection detection logic
        sanitizer/         — response sanitization
      src/index.ts
      moon.yml
    dashboard/             — @atlax/dashboard (optional)
      src/
        layout/            — grid layout engine
        prioritization/    — rule evaluator
        widgets/           — pinned widget registry
      src/index.ts
      moon.yml
    bridge/                — @atlax/bridge (MCP adapter)
      src/
        mcp/               — MCP protocol client
        sandbox/           — sandboxed tool call wrapper
        guard/             — integration with @atlax/prompt-guard
      src/index.ts
      moon.yml
    testing/               — @atlax/testing (skeleton stub only)
      src/index.ts         — empty re-exports; implementation deferred
      moon.yml
    cli/                   — atlax CLI
      src/
        commands/          — dev.ts, build.ts, publish.ts, doctor.ts, agent.ts
        lib/               — shared CLI utilities
      src/cli.ts           — entry point, commander setup
      moon.yml
  crates/
    atlax-core/            — Rust crate (Phase 2+)
      src/
        lib.rs             — wasm-bindgen public API
        cortex/            — agent registry, intent routing
        axp/               — AXP bus and message types
        bastion/           — permission enforcement
        vault/             — encrypted memory
        synapse/           — LLM router
      Cargo.toml
  .aikuora/                — aikuora-dev artifacts (spec, plan, decisions)
  .github/
    workflows/             — CI (lint, test, build, release-please)
```

## Entry Points

- **SDK consumer (library):** `packages/core/src/index.ts` — exports `defineAgent`,
  `defineConfig`, `defineRouter`, `@Agent`, `@Intent`, `@Hook`, `@Guard`, `@Tool`
- **AXUI renderer (web):** `packages/neo/src/web/index.ts` — exports `NeoProvider`,
  `renderSchema`
- **AXUI renderer (native):** `packages/neo/src/native/index.ts` — exports
  `NeoProvider`, `renderSchema`
- **Prompt guard:** `packages/prompt-guard/src/index.ts` — exports `PromptGuard`,
  `detectInjection`, `sanitize`
- **Dashboard:** `packages/dashboard/src/index.ts` — exports `defineDashboard`,
  `DashboardProvider`
- **Bridge:** `packages/bridge/src/index.ts` — exports `BridgeClient`, `mcpTool`
- **Testing stub:** `packages/testing/src/index.ts` — empty (implementation deferred)
- **CLI:** `packages/cli/src/cli.ts` — commander root command, registers all subcommands
- **Rust WASM (Phase 2+):** `crates/atlax-core/src/lib.rs` — `wasm-bindgen` exports
- **Tests:** each package's `vitest.config.ts`; run all via root `vitest.workspace.ts`
- **Moon tasks:** `.moon/workspace.yml` defines root tasks; each `moon.yml` defines
  package tasks (`build`, `test`, `lint`, `format`)

## Code Patterns

*Engineer fills this section as patterns are established during Phase 1.*

## Conventions

- **Package names:** `@atlax/<name>` for published packages; `atlax` for the CLI
- **File names:** `PascalCase` for classes and aggregates (`Agent.ts`, `AXPMessage.ts`);
  `camelCase` for use cases (`agentRegistrationUseCase.ts`)
- **Exports:** named exports only — never `export default` in library code
- **Ports (interfaces):** prefixed with `I` — `IAgentRepository`, `ISandbox`,
  `ICryptoAdapter`
- **Use cases:** suffixed with `UseCase` — one file per `spec.yaml` behavior
- **Adapters:** suffixed with the technology — `WorkerSandboxAdapter`,
  `SQLiteEpisodicRepository`
- **Domain events:** in `<Entity>Events.ts` alongside the aggregate
- **Imports:** workspace packages via `@atlax/<name>` (pnpm workspace protocol);
  never relative `../../` imports across package boundaries
- **Env vars:** `UPPER_SNAKE_CASE`, prefixed by subsystem
  (`ATLAX_ZION_URL`, `ATLAX_LLM_ANTHROPIC_KEY`)
- **Commits:** conventional commits — `feat(core): ...`, `fix(neo): ...`,
  `chore(cli): ...`; scope is the package name
- **IDs:** UUID v7 for AXP messages and AuditEntry IDs (timestamp-ordered)
- **Crypto:** never write inline crypto — use adapters in
  `packages/core/src/infrastructure/crypto/` exclusively

## Active Context

**Current task:** (none — Phase 0 setup not yet started)

**Relevant files:** (none)

**Recent discoveries:** (none)
