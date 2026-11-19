# Ownership transfer: expiry and audit history

The shared admin identity changes hands in two steps (`shared::auth`):

1. `propose_ownership_transfer(current_owner, target, expiry)`: admin only.
   `expiry` is a ledger timestamp and must be in the future.
2. `accept_ownership_transfer(target)`: only the proposed target, and only
   while `now <= expiry`. Ownership (the stored admin and the `Admin` role)
   moves atomically.

The current owner can withdraw the proposal at any time with
`cancel_ownership_transfer`.

## Audit history (Issue #450)

Every step appends one `OwnershipAuditEntry` through
`shared::ownership_history`. Entries are never rewritten or removed.

| `action` | Written when | `actor` |
|---|---|---|
| `Proposed` | a proposal is stored | current owner |
| `Accepted` | the target accepts before expiry | new owner |
| `Cancelled` | the current owner cancels | current owner |
| `Expired` | `expire_ownership_transfer` clears a lapsed proposal, an accept arrives too late, or a new proposal replaces a lapsed one | whoever triggered it |

Each entry also stores `previous_owner`, `target_owner`, `expiry`, and the
ledger timestamp and sequence. Read them with:

- `ownership_history(start, limit)`: oldest first; `limit` is 1 to 50.
- `get_ownership_history_entry(seq)` and `ownership_history_len()`.

An `("auth", "own_hist")` event carries each entry, for indexers.

## Expiry edge cases

- **Expiry is inclusive.** At exactly `expiry` the proposal can still be
  accepted, and `expire_ownership_transfer` returns `InvalidArgument`.
- **Late accept.** `accept_ownership_transfer` after expiry returns
  `TransferExpired`. Inside a contract invocation an error rolls back every
  write that invocation made, including the cleanup it attempts. Use
  `expire_ownership_transfer(caller)` to clear a lapsed proposal for good. It
  is permissionless but requires the caller's signature, so the history names
  a real actor.
- **Re-proposing.** A new proposal is rejected with `TransferAlreadyPending`
  while the old one is live. Once the old one has lapsed, the new proposal
  replaces it and an `Expired` entry for the old one is written first.
- **Unauthorized calls** (a stranger proposing, cancelling or accepting) fail
  before any write and leave no history.

## Recovery

A transfer to the wrong address can be cancelled until it is accepted. After
acceptance, the new owner has to transfer back with a fresh proposal; the
history keeps both transfers.

## Compatibility

The history uses its own storage keys (`OwnershipHistoryKey`). The
`PendingOwnershipTransfer` layout and the existing events are unchanged, so no
migration is needed. Transfers completed before this change have no history
entries; the history starts with the first transition after the upgrade.
