---
paths:
  - "packages/**/*.test.ts"
  - "packages/**/*.test.tsx"
  - "packages/**/__tests__/**"
---

# Testing Rules

## Test Framework

- All TypeScript packages use Vitest — never Jest
- Test files are co-located with source or placed in `src/__tests__/`
- Test file naming: `<subject>.test.ts` (not `.spec.ts`)
- Every package has a `vitest.config.ts`; the root has a `vitest.workspace.ts`
  that discovers all package configs

## Test Organization

Tests map directly to the spec.yaml structure:

- One test file per aggregate root in `domain/`:
  `Agent.test.ts`, `AXPMessage.test.ts`, `Memory.test.ts`
- One test file per use case in `application/`:
  `agentRegistrationUseCase.test.ts`, `vaultUnlockUseCase.test.ts`
- Integration tests in `src/__tests__/integration/` — test a use case wired
  to in-memory adapters (no real I/O)
- Infrastructure adapter tests in `src/__tests__/adapters/` — test only the
  adapter in isolation with a real or stubbed external dependency

## Spec Coverage

Every `derived_tests` entry in `spec.yaml` must have a corresponding Vitest
test. Use the Given/When/Then structure from the spec as the test description:

```typescript
it('given no AuthSession, when invalid credentials, then no session created', async () => {
  // arrange
  // act
  // assert
})
```

The `given/when/then` naming in test descriptions must match the
`spec.yaml derived_tests` language exactly — this makes coverage audits
mechanical.

## Test Doubles

- Domain ports (`IAgentRepository`, `ISandbox`, `ICryptoAdapter`, etc.) are
  always mocked with hand-written in-memory implementations, not with
  `vi.mock()` auto-mocking
- Place shared test doubles in `packages/core/src/__tests__/doubles/`:
  `InMemoryAgentRepository.ts`, `InMemorySandbox.ts`, `NullAuditLog.ts`
- Use `vi.spyOn()` only for observing side effects (e.g. checking that
  `auditLog.append()` was called), not for replacing full implementations
- Never mock the domain layer itself — test aggregates and value objects
  directly with real instances

## Fixture Patterns

- Complex test fixtures (multi-agent setups, pre-populated Vault states) go
  in `src/__tests__/fixtures/` as factory functions:
  `createTestAgent(overrides?)`, `createTestSession(overrides?)`
- Factory functions accept `Partial<Entity>` overrides and fill defaults
  from the spec's attribute constraints
- Never construct test entities with `{}` — always use factories to ensure
  invariant-safe defaults

## Coverage

- Minimum line coverage: 80% for `domain/` and `application/` layers
- `infrastructure/` adapters have lower priority — focus coverage on behavior
  correctness at the use case level
- Coverage report: `@vitest/coverage-v8`; run with `moon run :test -- --coverage`

## Vitest Configuration

- `globals: true` in every `vitest.config.ts` to avoid importing
  `describe`/`it`/`expect` manually
- `environment: 'node'` for all non-rendering packages
- `environment: 'happy-dom'` for `@atlax/neo/web` tests that render React
- Timeout per test: 5000ms (Vitest default); override per-test with
  `{ timeout: 10000 }` for async integration tests only

## What Not to Test

- Do not write tests for TypeScript types alone — types are checked by `tsc`
- Do not test private implementation details — test the public interface of
  aggregates and use cases
- Do not duplicate the spec invariant checks at every layer — test invariants
  once at the aggregate level where they are enforced
