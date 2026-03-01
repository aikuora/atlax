---
paths:
  - "packages/**/*.ts"
  - "packages/**/*.tsx"
---

# TypeScript Rules

## Module System

- All packages output ESM as the primary format; CJS is a secondary output
  for CommonJS consumers
- Use `.js` extensions in relative imports even for TypeScript source files
  (required for ESM compatibility): `import { Agent } from './Agent.js'`
- Never use `require()` in library source — only in CLI Node.js-specific code
  where ESM interop is not available

## Exports

- Named exports only — never `export default` in library code (`@atlax/*`
  packages)
- The CLI (`packages/cli`) may use default exports only for commander Command
  objects where the library requires it
- Every package exposes its public API exclusively through `src/index.ts` —
  consumers never deep-import into `src/domain/`, `src/application/`, or
  `src/infrastructure/`

## TypeScript Configuration

- `strict: true` is mandatory in every `tsconfig.json`
- `exactOptionalPropertyTypes: true` must be enabled
- `noUncheckedIndexedAccess: true` must be enabled
- Target: `ES2022` for all packages; `lib: ["ES2022", "DOM"]` for web packages,
  `lib: ["ES2022"]` for Node.js packages

## Type Safety

- Never use `any` — use `unknown` for untyped external data and narrow with
  type guards
- Never use non-null assertion (`!`) on values that can genuinely be undefined
  — use explicit null checks
- Zod is the validation library for all external inputs (agent manifests,
  AXUI schemas, LLM responses, config files)
- All spec.yaml entity `attributes` map to TypeScript types validated by Zod
  schemas at the boundary (infrastructure layer on inbound, presentation layer
  on outbound)

## Error Handling

- Use typed error classes extending `AtlaxError extends Error` for all domain
  errors — never throw raw strings
- Every use case function signature has an explicit return type (no inferred
  `any` returns)
- `Result<T, E>` pattern (using `neverthrow` or a lightweight equivalent) is
  preferred over thrown exceptions for expected failure paths from
  `spec.yaml behaviors.*.on_failure`
- Unexpected errors (programming errors, assertion failures) are thrown and
  propagate to the top-level error handler

## Package Exports Configuration

- Every package `package.json` must include `"type": "module"` and a complete
  `exports` map
- The `exports` map must include `"types"`, `"import"`, and `"require"` keys
  for all entry points
- `@atlax/neo` additionally includes `"./web"` and `"./native"` conditional
  exports (see ADR-011)

## ESLint Rules

- ESLint flat config is in `eslint.config.ts` at the repo root
- Packages extend the root config — do not duplicate rules in package-level
  config files
- `@typescript-eslint/no-explicit-any` is set to `error`
- `@typescript-eslint/explicit-function-return-type` is set to `error` for
  all `application/` and `domain/` files
- Import order is enforced: Node built-ins, then external packages,
  then workspace packages (`@atlax/*`), then relative imports

## Prettier

- Single `.prettierrc` at the repo root applies to all packages
- Print width: 100 characters
- Trailing commas: `all`
- Semicolons: `true`
- Single quotes: `true`
- Never run `prettier` manually — use `moon run :format` which calls
  `prettier --write`

## tsup Build Config

- Every publishable package uses `tsup` with `format: ['esm', 'cjs']`
- `dts: true` to generate declaration files
- `entry` points match the `exports` map in `package.json`
- Source maps are always included: `sourcemap: true`
