# Architecture Decision Records (ADR)

Owner: Planner + Engineer

---

## ADR-001: Monorepo Tool — moonrepo (moon)

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The Atlax SDK spans multiple TypeScript packages (`@atlax/core`,
  `@atlax/neo`, `@atlax/prompt-guard`, `@atlax/dashboard`, `@atlax/bridge`,
  `@atlax/testing`, `atlax` CLI) and one Rust crate (`atlax-core`). A monorepo
  orchestrator is needed to coordinate builds, tests, linting, and task
  dependencies across heterogeneous languages and runtimes.
- **Decision:** Use moonrepo (`moon`) with `.moon/` workspace config.
- **Alternatives Considered:**
  1. Turborepo — TypeScript-only task graph; poor Rust/Cargo integration; no
     toolchain management.
  2. Nx — heavyweight, requires plugin ecosystem for Rust; adds significant
     config overhead.
  3. Bazel — maximum power, but extreme configuration complexity that is
     disproportionate to team size and open-source contributor friction.
- **Consequences:**
  - Task definitions live in each package's `moon.yml`; the root `.moon/`
    holds workspace-level config.
  - Rust `cargo` tasks are first-class moon tasks via the `system` task type.
  - Moon handles caching, dependency ordering, and affected-package detection
    across all languages.
  - Contributors must install `proto` to get `moon` — one extra step, but
    prototools automates it.

---

## ADR-002: Toolchain Manager — prototools (proto)

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The repo uses Node.js (TypeScript packages, CLI) and Rust
  (Phase 2+ core engine). All contributors must pin the same toolchain
  versions to eliminate "works on my machine" issues.
- **Decision:** Use prototools (`proto`) with a `.prototools` file at the repo
  root pinning all runtime versions. All toolchain installs are driven by
  `proto install`.
- **Alternatives Considered:**
  1. `.nvmrc` + `rustup` — separate tools for each language; no unified
     install story; no moon integration.
  2. Devcontainer / Docker — maximally reproducible but heavy; not suitable
     for a public open-source framework with diverse contributors.
  3. asdf — similar to proto but no native moon integration.
- **Consequences:**
  - `.prototools` is the single source of truth for Node.js, pnpm, and Rust
    versions.
  - `proto install` bootstraps the full toolchain from a clean machine.
  - Moon reads `.prototools` automatically when executing tasks.

---

## ADR-003: TypeScript Package Manager — pnpm

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The monorepo has 7 TypeScript packages that share dependencies
  and cross-reference each other as workspace packages. Hoisting, symlinks,
  and workspace protocol support are required.
- **Decision:** Use pnpm with `pnpm-workspace.yaml` at the repo root. Workspace
  settings go in `pnpm-workspace.yaml` (not `.npmrc`) per
  <https://pnpm.io/settings>.
- **Alternatives Considered:**
  1. npm workspaces — no phantom-dependency prevention; slower installs.
  2. yarn (Berry, PnP) — PnP requires special editor setup; plug-and-play is
     incompatible with some native modules needed in Phase 2.
  3. Bun workspaces — fast, but Bun is mandated only for the CLI question;
     rejected as CLI runtime (Node.js chosen); mixing package managers in a
     mono-repo is unnecessary complexity.
- **Consequences:**
  - All workspace packages reference each other as `"@atlax/core": "workspace:*"`.
  - `pnpm-workspace.yaml` lists `packages: ["packages/*"]`.
  - `node_modules` uses a content-addressable store with symlinks; phantom
    deps are blocked by default.

---

## ADR-004: Versioning and Publishing — release-please

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The SDK will eventually publish multiple `@atlax/*` packages to
  npm independently. Version bumps must follow SemVer and be driven by
  conventional commit messages without manual changelog editing.
- **Decision:** Use release-please (Google) with a manifest-based multi-package
  config (`release-please-config.json` + `.release-please-manifest.json`).
  One release PR per package when its commits warrant it.
- **Alternatives Considered:**
  1. changesets — popular in the pnpm ecosystem but requires manual changeset
     file authoring per PR; adds contributor friction for an open-source project.
  2. semantic-release — powerful but requires bespoke plugin chains for
     multi-package repos; harder to configure with moonrepo.
  3. Manual versioning — not viable at 7+ packages with open-source contributions.
- **Consequences:**
  - Every merged PR that touches a package with conventional commits gets a
    release PR auto-generated.
  - The Rust crate (`atlax-core`) is excluded from release-please (versioned
    separately via `Cargo.toml` once Phase 2 begins).
  - CI must have `GITHUB_TOKEN` with write access to create release PRs.

---

## ADR-005: Git Hooks — lefthook

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** Commit message linting (`commitlint`) and pre-commit checks
  (lint, format) must run locally before code reaches CI to keep the feedback
  loop short.
- **Decision:** Use lefthook with `lefthook.yml` at the repo root. `pre-commit`
  runs `pnpm lint` + `pnpm format:check`; `commit-msg` runs `commitlint`.
- **Alternatives Considered:**
  1. husky — requires `prepare` script in `package.json`; `.husky/` directory;
     does not integrate as cleanly with moonrepo's task system.
  2. lint-staged — pairs with husky but is a second tool; lefthook handles
     staged file filtering natively.
  3. No local hooks — pushes all validation to CI; slower feedback.
- **Consequences:**
  - `lefthook install` must be run once after `pnpm install` (can be automated
    via moon's `syncProjects` or a postinstall script).
  - Hooks are committed to the repo and apply uniformly to all contributors.

---

## ADR-006: Commit Linting — commitlint with @commitlint/config-conventional

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** release-please (ADR-004) parses conventional commit messages to
  determine version bumps and generate changelogs. Commit message discipline
  must be enforced mechanically.
- **Decision:** Use `commitlint` with `@commitlint/config-conventional` and a
  `commitlint.config.ts` at the repo root. Enforced via lefthook `commit-msg`
  hook (ADR-005).
- **Alternatives Considered:**
  1. Custom regex in a shell script — fragile, harder to extend.
  2. No enforcement — relies on contributor discipline; breaks release-please
     automation.
- **Consequences:**
  - Commit format: `type(scope): subject` where scope is the package name
    (e.g. `feat(core): add intent router`).
  - Breaking changes use `BREAKING CHANGE:` footer or `!` suffix.

---

## ADR-007: TypeScript Linting and Formatting — ESLint + Prettier

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** Consistent code style across 7 TypeScript packages maintained
  by open-source contributors requires automated formatting and linting.
- **Decision:** Use ESLint (with `@typescript-eslint/eslint-plugin` and
  `eslint-config-prettier`) for linting and Prettier for formatting. Root-level
  `eslint.config.ts` (flat config) and `.prettierrc` apply to all packages.
  Each package can extend with local overrides.
- **Alternatives Considered:**
  1. Biome — fast all-in-one alternative; ecosystem is maturing but not yet at
     ESLint plugin parity for TypeScript-specific rules needed here.
  2. ESLint only (no Prettier) — ESLint's formatting rules conflict with
     TypeScript-eslint and are deprecated in favor of Prettier.
- **Consequences:**
  - `eslint.config.ts` uses the flat config API (ESLint v9+).
  - Prettier is the single source of truth for formatting; ESLint rules that
    conflict with Prettier are disabled.
  - Moon tasks: `moon run :lint` and `moon run :format` call ESLint and Prettier
    respectively across all packages.

---

## ADR-008: TypeScript Testing — Vitest

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** All TypeScript packages need unit and integration tests. The
  framework must support ESM natively, be fast in watch mode, and integrate
  with moonrepo's task system without a complex Jest config.
- **Decision:** Use Vitest across all TypeScript packages. Each package has its
  own `vitest.config.ts`. The root provides a shared `vitest.workspace.ts` for
  running all package tests from the repo root.
- **Alternatives Considered:**
  1. Jest — requires `ts-jest` or `babel-jest` transformer; slower startup;
     ESM support is still partial.
  2. node:test — built-in, no dependencies, but lacks watch mode, coverage
     integration, and the snapshot/mocking ergonomics needed for complex
     agent lifecycle tests.
- **Consequences:**
  - Tests live in `src/__tests__/` within each package, or alongside source
    files as `*.test.ts`.
  - Coverage via `@vitest/coverage-v8`.
  - Moon task `test` in each package calls `vitest run`; CI calls
    `moon run :test` for all.

---

## ADR-009: Monorepo Package Split

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The Atlax SDK spec defines distinct subsystems (Cortex, Bastion,
  Vault, Synapse, Neo, Bridge) and standalone deliverables (`@atlax/prompt-guard`,
  `@atlax/dashboard`, CLI). These have different dependency profiles, release
  cadences, and consumer audiences. The split determines install size and
  tree-shaking boundaries.
- **Decision:** The monorepo contains the following packages:

  | Package | npm name | Description |
  | --- | --- | --- |
  | `packages/core` | `@atlax/core` | Main SDK — Agent, Intent, Memory, AXP, Bastion, Synapse, Cortex |
  | `packages/neo` | `@atlax/neo` | AXUI renderer — `/web` (React) and `/native` (React Native) |
  | `packages/prompt-guard` | `@atlax/prompt-guard` | Standalone prompt injection guard |
  | `packages/dashboard` | `@atlax/dashboard` | Optional Ambient Dashboard |
  | `packages/bridge` | `@atlax/bridge` | MCP adapter — sandboxes tool calls |
  | `packages/testing` | `@atlax/testing` | Skeleton stub only (Phase 0); implementation deferred |
  | `packages/cli` | `atlax` | CLI tool (Node.js, esbuild bundle) |
  | `crates/atlax-core` | `atlax-core` (crate) | Rust core engine (Phase 2+) |

- **Alternatives Considered:**
  1. All in one package — no tree-shaking; consumers who only want
     `@atlax/prompt-guard` pull in React, React Native, and the full core.
  2. Finer split (separate `@atlax/cortex`, `@atlax/vault`, etc.) — excessive
     for Phase 0–1; premature; the subsystems are tightly coupled in TS phase.
- **Consequences:**
  - `@atlax/core` is the primary install for application developers.
  - `@atlax/neo`, `@atlax/dashboard`, and `@atlax/bridge` are opt-in.
  - `@atlax/prompt-guard` is independently usable without the full core.

---

## ADR-010: Zion Marketplace — Client-Only in This Repo

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The spec defines a Zion marketplace registry for publishing and
  installing agent packages. Zion can be hosted or self-hosted. The question is
  whether the server code belongs in this repo.
- **Decision:** Zion server code is in a separate repository. This monorepo
  contains only Zion client types (`ZionClient`, `AgentPackage` types) and HTTP
  contract stubs in `packages/core/src/infrastructure/zion/`. No server
  implementation here.
- **Alternatives Considered:**
  1. Include Zion server in this repo as a `packages/zion-registry` package —
     bloats the SDK repo with backend infrastructure; different release cadence;
     server-side tech choices pollute the SDK's dependency graph.
- **Consequences:**
  - `atlax publish` CLI calls the Zion HTTP API via the client stubs.
  - The Zion API contract is defined as TypeScript types and OpenAPI comments
    in `packages/core/src/infrastructure/zion/`.
  - Phase 3 tasks implement the Zion client, not the server.

---

## ADR-011: @atlax/neo — Single Package with Conditional Exports

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The Neo renderer must produce React components on web and React
  Native components on mobile. These have incompatible peer dependencies
  (`react-dom` vs `react-native`). The question is whether to split into two
  packages or use one package with conditional exports.
- **Decision:** One `@atlax/neo` package with conditional exports:
  `@atlax/neo/web` resolves to `dist/web/index.js` (React + react-dom),
  `@atlax/neo/native` resolves to `dist/native/index.js` (React Native).
  The `package.json` `exports` field uses the `react-native` condition for
  Metro and `default` for Vite/webpack.
- **Alternatives Considered:**
  1. Two packages (`@atlax/neo-web`, `@atlax/neo-native`) — doubles package
     management overhead; shared AXUI schema validation and node type registry
     must be duplicated or extracted to a third package.
- **Consequences:**
  - A shared `src/shared/` directory inside `packages/neo` holds AXUI
    validation, node type registry, and theme token resolution — used by both
    platform entry points.
  - `package.json` `exports` map:

    ```json
    {
      "./web": { "import": "./dist/web/index.js" },
      "./native": {
        "react-native": "./dist/native/index.js",
        "default": "./dist/native/index.js"
      }
    }
    ```

  - The `react-dom` and `react-native` dependencies are `peerDependencies`
    (not `dependencies`), so the consumer installs only what it needs.

---

## ADR-012: CLI Runtime — Node.js with esbuild Bundle

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The `atlax` CLI runs `dev`, `build`, `publish`, `doctor`, and
  other commands. It needs broad developer machine compatibility, access to the
  Node.js ecosystem (Vite, child processes, file system), and fast startup.
- **Decision:** Node.js as the CLI runtime. Development uses `tsx` for
  TypeScript execution without compilation. Production bundle uses `esbuild` to
  produce a single `dist/cli.js` with `#!/usr/bin/env node` shebang, published
  as the `bin` entry in `packages/cli/package.json`.
- **Alternatives Considered:**
  1. Bun — faster startup but narrower adoption; contributors on Windows or
     restricted CI environments may not have Bun available.
  2. Deno — different module system (`npm:` specifiers); breaks pnpm workspace
     imports without extra config.
- **Consequences:**
  - `tsx` is a `devDependency` of `packages/cli`; used only in `atlax dev`
    watch mode.
  - The production bundle is self-contained (esbuild bundles all deps except
    optional native modules).
  - The CLI is published to npm as the `atlax` package.

---

## ADR-013: Hexagonal Architecture for @atlax/core

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** `@atlax/core` implements 25 behaviors across 10 entities with
  complex cross-entity invariants, 4+ external integrations (LLM providers, MCP
  servers, Zion, cloud sync), and cryptographic security requirements. The
  domain logic (Bastion permission checks, circuit breaker, AXP signature
  verification) must be independently testable without real databases, LLM
  APIs, or WASM runtimes.
  This affects all 25 behaviors in `spec.yaml`.
- **Decision:** Apply Hexagonal Architecture (Ports and Adapters) inside
  `packages/core/src/`:

  ```text
  src/
    domain/       — aggregates, value objects, ports (interfaces only)
    application/  — use cases (one per spec.yaml behavior)
    infrastructure/ — adapters (crypto, SQLite, ONNX, LLM HTTP, Zion HTTP)
    presentation/ — SDK public API (defineAgent, @Agent, defineConfig, etc.)
  ```

- **Alternatives Considered:**
  1. Flat module structure — fast to start but collapses under 25 behaviors and
     10 entities; infrastructure details bleed into domain logic.
  2. Feature-folder (one folder per behavior) — better than flat but makes
     cross-cutting concerns (Bastion, AuditEntry) hard to locate.
- **Consequences:**
  - `domain/` has zero `import` statements from outside `domain/`.
  - Use cases in `application/` are each one file corresponding to exactly one
    `behaviors.*` entry in `spec.yaml`.
  - Infrastructure adapters are injected via constructor; no `new ConcreteClass()`
    inside use cases.
  - Test doubles (mocks) implement domain ports, enabling pure-TS unit tests
    without any I/O.

---

## ADR-014: DDD Bounded Contexts inside @atlax/core

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** `@atlax/core` implements behaviors spanning identity (auth,
  session), agent lifecycle, memory/vault, inter-agent communication (AXP),
  LLM routing, UI rendering, and marketplace. These domains change
  independently.
- **Decision:** Organize `packages/core/src/domain/` into six bounded contexts:

  | Context | Entities | Key behaviors |
  | --- | --- | --- |
  | `identity` | AuthSession | user_authentication |
  | `agent` | Agent, Intent, Permission | agent_registration, agent_execution, permission_approval, permission_check, permission_revocation, reputation_update |
  | `vault` | Memory | vault_unlock, memory_read, memory_write, cloud_sync |
  | `messaging` | AXPMessage, AuditEntry | axp_send, axp_broadcast, axp_delegate, audit_log_entry |
  | `llm` | LLMProvider | llm_routing, budget_management |
  | `ui` | AXUISchema | axui_render, dashboard_prioritization, voice_input |
  | `marketplace` | AgentPackage | agent_install_from_marketplace, agent_publish, agent_deprecate, mcp_tool_invocation |

- **Alternatives Considered:**
  1. Single `domain/` flat — all 10 aggregates at root level; `Agent.ts` and
     `AuditEntry.ts` are peers with no grouping; cross-context relationships
     become implicit.
- **Consequences:**
  - Each context folder contains its aggregate roots, value objects, domain
    events, and repository ports.
  - Cross-context references use IDs only (e.g. `AuditEntry.agent` is a
    string fingerprint, not an `Agent` reference).

---

## ADR-015: Phase 0–1 Sandbox — Web Workers (TypeScript)

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** The spec mandates agent execution isolation. Phase 2+ uses WASM
  (Javy + QuickJS + Wasmtime). For Phase 0–1 in TypeScript, a sandboxing
  approach must be chosen that: (a) provides meaningful isolation without WASM
  compilation, (b) supports the 30-second timeout kill, and (c) enables hot
  reload in `atlax dev`.
- **Decision:** Use Web Workers for agent isolation in Phase 0–1 on web. Mobile
  (Phase 0–1) uses JSC/Hermes isolates via React Native's built-in threading.
  The 30-second timeout is enforced by `worker.terminate()` after a
  `setTimeout`.
  This affects `behaviors.agent_execution` in `spec.yaml`.
- **Alternatives Considered:**
  1. `vm` module (Node.js) — server-side only; not available in browsers or
     React Native.
  2. iframe sandboxing — browser-only; no memory isolation; not applicable to
     mobile.
- **Consequences:**
  - In Phase 0–1, locally-defined (unsigned) agents run in Web Workers.
  - Marketplace agents (with `identity.signedBy`) use the same Web Worker path
    in Phase 0–1, switching to WASM sandbox in Phase 2+.
  - The `WorkerSandboxAdapter` in `packages/core/src/infrastructure/sandbox/`
    wraps the Web Worker API and implements the `ISandbox` port.

---

## ADR-016: Rust Crate Structure (Phase 2)

- **Date:** 2026-03-01
- **Status:** Accepted
- **Context:** Phase 2 migrates the Atlax core engine to Rust compiled to WASM.
  The five subsystems (Cortex, AXP bus, Bastion, Vault, Synapse) need a crate
  organization that maps to the TypeScript bounded contexts and produces a
  single WASM artifact usable from TypeScript via `wasm-bindgen`.
- **Decision:** `crates/atlax-core/` is a Cargo workspace member with a single
  `src/lib.rs` that re-exports five internal modules: `cortex`, `axp`,
  `bastion`, `vault`, `synapse`. `wasm-bindgen` attributes on public functions
  generate the TypeScript bindings. A separate `build.rs` compiles to
  `atlax_core_bg.wasm` and `atlax_core.js`.
- **Alternatives Considered:**
  1. Five separate Rust crates (one per subsystem) — fine-grained, but WASM
     binary size suffers from five separate `wasm-bindgen` outputs; harder to
     share types across crates.
  2. napi-rs for web — napi-rs targets Node.js native addons, not WASM; napi-rs
     is used in Phase 4 for native desktop/mobile (Tauri, napi addon), not
     for web.
- **Consequences:**
  - The TypeScript packages import the WASM artifact from a generated
    `@atlax/core-wasm` internal package in Phase 2.
  - `wasm-pack` or a custom `moon` task drives the WASM compilation.
  - `Wasmtime` is embedded in the Rust binary for native target (Phase 4);
    browser uses the native WebAssembly API.

---
