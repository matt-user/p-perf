# Pinocchio CU Patterns

Short technique cards. For full before/after listings see [examples.md](examples.md). Sources: [references.md](references.md).

## Lazy entrypoint + no allocator

Prefer lazy entrypoint and forbid heap when the program is allocation-free:

```rust
#![no_std]
use pinocchio::{
    entrypoint::InstructionContext, lazy_program_entrypoint, log,
    no_allocator, nostd_panic_handler, ProgramResult,
};

lazy_program_entrypoint!(process_instruction);
no_allocator!();
nostd_panic_handler!();

fn process_instruction(context: InstructionContext) -> ProgramResult {
    // ...
    Ok(())
}
```

## `perf` feature flag

Gate logs and optional safety checks so production builds stay lean:

```toml
[features]
default = ["perf"]
perf = []
```

```rust
#[cfg(not(feature = "perf"))]
sol_log("Create Class");

#[cfg(not(feature = "perf"))]
if name.len() > MAX_NAME_LEN {
    return Err(ProgramError::InvalidArgument);
}
```

Build with checks: `cargo build-sbf --no-default-features` (or project-equivalent).

## Zero-copy account data

Access account bytes in place with `#[repr(C)]` + bytemuck (or equivalent):

```rust
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
pub struct Token {
    pub owner: [u8; 32],
    pub mint: [u8; 32],
    pub balance: u64,
}

let token: &Token = bytemuck::from_bytes(data);
```

## Fixed-size fields first

Put fixed-width fields before variable-length data for cheaper deserialize and RPC memcmp filters:

```rust
// Prefer
pub struct Record {
    owner: Pubkey, // offset 0
    name: String,
}
```

Avoid dynamically sized data on hot accounts when a few extra rent lamports are cheaper than repeated CU.

## Pubkey in instruction data

Do not include an account solely to read its pubkey. Pass the key in ix data instead.

## PDA: skip recompute for signers

When a PDA signs a CPI, the runtime already validates seeds — do not re-`find_program_address` / `create_program_address` just to assert equality.

## PDA: SHA-256 for existing addresses

For already-created PDAs (hashmap-style accounts), hash seeds directly and skip the off-curve check inside `create_program_address` (~100 CU with `solana_nostd_sha256` vs ~120 CU with `solana_program`):

```rust
use solana_nostd_sha256::hashv;
const PDA_MARKER: &[u8; 21] = b"ProgramDerivedAddress";

let bump = data[8];
let pda = hashv(&[
    signer.key().as_ref(),
    &[bump],
    ID.as_ref(),
    PDA_MARKER,
]);
assert_eq!(&pda, vault.key().as_ref());
```

## Superfluous checks

Skip pre-checks the callee will enforce (e.g. token balance before transfer). Prefer failing at the CPI. Gate early-fail debug checks behind `not(feature = "perf")` when useful for `skipPreflight: true` flows.

## Associated token accounts

Do not create ATAs or use `init-if-needed` inside hot instructions. Require ATAs created externally; verify by deriving the expected address:

```rust
let (associated_token_account, _) = find_program_address(
    &[
        self.accounts.owner.key(),
        self.accounts.token_program.key(),
        self.accounts.mint.key(),
    ],
    &pinocchio_associated_token_account::ID,
);
```

## Bitflags in one byte

Pack up to 8 booleans in a `u8`:

```rust
const FLAG_ACTIVE: u8 = 1 << 0;
const FLAG_VERIFIED: u8 = 1 << 1;

flags |= FLAG_ACTIVE;
if flags & FLAG_ACTIVE != 0 { /* ... */ }
flags &= !FLAG_ACTIVE;
```

## Zero allocation (refs over owned)

SVM already loads inputs. Prefer borrowed references over heap copies:

```rust
// Prefer
struct AccountView<'a> {
    key: &'a Pubkey,
    data: &'a [u8],
}
```

Limits: ~4KB stack per frame, ~32KB heap per execution. Use Pinocchio `no_allocator!()` when the program must never allocate; otherwise `default_allocator!()`.

## Libcalls and JIT intrinsics

Mental model: Rust op → LLVM libcall → (optional strong override in your crate) → optional `CALL_IMM` JIT intrinsic (native x86 in the SVM, no syscall exit). Same mechanics apply to any Pinocchio `no_std` program — no Pinocchio-specific API.

**Do not invent syscalls or IMM hashes.** Claim mainnet support for `sol_multi3` / similar only when the user’s toolchain and SVM are known to expose them.

### Libcall override (available today)

`compiler-builtins` defines libcalls as weak symbols. A strong local `extern "C"` definition wins at link time and controls lowering without forking the compiler.

Recipe: ship a custom `memcmp` (wide 64-bit loads; for large `n`, threshold to the existing `sol_memcmp_` syscall). Constant sizes inline/unroll; the branch is often compile-time eliminated. Suggest this when hot `[u8; N]` equality or similar memops show up in CU profiles. See examples 10a and Blueshift memops research.

### Explicit JIT wrapper (runtime-gated)

If the SVM recognizes a reserved `CALL_IMM` imm (Murmur3 of the intrinsic name), expose it as a thin wrapper — **only when that runtime is confirmed**:

```rust
#[inline(always)]
pub fn u64_wide_mul(a: u64, b: u64) -> u128 {
    let mut result = core::mem::MaybeUninit::<u128>::uninit();
    // murmur3("sol_u64_wide_mul") — prototype IMM from Blueshift research; not a public API guarantee
    let sol_u64_wide_mul: unsafe extern "C" fn(*mut u128, u64, u64) -> u64 =
        unsafe { core::mem::transmute(0x61BAE2E8usize) };
    unsafe {
        sol_u64_wide_mul(result.as_mut_ptr(), a, b);
        result.assume_init()
    }
}
```

Do not auto-apply this pattern or invent new hashes.

### `__multi3` → `sol_multi3` (research / runtime-gated)

Plain `u128` mul on BPF often expands to slow LLVM soft-mul (`__multi3`). Blueshift prototypes overriding `__multi3` to call a `sol_multi3` JIT intrinsic (~4× fewer CU / ~2× wall-clock in their bench — **prototype numbers, not mainnet guarantees**). Call sites stay `a * b`; the override is transparent.

**Agent behavior:** flag hot `u128` mul paths (P1/P2 awareness). Do not rewrite call sites or ship `__multi3` overrides unless the deployed toolchain/SVM exposes `sol_multi3`.
