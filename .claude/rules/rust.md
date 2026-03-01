---
paths:
  - "crates/**/*.rs"
  - "crates/**/Cargo.toml"
---

# Rust Rules (Phase 2+)

These rules apply when implementing `crates/atlax-core/`. Phase 0 and Phase 1
do not involve Rust — skip this file until Phase 2 tasks begin.

## Crate Structure

- `crates/atlax-core/src/lib.rs` is the WASM public API — only `pub` items
  here that are decorated with `#[wasm_bindgen]` are visible to TypeScript
- Internal modules (`cortex`, `axp`, `bastion`, `vault`, `synapse`) use
  `pub(crate)` visibility — never `pub` unless it must be a WASM export
- Each module maps to a bounded context from the TypeScript implementation:
  same invariants, same state machine transitions

## wasm-bindgen Exports

- All types crossing the WASM boundary must implement `wasm_bindgen::JsValue`
  conversion — use `#[wasm_bindgen]` on structs and `serde-wasm-bindgen` for
  complex types
- Async WASM exports use `#[wasm_bindgen(async)]` and return `Promise<T>` on
  the TypeScript side
- Never expose raw Rust pointers across the WASM boundary — use owned types or
  `JsValue`
- The generated TypeScript bindings live in
  `packages/core/src/infrastructure/wasm/atlax_core.d.ts` — do not hand-edit
  this file

## Compilation Targets

- Web target: `wasm32-unknown-unknown` with `wasm-bindgen` + `wasm-pack`
- Native target (Phase 4): `x86_64-unknown-linux-gnu` or platform-specific,
  with `napi-rs` bindings
- The moon task `moon run atlax-core:build:wasm` drives wasm-pack compilation

## Memory and Safety

- No `unsafe` blocks unless wrapping a C FFI call that has no safe alternative
  — document every `unsafe` block with a `SAFETY:` comment explaining the
  invariant that makes it sound
- The Vault encryption key (`[u8; 32]`) is held in a `Zeroize`-on-drop wrapper
  (`ZeroizeKey`) — never clone or log it
- AES-256-GCM encryption uses the `aes-gcm` crate; Argon2id uses the
  `argon2` crate; Ed25519 uses the `ed25519-dalek` crate
- MessagePack serialization for AXP messages uses `rmp-serde`

## Error Handling

- Use `thiserror` for domain error types in each module
- WASM-exported functions return `Result<T, JsValue>` — convert internal errors
  to `JsValue::from_str(err.to_string())` at the WASM boundary
- Never panic in WASM-exported code paths — panics in WASM produce opaque
  runtime errors in the browser

## Testing

- Rust unit tests use `#[cfg(test)]` modules in each source file
- WASM integration tests use `wasm-bindgen-test` with `#[wasm_bindgen_test]`
  and run via `wasm-pack test --node`
- The moon task `moon run atlax-core:test` runs both native unit tests
  (faster) and WASM tests

## Javy / QuickJS (agent.wasm compilation)

- `atlax build` compiles agent TypeScript via Javy (QuickJS inside WASM)
- The Javy compilation pipeline is in `packages/cli/src/commands/build.ts`
  — the CLI invokes `javy compile` as a child process
- The output `agent.wasm` is embedded with QuickJS — it does not use the
  `atlax-core` Rust crate directly; it communicates with the host via AXP
  message passing through WASM imports/exports defined in `crates/atlax-core/src/axp/`
