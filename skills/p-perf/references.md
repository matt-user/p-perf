# References

Primary sources for this skill. Re-read when applying advanced or toolchain-specific advice.

## Talk transcript

- [Laugharne/solana_optimized_programs](https://github.com/Laugharne/solana_optimized_programs) — transcript of Dean Little (Blueshift), *Scale or Die at Accelerate 2025: Writing Optimized Solana Programs*. CU tips, Pinocchio vs Anchor memo path, zero-copy, PDA SHA-256, superfluous checks, assembly comparison.

## Course

- [Pinocchio for Dummies — Performance](https://learn.blueshift.gg/en/courses/pinocchio-for-dummies/performance) — superfluous checks, ATA derive-only, `perf` feature flags, bitflags, SVM memory map, zero-allocation / `no_allocator!()`.

## Research

- [Accelerating u128 Math with Libcalls and JIT Intrinsics](https://blueshift.gg/research/accelerating-u128-math-with-libcalls-and-jit-intrinsics) — Claire Fan (Blueshift), Jan 2026. Custom `__multi3` libcalls + `sol_multi3` JIT intrinsic vs LLVM soft expand. Treat as research/toolchain-dependent unless the runtime exposes the intrinsic.

## Related tooling (optional)

- [anza-xyz/pinocchio](https://github.com/anza-xyz/pinocchio)
- [Agent Skills specification](https://agentskills.io/specification)
- [skills.sh](https://skills.sh/) / [`npx skills`](https://www.npmjs.com/package/skills)
