---
name: p-perf
description: >-
  Audits and optimizes Pinocchio Solana programs for compute units (CU).
  Covers zero-copy, lazy entrypoint, no_allocator, perf flags, superfluous
  checks, PDA hashing, and bitflags. Use when the user explicitly asks for
  Pinocchio CU optimization, compute unit reduction, or names this skill.
license: MIT
metadata:
  author: p-perf
  version: "0.1.0"
---

# Pinocchio CU Optimization

## Activation

Activate **only** when the user explicitly asks for Pinocchio CU optimization, compute unit reduction, or names this skill (`p-perf`). Do not load for unrelated Solana or Rust work.

## Workflow

Copy and track:

```
CU opt progress:
- [ ] 1. Scope
- [ ] 2. Baseline
- [ ] 3. Audit
- [ ] 4. Report then fix
- [ ] 5. Verify
```

### 1. Scope

Identify Pinocchio crates (`pinocchio`, `lazy_program_entrypoint`, `no_allocator!`, etc.). Skip Anchor-only paths unless the user wants a migration. Stay in Pinocchio/Rust — recommend sBPF assembly only if the user asks.

### 2. Baseline

Note how CU is measured in-repo (mollusk, liteSVM, benches, CU asserts). If none exists, recommend adding a CU assert after changes; do not invent a full harness unless asked.

### 3. Audit (highest impact first)

1. Entrypoint: `lazy_program_entrypoint` + `no_allocator!` + `nostd_panic_handler!` when heap-free
2. Heap / copies: owned `Pubkey`/`Vec` → borrowed refs / slices; zero-copy over full deserialize
3. Account design: fixed-size fields first; avoid dynamic sizes on hot accounts; pubkey in ix data when the account is unused for signing/data
4. PDA: skip re-`find`/`create` when signer CPI already validates; for existing PDAs prefer SHA-256 over `create_program_address` when off-curve is already established
5. Superfluous checks: drop pre-CPI balance/owner checks the callee enforces; gate logs/extra validates behind `#[cfg(not(feature = "perf"))]`
6. ATAs: do not `init-if-needed` in hot paths; verify by deriving the expected address
7. Bitflags: pack booleans into `u8` bitfields on-chain
8. Libcalls / math: suggest `memcmp`-style libcall overrides when hot memops show up; flag expensive `u128` mul soft-expand paths and JIT-intrinsic research (`sol_multi3`, etc.) — apply intrinsic wrappers only when the user’s toolchain/SVM is known to support them

Read [patterns.md](patterns.md), [examples.md](examples.md), and [anti-patterns.md](anti-patterns.md) as needed.

### 4. Report then fix

Emit findings, then apply fixes in severity order unless the user scopes otherwise.

```markdown
## CU audit
- Target: [crate/ix]
- Findings (P0/P1/P2):
  - [P0] path:line — issue — CU rationale

## Changes applied
- ...

## Verification
- Build / CU delta: ...
```

Severity: **P0** = large CU or always-on hot path; **P1** = clear waste, moderate impact; **P2** = polish / conditional.

### 5. Verify

Rebuild. Run existing CU tests/benches if present. Summarize before/after when numbers exist.

## Resources

- [patterns.md](patterns.md) — technique cards + snippets
- [examples.md](examples.md) — before/after curated examples
- [anti-patterns.md](anti-patterns.md) — what to strip
- [references.md](references.md) — source links
