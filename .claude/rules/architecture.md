---
paths:
  - "packages/core/src/**"
---

# Architecture Rules for @atlax/core

## Hexagonal Layer Dependency Rules

The four layers enforce a strict import direction. Violations are bugs.

- `domain/` imports NOTHING from `application/`, `infrastructure/`, or
  `presentation/` — zero external imports, only Node.js built-ins if
  absolutely necessary
- `application/` imports ONLY from `domain/` — never from `infrastructure/`
  or `presentation/`
- `infrastructure/` imports from `domain/` and `application/` only — never
  from `presentation/`
- `presentation/` imports from `application/` only — never from `domain/`
  or `infrastructure/` directly
- Circular imports between any layers are always a violation

## Bounded Context Rules

The `domain/` directory is split into seven bounded contexts. Cross-context
references use IDs (strings) only — never object references across context
boundaries.

Valid contexts: `identity`, `agent`, `vault`, `messaging`, `llm`, `ui`,
`marketplace`, `shared`

- `agent/` may reference `identity/` types via string `sessionId` only
- `messaging/` references `agent/` via string fingerprint only
- `vault/` references `agent/` via string `agentName` only
- `marketplace/` references `agent/` via string `agentName` only

## Port and Adapter Naming

- Interfaces (ports) in `domain/` are prefixed with `I`:
  `IAgentRepository`, `ISandbox`, `ICryptoAdapter`, `IAuditLog`
- Implementations (adapters) in `infrastructure/` are suffixed with the
  technology:
  `WorkerSandboxAdapter`, `SQLiteEpisodicRepository`, `AnthropicLLMAdapter`
- Application-level ports in `application/ports/` are also prefixed with `I`:
  `IEmbeddingService`, `IZionClient`

## Use Case Rules

- One use case per `spec.yaml` behavior — never combine two behaviors in one
  use case file
- Use case filename mirrors the behavior key from `spec.yaml`:
  `behaviors.agent_registration` -> `agentRegistrationUseCase.ts`
- Use case function/class receives a typed command DTO and returns a typed
  result DTO
- Failure paths from `spec.yaml behaviors.*.on_failure` become typed error
  classes exported from the use case file
- No `new ConcreteAdapterClass()` inside use cases — inject via constructor
  parameters or a dependency injection container

## SOLID Constraints

- **S:** One file per use case, one file per aggregate root, no mixed-concern
  utility files
- **O:** Adding a new behavior means adding a new use case file — never
  modifying an existing use case
- **L:** Every `IRepository` adapter returns the same types as the port
  declares — no nulls where the port promises a value
- **I:** Prefer focused ports over generic ones — `IAgentReader` and
  `IAgentWriter` are acceptable if read and write contexts diverge
- **D:** Domain defines ports; infrastructure implements them; domain never
  imports infrastructure

## Domain Invariant Enforcement

Invariants from `spec.yaml invariants.*` are enforced inside the aggregate
root, not in use cases or adapters.

- Agent state machine transitions are validated inside `Agent.ts` —
  an invalid transition throws a `DomainError` before any infrastructure is
  called
- Permission checks via `Bastion` run on every operation inside the
  relevant use case, calling the `IPermissionChecker` port
- Circuit breaker logic (3 failures in 5 minutes) lives in the `Agent`
  aggregate

## Crypto Rules

- Never write inline cryptographic operations — always call the adapters in
  `infrastructure/crypto/`
- The Vault encryption key must never be serialized to disk — keep it in
  a `CryptoKey` or `Uint8Array` held only in memory
- Ed25519 key pairs are generated per agent instance at registration time —
  never reused across agents
