# Curated Examples

Before → after snippets for Pinocchio CU work. Keep edits surgical; match the target crate’s APIs.

**Sources:** [Dean Little / Laugharne transcript](https://github.com/Laugharne/solana_optimized_programs), [Blueshift Performance](https://learn.blueshift.gg/en/courses/pinocchio-for-dummies/performance), [u128 research](https://blueshift.gg/research/accelerating-u128-math-with-libcalls-and-jit-intrinsics).

---

## 1. Naive Pinocchio memo → lazy entrypoint

**Before** (~109 CU in the talk’s memo demo):

```rust
use pinocchio::{
    account_info::AccountInfo, log, program_entrypoint, pubkey::Pubkey, ProgramResult,
};

program_entrypoint!(process_instruction);

fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    unsafe {
        log::sol_log(core::str::from_utf8_unchecked(instruction_data));
    }
    Ok(())
}
```

**After** (~108 CU; heap forbidden):

```rust
#![no_std]
use pinocchio::{
    entrypoint::InstructionContext, lazy_program_entrypoint, log, no_allocator,
    nostd_panic_handler, ProgramResult,
};

lazy_program_entrypoint!(process_instruction);
no_allocator!();
nostd_panic_handler!();

fn process_instruction(context: InstructionContext) -> ProgramResult {
    unsafe {
        log::sol_log(core::str::from_utf8_unchecked(context.instruction_data()?));
    }
    Ok(())
}
```

---

## 2. Zero-copy token layout

**Before:** deserialize into owned structs / `Vec`s.

**After:**

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

---

## 3. Fixed fields before variable data

**Before:**

```rust
pub struct Record {
    name: String,
    owner: Pubkey,
}
```

**After:**

```rust
pub struct Record {
    owner: Pubkey,
    name: String,
}
```

---

## 4. Account-only-for-pubkey → ix data

**Before (Anchor-shaped anti-pattern):** extra account just to seed a PDA.

**After:** pass `owner: Pubkey` in instruction data; derive/check the vault from that key without deserializing a throwaway account.

---

## 5. Existing PDA: create_program_address → SHA-256

**Before:**

```rust
let (pda, bump) = Pubkey::try_find_program_address(&[signer.key.as_ref()], &crate::ID)
    .ok_or(ProgramError::InvalidSeeds)?;
assert!(pda.eq(vault.key));
```

**After** (PDA already created; skip off-curve check):

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

---

## 6. `perf` flag around logs and checks

**Cargo.toml:**

```toml
[features]
default = ["perf"]
perf = []
```

**Code:**

```rust
pub fn process(ctx: Context<'_>) -> ProgramResult {
    #[cfg(not(feature = "perf"))]
    sol_log("Create Class");
    Self::try_from(ctx)?.execute()
}

#[cfg(not(feature = "perf"))]
if name.len() > MAX_NAME_LEN {
    return Err(ProgramError::InvalidArgument);
}
```

---

## 7. ATA: derive, do not create in-ix

```rust
let (associated_token_account, _) = find_program_address(
    &[
        self.accounts.owner.key(),
        self.accounts.token_program.key(),
        self.accounts.mint.key(),
    ],
    &pinocchio_associated_token_account::ID,
);
// Compare to the passed account; do not init-if-needed here.
```

---

## 8. Bitflag pack / check / clear

```rust
const FLAG_ACTIVE: u8 = 1 << 0;
const FLAG_VERIFIED: u8 = 1 << 1;
const FLAG_PREMIUM: u8 = 1 << 2;

let mut flags = 0u8;
flags |= FLAG_ACTIVE | FLAG_VERIFIED;

if flags & FLAG_ACTIVE != 0 { /* active */ }
if (flags & (FLAG_ACTIVE | FLAG_PREMIUM)) == (FLAG_ACTIVE | FLAG_PREMIUM) { /* both */ }

flags &= !FLAG_ACTIVE;
flags ^= FLAG_PREMIUM; // toggle
```

---

## 9. Owned heap structs → borrowed refs

**Before:**

```rust
struct AccountInfoOwned {
    key: Pubkey,
    data: Vec<u8>,
}
```

**After:**

```rust
struct AccountInfoView<'a> {
    key: &'a Pubkey,
    data: &'a [u8],
}
```

---

## 10. u128 mul — soft expand vs intrinsic (awareness)

Hot path using plain Rust `u128` mul (may expand to dozens of BPF ops / `__multi3`):

```rust
fn calculate_invariant(a: u128, b: u128) -> u128 {
    a * b
}
```

Blueshift research prototypes a libcall override that maps `__multi3` → `sol_multi3` JIT intrinsic (~4x fewer CU in their bench). **Do not rewrite call sites unless the deployed toolchain/SVM exposes that intrinsic.** Prefer flagging the hot path and documenting the dependency.
