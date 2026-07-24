# p-perf — Pinocchio CU Optimization Skill

Agent skill that audits and optimizes [Pinocchio](https://github.com/anza-xyz/pinocchio) Solana programs for compute units (CU): zero-copy layouts, lazy entrypoints, `perf` flags, superfluous-check removal, PDA hashing, and related patterns.

Compatible with the [Agent Skills](https://agentskills.io/specification) standard and any harness that loads `SKILL.md` skills (Cursor, Claude Code, Codex, OpenCode, GitHub Copilot, Gemini CLI, and others supported by the skills CLI).

## Install

Install into **all detected agent harnesses**:

```bash
npx skills add matt-user/p-perf
```

Pin to one skill or specific agents:

```bash
npx skills add matt-user/p-perf --skill p-perf
npx skills add matt-user/p-perf -a cursor -a claude-code -a codex
```

## Invoke

Activate only when you explicitly ask for Pinocchio CU optimization, compute unit reduction, or name the skill (e.g. `/p-perf` where the harness supports slash commands).

The agent will examine Pinocchio code, report prioritized findings, apply fixes, then verify with any existing CU tests or benches.

## Attribution

Techniques are curated from:

- [Scale or Die — Writing Optimized Solana Programs (Dean Little / Blueshift)](https://github.com/Laugharne/solana_optimized_programs)
- [Pinocchio for Dummies — Performance (Blueshift)](https://learn.blueshift.gg/en/courses/pinocchio-for-dummies/performance)
- [Accelerating u128 Math with Libcalls and JIT Intrinsics (Blueshift Research)](https://blueshift.gg/research/accelerating-u128-math-with-libcalls-and-jit-intrinsics)

## License

MIT
