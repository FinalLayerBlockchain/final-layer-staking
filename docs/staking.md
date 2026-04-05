# Staking Pool Mechanics

## Overview

Final Layer uses a custom staking pool contract (`fl_staking_pool v5`) deployed to validator accounts. It extends NEAR's standard staking pool with:

- **Time-based lockup** (48-hour lock before unstaking is allowed)
- **4-epoch unbonding period** after unstaking
- **PQC key registration** with exact-length enforcement
- **Quantum-resistant reward claiming**

---

## Key Parameters

| Parameter | Value |
|---|---|
| `LOCKUP_NS` | `4 * 43200 * 1_000_000_000` ns = 48 hours |
| `NUM_EPOCHS_TO_UNLOCK` | 4 epochs (~48 hours at ~12h/epoch) |
| Epoch length | 43,200 blocks |

---

## Full Staking Lifecycle

### Step 1 — `deposit_and_stake()`

Deposit FLC and immediately stake it.

- `staked_balance` increases
- `unlock_timestamp_ns = block_timestamp + LOCKUP_NS` (48h lock starts)
- `is_locked = true`

**First-depositor protection:** When the pool has no existing shares (`total_stake_shares == 0`), `last_locked_balance` is synced to the protocol's current locked balance before `internal_ping()` runs. This prevents phantom rewards being credited to the first depositor.

```rust
if self.total_stake_shares == 0 {
    self.last_locked_balance = env::account_locked_balance().as_yoctonear();
}
self.internal_ping();
```

---

### Step 2 — `claim_rewards()` (optional, resets lock)

Moves accrued rewards from `rewards_earned` into `unstaked_balance`.

**Important:** Calling `claim_rewards()` **resets the lock** to `now + LOCKUP_NS`. Do not call this if you intend to unstake soon.

```rust
pub fn claim_rewards(&mut self) {
    let account_id = env::predecessor_account_id();
    let mut d = self.get_delegator(&account_id);
    self.internal_ping();
    let rewards = d.rewards_earned;
    d.unstaked += rewards;
    d.rewards_earned = 0;
    d.unstake_available_epoch = env::epoch_height() + NUM_EPOCHS_TO_UNLOCK;
    d.unlock_timestamp_ns = env::block_timestamp() + LOCKUP_NS;  // lock reset
    self.save_delegator(&account_id, &d);
}
```

---

### Step 3 — `unstake_all()` (requires lock expired)

Moves all staked balance to `unstaked_balance` and starts the unbonding clock.

- Requires `block_timestamp >= unlock_timestamp_ns` — panics with `"Stake is still locked"` if called early
- Sets `unstake_available_epoch = epoch_height + NUM_EPOCHS_TO_UNLOCK`
- `staked_balance → 0`
- `unstaked_balance` increases

---

### Step 4 — `withdraw_all()` (requires unbonding complete)

Sends the unstaked balance back to the delegator's wallet.

- Requires `epoch_height >= unstake_available_epoch` — panics with `"Still in unbonding period"` if called early
- `unstaked_balance → 0`
- FLC transferred to caller's account

---

## Security Properties

### Lock enforcement

```rust
require!(
    env::block_timestamp() >= self.unlock_timestamp_ns,
    "Stake is still locked"
);
```

Both `unstake_available_epoch` (epoch-based) and `unlock_timestamp_ns` (nanosecond-based) must be satisfied. The dual check prevents clock manipulation.

### Unbonding enforcement

```rust
require!(
    account.unstaked_available_epoch_height <= env::epoch_height(),
    "The unstaked balance is not yet available due to unbonding period"
);
```

---

## PQC Key Registration

The staking pool supports registering PQC public keys for reward recipients and stakers. Keys are validated via `parse_key_string()`:

```rust
let pk_len: usize = match algo {
    "mldsa"  => 1952,
    "fndsa"  => 897,
    "slhdsa" => 32,
    _        => key_bytes.len(),
};
if key_bytes.len() != pk_len {
    panic!("Key must be exactly {} bytes for {}, got {}", pk_len, algo, key_bytes.len());
}
```

**No truncation** — keys with the wrong length are rejected with a descriptive panic. Unknown algorithms also panic.

Keys are stored with Borsh `Vec<u8>` encoding (4-byte LE u32 length prefix + raw bytes), which is required for proper deserialization in the NEAR VM.

---

## Staking Pool Versions

| Version | Changes |
|---|---|
| v1 | Initial implementation — wrong key bytes (missing Borsh length prefix) |
| v2 | Fixed Borsh encoding for PQC keys |
| v3 | Added lockup mechanism |
| v4 | `parse_key_string()` exact-length enforcement (no truncation) |
| v5 | First-delegator `last_locked_balance` fix |
