# CU Anti-Patterns

Strip or avoid these unless the user explicitly needs them for safety/UX.

## Framework / memory

- [ ] Using heavy `solana_program` patterns (heap allocations, verbose account macros) when Pinocchio fits
- [ ] Owned `Pubkey` / `Vec<u8>` copies of data the SVM already loaded
- [ ] Missing `no_allocator!()` on programs that never need heap (or accidental alloc with allocator disabled)
- [ ] Full deserialize when a zero-copy view (`#[repr(C)]` + bytemuck/slice) suffices

## Accounts & data layout

- [ ] Variable-length fields before fixed-size fields
- [ ] Dynamically sized data on hot, frequently accessed accounts (prefer fixed size + a bit more rent)
- [ ] Accounts included only to obtain a pubkey (pass key in instruction data)
- [ ] Requiring ATA creation or `init-if-needed` inside hot swap/router-style instructions

## PDA

- [ ] Recomputing `find_program_address` / `create_program_address` for PDA signers the CPI already validates
- [ ] Using `create_program_address` on every access for an already-created PDA (prefer direct SHA-256 + known bump)

## Checks & logging

- [ ] Pre-CPI token balance / owner checks the token program will enforce anyway
- [ ] Unconditional `sol_log` / instruction-name logging in production (`perf`) builds
- [ ] Expensive “fail early” validates as default when the instruction is mostly used with `skipPreflight: true` — put them behind `not(feature = "perf")` instead of deleting thoughtlessly if debug UX matters

## On-chain state

- [ ] Multiple `bool` fields instead of a bitflag `u8` when packing matters
- [ ] Hot `u128` arithmetic without awareness of BPF soft-mul cost (see patterns; do not invent syscalls)

## Escalation (out of default scope)

- [ ] Jumping to sBPF assembly for marginal CU wins without an explicit user request
