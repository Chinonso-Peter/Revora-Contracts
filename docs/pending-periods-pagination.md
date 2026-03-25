# Pending Periods Pagination

## Scope

This document covers the `get_pending_periods_page` read-only helper in
[`src/lib.rs`](/home/chinonso-peter/Revora-Contracts/src/lib.rs) and the
corresponding test coverage in
[`src/test.rs`](/home/chinonso-peter/Revora-Contracts/src/test.rs).

The feature is intentionally scoped to contracts code only. It does not change
off-chain indexers, frontend behavior, or deployment flow.

## API Summary

`get_pending_periods_page(env, issuer, namespace, token, holder, start, limit) -> (Vec<u64>, Option<u32>)`

Behavior:

- Returns pending period ids in deposit order.
- Uses `start` as a storage-index cursor, not as a `period_id`.
- Returns `Some(next_cursor)` only when more pending entries remain.
- Treats `limit = 0` as "use the default page size".
- Caps `limit` to `MAX_PAGE_LIMIT` to keep read cost predictable.
- Returns an empty page and `None` when `start` is already at or beyond the end.

## Security Assumptions

The hardened implementation makes the following assumptions explicit:

- Pending-period enumeration is treated as entitlement-scoped data, not public discovery data.
- A holder with `share_bps == 0` should not learn which deposited periods exist through
  pagination-only queries.
- Claim progress is represented by `LastClaimedIdx`, so pagination must never return
  periods before the holder's current claim cursor even if the caller supplies a stale `start`.
- Deposited periods are stored by append-only index (`PeriodEntry(offering_id, index)`),
  so the returned order is deterministic and stable across calls.

## Abuse and Failure Paths

### Zero-share probing

Risk:
A caller with no configured share could repeatedly page through results to infer offering
activity.

Mitigation:
Both `get_pending_periods` and `get_pending_periods_page` now return empty results for
zero-share holders.

### Oversized page requests

Risk:
Large `limit` values could encourage unexpectedly expensive read-only loops.

Mitigation:
The function normalizes page size with the same `MAX_PAGE_LIMIT` cap used by other
pagination endpoints.

### Stale or malicious cursors

Risk:
A caller can pass `start = 0` after partially claiming to try to reread already-claimed
entries.

Mitigation:
The effective cursor is `max(start, LastClaimedIdx)`, so the contract never pages before
the holder's claim boundary.

### Boundary arithmetic

Risk:
`start + limit` can overflow in edge cases.

Mitigation:
Cursor end calculations use `saturating_add` before clamping to the stored count.

## Deterministic Test Coverage

The test suite covers:

- first-page retrieval and cursor emission
- multi-page iteration to exhaustion
- stale cursor handling after partial claims
- `limit = 0` default-cap behavior
- oversized limit capping
- end-of-list empty-page behavior
- zero-share holders receiving empty results

Key tests live in:

- [`src/test.rs`](/home/chinonso-peter/Revora-Contracts/src/test.rs)
- [`src/chunking_tests.rs`](/home/chinonso-peter/Revora-Contracts/src/chunking_tests.rs)

## Reviewer Notes

- This change does not alter the write path for deposits or claims.
- The pagination cursor remains index-based for deterministic continuation.
- Returning empty results for zero-share holders is an intentional privacy hardening choice.
