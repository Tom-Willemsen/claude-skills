---
name: rust
description: Rust programming guidelines. Use the skill to understand local coding conventions when you are working with a Rust codebase
---

Rust is used in IBEX for newer processes where performance is important. IBEX uses Rust in two ways: standalone binaries, and Python extensions.

# Application styles

## Standalone binaries

These will use the typical rust tooling:
- Build: `cargo build`
- Test: `cargo test`
- Benchmarks: `cargo bench`
- Documentation: `cargo doc --no-deps`
- Formatting: `cargo fmt`
- Linting: `cargo clippy`

## Python extension modules

Prefer PyO3 and Maturin.

# Memory safety

In production code:
- Use memory-safe constructs whenever possible
- Unsafe code must have a `SAFETY:` comment describing safety preconditions
- Prefer `.get()` over raw array indexing
- Do not use `.unwrap()`
- `expect()` or `panic()` are allowable for cases where an application genuinely cannot continue (for example, an invalid configuration at startup)

These restrictions are relaxed in test-only code.
