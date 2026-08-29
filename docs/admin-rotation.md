# Admin Rotation Procedure

This document describes the safe procedure for rotating admin control of the
Stellar Wrap Contract. Operators should follow it any time the admin address or
the Ed25519 signing pubkey needs to change.

---

## Two independent rotation types

The contract stores two separate privileged values. Rotating one does **not**
affect the other.

| Value | Storage key | Controls |
|---|---|---|
| Admin address | `DataKey::Admin` | Authorization for all admin-only functions (`update_admin`, `pause`, `upgrade`, `migrate`, …) |
| Signing pubkey | `DataKey::AdminPubKey` | Ed25519 public key used to verify mint payload signatures |

---

## Admin address rotation

### Preparation

1. Confirm the **current** admin on-chain before starting:
   ```bash
   stellar contract invoke \
     --id <CONTRACT_ID> \
     --network <NETWORK> \
     -- get_admin
   ```
2. Have the new admin verify they control their private key by signing a test
   transaction on testnet first.
3. Ensure no pending proposal already exists:
   ```bash
   stellar contract invoke \
     --id <CONTRACT_ID> \
     --network <NETWORK> \
     -- get_pending_admin
   ```
   If a proposal exists, cancel it before proceeding (see
   [Cancel a pending proposal](#cancel-a-pending-proposal)).

### Ruta A — Direct transfer (`update_admin`)

> **Warning:** This path is immediate and irreversible in the same transaction.
> Only use it when the new admin address has already been verified on testnet.

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  --source <CURRENT_ADMIN_SECRET> \
  -- update_admin \
  --new_admin <NEW_ADMIN_ADDRESS>
```

The contract replaces the stored admin address and emits an
`("admin", "updated")` event immediately.

### Ruta B — Two-step transfer (`propose_admin` + `accept_admin`) — recommended

This path keeps the current admin in control until the new admin explicitly
accepts, protecting against address typos and key-loss scenarios.

**Step 1 — Current admin proposes a new admin:**
```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  --source <CURRENT_ADMIN_SECRET> \
  -- propose_admin \
  --new_admin <NEW_ADMIN_ADDRESS>
```

**Step 2 — Verify the pending admin is correct:**
```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  -- get_pending_admin
```

Confirm the returned address matches the intended new admin before proceeding.

**Step 3 — New admin accepts:**
```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  --source <NEW_ADMIN_SECRET> \
  -- accept_admin
```

The contract atomically moves `PendingAdmin` → `Admin` and clears the pending
slot. An `("admin", "updated")` event is **not** emitted by `accept_admin`;
monitor via `get_admin` (see [Verification](#verification)).

### Cancel a pending proposal

If a mistake is detected **after** `propose_admin` but **before** `accept_admin`,
the current admin can cancel:

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  --source <CURRENT_ADMIN_SECRET> \
  -- cancel_proposed_admin
```

Only one proposal may be open at a time. `propose_admin` will panic with
`AdminTransferProposalExists` if you try to open a second one without cancelling
the first.

---

## Verification

Run these queries after every rotation to confirm the expected state:

```bash
# 1. Confirm new admin is active
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  -- get_admin

# 2. Confirm no pending proposal remains
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  -- get_pending_admin

# 3. Confirm contract health (initialized, has_admin, has_signing_key)
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  -- health

# 4. Smoke-test admin access with a no-op privileged call (e.g. unpause if paused, or pause+unpause)
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network <NETWORK> \
  --source <NEW_ADMIN_SECRET> \
  -- is_paused
```

---

## Event monitoring

A successful `update_admin` call emits one event:

- **Topic 0:** `admin` (`Symbol`)
- **Topic 1:** `updated` (`Symbol`)
- **Data:** `(old_admin, new_admin)` — both as `Address`

Indexers can watch for this event to detect admin rotations without polling, but
must always verify the live admin via `get_admin` before enforcing privileged
flows. **Do not infer state from events alone.**

`accept_admin` does not currently emit an event; use `get_admin` to confirm.

---

## Signing pubkey rotation

The Ed25519 signing pubkey (`DataKey::AdminPubKey`) is updated on-chain by the admin using `update_admin_pubkey(new_pubkey: BytesN<32>)`.

### Procedure

1. Admin calls `update_admin_pubkey`:
   ```bash
   stellar contract invoke \
     --id <CONTRACT_ID> \
     --network <NETWORK> \
     --source <ADMIN_SECRET> \
     -- update_admin_pubkey \
     --new_pubkey <NEW_ED25519_PUBKEY_HEX>
   ```

2. The contract validates that `new_pubkey` is not the all-zero key, overwrites `DataKey::AdminPubKey` in instance storage, and emits a rotation event:
   - **Topic 0:** `pubkey` (`Symbol`)
   - **Topic 1:** `rotate` (`Symbol`)
   - **Data:** `(old_pubkey, new_pubkey)` (`(BytesN<32>, BytesN<32>)`)

3. Signatures produced by the old key are rejected immediately upon rotation.

### Timelock Exemption Rationale

`update_admin_pubkey` is intentionally exempt from the timelock controller (like `pause`).

**Rationale:** Key compromise is the primary emergency event where a mandatory delay is actively harmful: during a timelock delay window (1 hour to 30 days), a compromised key would continue producing valid signatures on-chain. Exemption from the timelock allows the admin to immediately invalidate a compromised signing key without pausing all minting or requiring a WASM upgrade.

---

## Rollback plan

| Scenario | Recovery action |
|---|---|
| Wrong address proposed, not yet accepted (Ruta B) | Current admin calls `cancel_proposed_admin` |
| Tx not sent yet (Ruta A) | Do not broadcast; re-run with correct address |
| Wrong admin active, new admin **cooperates** | New admin calls `update_admin` to the correct address |
| Wrong admin active, new admin **does not cooperate** | No on-chain recovery — the contract remains under the wrong admin's control |

**The safest default is always Ruta B (propose + accept).** It provides a
cancel window and requires the new admin to prove key control before the old
admin loses access.

For mainnet rotations, always rehearse the full procedure on testnet with the
exact addresses involved before executing on mainnet.
