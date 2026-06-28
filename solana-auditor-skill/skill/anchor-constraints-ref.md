# Anchor Account Constraints Reference

A complete quick-reference for every `#[account(...)]` constraint available in the Anchor framework (v0.29–v0.30).

Use this file when auditing an Anchor program to verify that every account in a `Context` struct is properly constrained. Each entry covers the constraint's purpose, when to use it, what it validates automatically, and a usage example.

---

## Constraint Index

| Constraint | Category | Purpose |
|---|---|---|
| [`mut`](#mut) | Access | Marks account as writable |
| [`signer`](#signer) | Auth | Requires transaction signature |
| [`init`](#init) | Lifecycle | Initializes a new account |
| [`init_if_needed`](#init_if_needed) | Lifecycle | Init only if account doesn't exist |
| [`payer`](#payer) | Lifecycle | Designates who pays for account creation |
| [`space`](#space) | Lifecycle | Sets allocated byte size for new accounts |
| [`seeds`](#seeds) | PDA | Specifies PDA seed components |
| [`bump`](#bump) | PDA | Validates or captures the PDA bump byte |
| [`has_one`](#has_one) | Relationship | Asserts a field key matches another account's key |
| [`owner`](#owner) | Ownership | Verifies the account is owned by a specific program |
| [`address`](#address) | Identity | Asserts an exact public key match |
| [`constraint`](#constraint) | Custom | Arbitrary boolean expression with custom error |
| [`close`](#close) | Lifecycle | Closes the account and returns lamports |
| [`realloc`](#realloc) | Lifecycle | Resizes an existing account |
| [`realloc::payer`](#reallocpayer) | Lifecycle | Designates who pays/receives for realloc |
| [`realloc::zero`](#realloczero) | Lifecycle | Controls whether new bytes are zeroed |
| [`zero`](#zero) | Init | Initializes account without charging for space |
| [`executable`](#executable) | Validation | Requires the account to be a deployed program |

---

## `mut`

### Purpose
Marks the account as writable in the transaction. Anchor requires any account whose data or lamports will change to be marked `mut`.

### What It Does
- Sets the `is_writable` flag on the account info passed to the runtime.
- The program will panic at runtime if you attempt to write to an account not marked `mut`.

### When to Use
Every account whose data fields or lamport balance will be modified within the instruction handler.

### Example
```rust
#[derive(Accounts)]
pub struct UpdateVault<'info> {
    #[account(mut)]       // vault.balance will be modified
    pub vault: Account<'info, Vault>,

    #[account(mut)]       // lamports will leave this account
    pub payer: Signer<'info>,
}
```

### Security Notes
- Marking an account `mut` that is not actually modified wastes transaction weight but is not a security vulnerability.
- Failing to mark an account `mut` that is modified will cause a runtime error — the program is safe but non-functional.

---

## `signer`

### Purpose
Enforces that the account has signed the transaction. Equivalent to using `Signer<'info>` as the account type.

### What It Does
- Checks `account_info.is_signer == true` at constraint evaluation time.
- If the check fails, the instruction returns an error before the handler runs.

### When to Use
Any account that represents an authority, admin, or user who must authorize the action. **This is one of the most commonly omitted constraints and a top cause of Critical vulnerabilities.**

### Example
```rust
#[derive(Accounts)]
pub struct Withdraw<'info> {
    #[account(mut, has_one = authority)]
    pub vault: Account<'info, Vault>,

    // Both of these are equivalent — prefer the Signer<> type form
    pub authority: Signer<'info>,
    // #[account(signer)]
    // pub authority: AccountInfo<'info>,  // alternative if type must be AccountInfo
}
```

### Security Notes
- Using `Signer<'info>` as the type is idiomatic and preferred over `#[account(signer)]` on `AccountInfo`.
- A `constraint = authority.key() == vault.authority` check validates identity but does **not** validate that the account signed. Always combine key checks with `Signer<'info>`.

---

## `init`

### Purpose
Initializes a brand-new on-chain account. Creates the account via a CPI to the System Program, allocates space, assigns ownership to the executing program, and writes the 8-byte Anchor discriminator.

### What It Does
- Derives the account address (or PDA if `seeds` is also provided).
- Calls `system_program::create_account` under the hood.
- **Errors if the account already exists** — prevents reinitialization attacks.
- Requires `payer` and `space` to also be specified.
- Requires `system_program` in the same `Context`.

### When to Use
Any instruction that creates a new on-chain data account for the first time.

### Example
```rust
#[derive(Accounts)]
pub struct CreateUserProfile<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + UserProfile::INIT_SPACE,
        seeds = [b"profile", user.key().as_ref()],
        bump,
    )]
    pub profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Security Notes
- Prefer `init` over `init_if_needed` unless re-use is a documented requirement.
- `init` prevents account reinitialization attacks by erroring if the account already exists.
- Always pair with `seeds` + `bump` for PDA accounts to ensure deterministic addressing.

---

## `init_if_needed`

### Purpose
Initializes the account if it does not yet exist; silently skips initialization if it already exists.

### What It Does
- Behaves like `init` on first call.
- On subsequent calls, performs no initialization — the existing account is returned as-is.

### When to Use
Only when an instruction must be idempotent (e.g., a "deposit-or-create" pattern where initialization and first deposit happen atomically). Use with extreme caution.

### Example
```rust
// Requires the `init-if-needed` feature flag in Cargo.toml:
// anchor-lang = { version = "0.29.0", features = ["init-if-needed"] }

#[derive(Accounts)]
pub struct DepositOrCreate<'info> {
    #[account(
        init_if_needed,
        payer = user,
        space = 8 + UserPosition::INIT_SPACE,
        seeds = [b"position", user.key().as_ref()],
        bump,
    )]
    pub position: Account<'info, UserPosition>,

    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Security Notes
> ⚠️ **High-Risk Constraint** — `init_if_needed` enables account reinitialization attacks if the handler writes to account fields unconditionally on every call.

**Always add an `is_initialized` guard in the handler:**
```rust
pub fn deposit_or_create(ctx: Context<DepositOrCreate>, amount: u64) -> Result<()> {
    let pos = &mut ctx.accounts.position;
    if !pos.is_initialized {
        pos.is_initialized = true;
        pos.user = ctx.accounts.user.key();
        pos.bump = ctx.bumps.position;
    }
    // Safe to modify shared fields below
    pos.deposited = pos.deposited.checked_add(amount).ok_or(ErrorCode::Overflow)?;
    Ok(())
}
```

---

## `payer`

### Purpose
Specifies which account's lamports will fund the creation of a new account (used with `init` or `realloc`).

### What It Does
- Deducts rent-exempt lamports from the `payer` account.
- The `payer` account must be `mut` and a `Signer`.

### Example
```rust
#[account(
    init,
    payer = authority,     // authority pays for this account
    space = 8 + Config::INIT_SPACE,
)]
pub config: Account<'info, Config>,

#[account(mut)]
pub authority: Signer<'info>,
```

### Security Notes
- The `payer` must be marked `mut` — Anchor enforces this automatically.
- Verify that the intended party (user, protocol, DAO) is correctly identified as `payer`. Charging the wrong account can be exploited to drain admin wallets.

---

## `space`

### Purpose
Specifies the number of bytes to allocate when creating a new account with `init`.

### What It Does
- Passes the `space` value to `system_program::create_account`.
- Anchor uses this to calculate the rent-exempt minimum lamports required.

### When to Use
Always paired with `init`. The value must be `8 + [sum of all field sizes]` — the leading 8 bytes are Anchor's discriminator.

### Example
```rust
// Preferred: use INIT_SPACE derived from the account struct
#[account(
    init,
    payer = payer,
    space = 8 + Vault::INIT_SPACE,
)]
pub vault: Account<'info, Vault>,

// Vault struct with INIT_SPACE:
#[account]
#[derive(InitSpace)]           // Anchor derive macro that calculates INIT_SPACE
pub struct Vault {
    pub authority: Pubkey,     // 32 bytes
    pub balance: u64,          // 8 bytes
    pub bump: u8,              // 1 byte
    // INIT_SPACE = 41
}
```

### Security Notes
- **Under-allocation** causes runtime serialization errors and can brick an account.
- **Over-allocation** wastes lamports but is not a security vulnerability.
- Use `#[derive(InitSpace)]` to calculate space automatically and avoid manual arithmetic errors.
- For accounts with `Vec` or `String` fields, pass `max_len` to `InitSpace` and document the maximum.

---

## `seeds`

### Purpose
Specifies the byte seeds used to derive a PDA (Program Derived Address). Anchor re-derives the PDA from these seeds and verifies the account matches.

### What It Does
- Calls `Pubkey::find_program_address(seeds, program_id)` (or `create_program_address` with the stored bump).
- Asserts the derived address equals the provided account key.
- **Eliminates the need for manual PDA derivation and comparison in the handler.**

### When to Use
Every account that is a PDA must use the `seeds` constraint. Never accept a PDA from the caller without re-deriving it.

### Example
```rust
#[derive(Accounts)]
pub struct ClaimReward<'info> {
    #[account(
        mut,
        seeds = [
            b"reward",
            user.key().as_ref(),
            reward_round.to_le_bytes().as_ref(),
        ],
        bump = reward_account.bump,
    )]
    pub reward_account: Account<'info, RewardAccount>,
    pub user: Signer<'info>,
}
```

### Security Notes
- `seeds` alone (without `bump`) uses `find_program_address` internally, which iterates bumps — use `bump = stored_bump` for gas efficiency and to enforce the canonical bump.
- Every seed component must be deterministic from on-chain data. Never use user-supplied arbitrary bytes as seeds without domain separation prefixes (e.g., `b"reward"`).

---

## `bump`

### Purpose
Validates or captures the canonical bump seed used to derive a PDA.

### What It Validates
- **With `bump` alone** (no value): Anchor uses `find_program_address` to find the canonical bump and stores it in `ctx.bumps.<account_name>`.
- **With `bump = <expr>`**: Anchor uses `create_program_address(seeds + [bump_value], program_id)` and asserts the result matches the account key — this is faster (no iteration) and enforces the stored canonical bump.

### When to Use
- Use `bump` on `init` instructions to capture the canonical bump and store it in the account.
- Use `bump = account.bump` on all subsequent instructions to enforce the stored canonical bump.

### Example
```rust
// On init — capture the canonical bump:
#[account(
    init,
    payer = payer,
    space = 8 + Vault::INIT_SPACE,
    seeds = [b"vault", payer.key().as_ref()],
    bump,
)]
pub vault: Account<'info, Vault>,

// Store it:
ctx.accounts.vault.bump = ctx.bumps.vault;

// On subsequent use — enforce the stored canonical bump:
#[account(
    mut,
    seeds = [b"vault", authority.key().as_ref()],
    bump = vault.bump,             // uses stored value, not find_program_address
    has_one = authority,
)]
pub vault: Account<'info, Vault>,
```

### Security Notes
- Never accept a `bump: u8` as an instruction argument and use it to derive or validate a PDA — an attacker can supply a non-canonical bump pointing to a different account.
- Always store the canonical bump on-chain at initialization time.

---

## `has_one`

### Purpose
Asserts that a field on the account struct equals the key of another account in the same `Context`.

### What It Does
Generates a constraint equivalent to:
```rust
constraint = account.field == other_account.key()
```

### When to Use
Validating ownership relationships between accounts. The most common use cases:
- `vault.authority == authority.key()` → `#[account(has_one = authority)]`
- `config.admin == admin.key()` → `#[account(has_one = admin)]`
- `vault.token_account == vault_token_account.key()` → `#[account(has_one = vault_token_account)]`

### Example
```rust
#[derive(Accounts)]
pub struct Withdraw<'info> {
    #[account(
        mut,
        has_one = authority @ VaultError::Unauthorized,
        has_one = vault_token_account @ VaultError::InvalidTokenAccount,
    )]
    pub vault: Account<'info, Vault>,
    pub authority: Signer<'info>,
    #[account(mut)]
    pub vault_token_account: Account<'info, TokenAccount>,
}
```

### Security Notes
- `has_one` validates that `vault.authority == authority.key()` but does **not** validate that `authority` signed the transaction — always combine with `Signer<'info>`.
- Always provide a custom error code (`@ ErrorCode::Variant`) for debuggable error messages in production.
- Chain multiple `has_one` constraints on the same account for multi-relationship validation.

---

## `owner`

### Purpose
Asserts that the account is owned by a specific program ID.

### What It Does
- Checks `account.owner == expected_program_id` at constraint evaluation.
- Prevents account spoofing — an attacker cannot pass an account owned by their program if `owner` is enforced.

### When to Use
When using `AccountInfo<'info>` for accounts that must be owned by a specific program (e.g., a foreign program's account, or the System Program). Note: `Account<'info, T>` already validates ownership automatically.

### Example
```rust
// Verify a native system account:
#[account(owner = system_program::ID)]
pub system_account: AccountInfo<'info>,

// Verify an account owned by a specific external program:
#[account(owner = some_protocol::ID @ ErrorCode::WrongOwner)]
pub external_state: AccountInfo<'info>,
```

### Security Notes
- `Account<'info, T>` implicitly validates `owner == current_program_id` — you do not need an explicit `owner` constraint for your own program's accounts.
- Always use `Account<'info, T>` over `AccountInfo<'info>` + `#[account(owner = ...)]` for your own program's data accounts, because `Account` also validates the discriminator.

---

## `address`

### Purpose
Asserts that the account's public key exactly equals a specific, hardcoded address.

### What It Does
- Checks `account.key() == expected_pubkey` at constraint evaluation.

### When to Use
Hardcoding required system addresses, oracle addresses, or known program IDs when `Program<'info, T>` is not available.

### Example
```rust
use anchor_lang::solana_program::sysvar;

#[derive(Accounts)]
pub struct ConsumeEntropy<'info> {
    // Enforce the SlotHashes sysvar address exactly
    #[account(address = sysvar::slot_hashes::ID)]
    pub slot_hashes: AccountInfo<'info>,

    // Enforce a known oracle account
    #[account(address = known_oracle::PRICE_FEED_PUBKEY @ ErrorCode::WrongOracle)]
    pub price_feed: AccountInfo<'info>,
}
```

### Security Notes
- `address` is appropriate for one-off hardcoded requirements. For type-safe program accounts, prefer `Program<'info, T>` which validates the program ID and marks the account as executable.
- Hardcoded addresses must be kept up to date if the protocol upgrades its oracle or external dependencies.

---

## `constraint`

### Purpose
Evaluates an arbitrary boolean expression. If the expression is `false`, the instruction fails with the provided error code.

### What It Does
- Accepts any Rust expression that evaluates to `bool`.
- Can reference any field of any account in the same `Context`.
- Supports a custom error code via `@ ErrorCode::Variant`.

### When to Use
For any validation logic that cannot be expressed by the other built-in constraints.

### Example
```rust
#[derive(Accounts)]
pub struct Swap<'info> {
    #[account(
        mut,
        // Ensure the two token accounts are not the same (prevent duplicate account attack)
        constraint = input_token_account.key() != output_token_account.key()
            @ SwapError::DuplicateAccount,
        // Ensure the user is not sending more than the protocol maximum
        constraint = amount <= config.max_swap_amount
            @ SwapError::ExceedsMaximum,
    )]
    pub input_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub output_token_account: Account<'info, TokenAccount>,

    pub config: Account<'info, ProtocolConfig>,
}
```

### Security Notes
- `constraint` expressions are evaluated after all other constraints, so they can reference deserialized fields.
- Keep individual `constraint` expressions simple and readable — split complex logic into multiple constraints or move it into the handler with `require!`.
- Always include `@ ErrorCode::DescriptiveError` so failures are diagnosable in production.

---

## `close`

### Purpose
Closes the account at the end of the instruction: transfers all lamports to a designated receiver account and zeroes the account's data.

### What It Does
- Writes the `CLOSED_ACCOUNT_DISCRIMINATOR` (all `0xFF` bytes) to the account data to prevent resurrection.
- Transfers all lamports from the account to the `close = destination` account.
- The account will be garbage-collected by the runtime.

### When to Use
Any instruction that permanently closes a data account (e.g., canceling an order, closing a position, withdrawing all funds from a one-time escrow).

### Example
```rust
#[derive(Accounts)]
pub struct CancelEscrow<'info> {
    #[account(
        mut,
        close = authority,           // lamports returned to authority
        has_one = authority @ EscrowError::Unauthorized,
    )]
    pub escrow: Account<'info, Escrow>,
    pub authority: Signer<'info>,
}
```

### Security Notes
- **Never close an account manually** by transferring lamports without using the `close` constraint — manual closure leaves account data intact, enabling resurrection attacks.
- The `close` constraint zeroes data before the lamport transfer and marks the account with the closed discriminator, preventing any further deserialization.
- Closed accounts cannot be used in the same transaction after they are closed — Anchor enforces this at the constraint level.

---

## `realloc`

### Purpose
Resizes an existing account's data allocation. Used when the account needs more (or less) space than originally allocated.

### What It Does
- Calls `account.realloc(new_size, zero_init)` during constraint evaluation.
- Adjusts lamports in or out (via `realloc::payer`) to maintain rent-exemption.
- Must be combined with `realloc::payer` and `realloc::zero`.

### When to Use
Dynamic data accounts that grow over time (e.g., a protocol config that adds new fields in an upgrade, or a list that appends entries).

### Example
```rust
#[derive(Accounts)]
pub struct AddValidator<'info> {
    #[account(
        mut,
        realloc = 8 + ValidatorSet::INIT_SPACE + (validator_set.validators.len() + 1) * 32,
        realloc::payer = authority,
        realloc::zero = true,           // new bytes are zeroed
    )]
    pub validator_set: Account<'info, ValidatorSet>,

    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Security Notes
- **Always set `realloc::zero = true`** when growing an account — uninitialized memory can contain stale data from previous uses that could be misread as valid state.
- Verify the new size calculation cannot overflow `usize` on the target platform.
- Ensure the `realloc::payer` has sufficient lamports before calling — the instruction will fail if it does not, but verify this in tests.

---

## `realloc::payer`

### Purpose
Designates which account provides or receives lamports when the account is resized with `realloc`.

### What It Does
- If the account grows, lamports are transferred **from** the `realloc::payer` to the resized account to maintain rent-exemption.
- If the account shrinks, excess lamports are returned **to** the `realloc::payer`.

### Example
```rust
#[account(
    mut,
    realloc = new_size,
    realloc::payer = admin,      // admin pays for growth, receives refund for shrinkage
    realloc::zero = false,
)]
pub state: Account<'info, ProtocolState>,
```

---

## `realloc::zero`

### Purpose
Controls whether newly allocated bytes (when growing an account) are initialized to zero.

### Values
| Value | Behavior |
|---|---|
| `true` | New bytes are zeroed — **strongly recommended** |
| `false` | New bytes are left uninitialized — only safe if you immediately overwrite them |

### Security Notes
- Set `realloc::zero = true` in all cases unless you have a documented, performance-critical reason not to, and you are certain the new bytes are overwritten before being read.

---

## `zero`

### Purpose
Used with accounts that are pre-allocated (e.g., via `solana_program::system_instruction::create_account`) before the Anchor instruction runs. Validates the account is all zeroes (uninitialized) and assigns it the program's ownership.

### What It Does
- Checks that all account data bytes are zero (no prior initialization).
- Sets the program ID as the account owner.
- Does **not** call System Program — the account must already exist with sufficient lamports.

### When to Use
Large accounts (> 10 KB) that cannot be created in a single transaction due to compute unit limits. The pattern is:
1. Create the account via a separate System Program transaction.
2. Use `zero` in the Anchor init instruction to take ownership.

### Example
```rust
#[derive(Accounts)]
pub struct InitializeLargeState<'info> {
    #[account(zero)]              // account was pre-created, now being initialized
    pub large_state: Account<'info, LargeState>,
    pub authority: Signer<'info>,
}
```

### Security Notes
- The `zero` check (all bytes are zero) prevents an attacker from passing a pre-populated account that appears fresh.
- Ensure the pre-creation transaction is also signed by the intended authority so an attacker cannot front-run the creation with a different account.

---

## `executable`

### Purpose
Asserts that the account contains a deployed, executable Solana program.

### What It Does
- Checks `account.executable == true` at constraint evaluation time.

### When to Use
When an instruction accepts a program account that will be used for a CPI, but `Program<'info, T>` is not available because the program type is unknown at compile time (e.g., a plugin or dynamic dispatch system).

### Example
```rust
#[derive(Accounts)]
pub struct PluginInstruction<'info> {
    // Verify it is a program, not a data account
    #[account(executable)]
    pub plugin_program: AccountInfo<'info>,
}
```

### Security Notes
- `executable` only verifies the account is a program — it does **not** verify which program it is. Always combine with `address = known_program::ID` or validate the program ID in the handler if security depends on it.
- For known programs, prefer `Program<'info, T>` (validates ID + executable) over `executable` alone on `AccountInfo`.

---

## Constraint Interaction Cheatsheet

| Situation | Correct Constraint Combination |
|---|---|
| New PDA account, user pays | `init, payer = user, space = 8 + T::INIT_SPACE, seeds = [...], bump` |
| Existing PDA, read-only | `seeds = [...], bump = account.bump` |
| Existing PDA, writable | `mut, seeds = [...], bump = account.bump` |
| Closing a PDA | `mut, close = destination, seeds = [...], bump = account.bump` |
| Authority check | `has_one = authority` on data account + `Signer<'info>` on authority |
| Exact address required | `address = SomePubkey::ID` |
| Custom validation | `constraint = <bool_expr> @ ErrorCode::Variant` |
| Growing a dynamic list | `mut, realloc = new_size, realloc::payer = payer, realloc::zero = true` |
| Idempotent init (with guard) | `init_if_needed, payer = user, space = ..., seeds = [...], bump` + `is_initialized` guard in handler |
| External owned account | `owner = external_program::ID` on `AccountInfo` |

---

*Last updated: 2025-01-15 | Anchor v0.29–v0.30 compatible*
