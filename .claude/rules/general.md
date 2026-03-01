# General Project Rules

## Commit Conventions

- Every commit must follow the Conventional Commits format:
  `type(scope): subject`
- Valid types: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `ci`
- Scope is the affected package name without the `@atlax/` prefix:
  `core`, `neo`, `prompt-guard`, `dashboard`, `bridge`, `testing`, `cli`
- Breaking changes: append `!` to the type or add `BREAKING CHANGE:` footer
- Examples: `feat(core): add intent router`, `fix(neo): resolve ttl timer leak`,
  `chore(cli): upgrade esbuild to 0.25`

## moonrepo Task Naming

- Every package must have these moon tasks defined in its `moon.yml`:
  `build`, `test`, `lint`, `format`, `typecheck`
- Run a single package task: `moon run core:test`
- Run a task across all packages: `moon run :test`
- Do not invoke `pnpm run` or `npx` directly inside moon tasks — use
  `moon run` or call the tool binary directly

## pnpm Workspace Imports

- Reference sibling packages with the workspace protocol:
  `"@atlax/core": "workspace:*"` in `package.json`
- Never use relative `../../` paths to import from another package
- Never use `npm:` or `yarn:` protocols — pnpm workspace only
- Install a dependency to a specific package:
  `pnpm add <dep> --filter @atlax/core`
- Install a dev dependency to the root:
  `pnpm add -D <dep> -w`

## File Organization

- One aggregate root per file in `domain/`
- One use case per file in `application/` — filename matches behavior name
  from `spec.yaml` (e.g. `agentRegistrationUseCase.ts`)
- No `utils.ts`, `helpers.ts`, or `common.ts` files — name by domain concept
- No barrel `index.ts` files inside `domain/` subdirectories — only at
  package root `src/index.ts`

## Environment Variables

- All env vars are `UPPER_SNAKE_CASE` prefixed by subsystem:
  `ATLAX_ZION_URL`, `ATLAX_LLM_ANTHROPIC_KEY`, `ATLAX_SYNC_ENDPOINT`
- Never hardcode secrets or API keys in source files
- Never commit `.env` files — only `.env.example` with placeholder values

## Lefthook Hooks

- `pre-commit`: runs `moon run :lint` and `moon run :format:check`
- `commit-msg`: runs `commitlint --edit`
- Do not bypass hooks with `--no-verify` except in documented CI override cases
