# Permission changes and hidden records

Contract tests: `shared/src/test_permission_visibility.rs` (Issue #445).

The timeline decides visibility at read time (`timeline::can_view`), using a
`Viewer` that `viewer_for` builds from on-chain roles. These tests pin down how
hidden records behave when roles change.

| Scenario | Expected behaviour |
|---|---|
| `Upgrader` or `Admin` granted | The address becomes a maintainer and sees participant-only and redacted entries |
| That role revoked | Access goes away on the next read |
| A non-maintainer role granted (e.g. `TreasuryManager`) | No extra visibility |
| Redacted public entry | Hidden from anonymous readers and from its own author; visible to maintainers |
| Admin ownership transfer accepted | The new owner gains maintainer visibility; the old owner loses it |
| `Admin` role revoked | That address can no longer redact (`Unauthorized`) |
| Hidden entries on a page | Skipped without shifting other viewers' pages or cursors |
| Deleted entry | Gone for everyone, maintainers included; the deletion stays in the audit trail |

## Edge case: stale viewers

A `Viewer` is a value snapshot. A copy held across a role change still reports
the old `is_maintainer`. Contract code must build the viewer with `viewer_for`
in the same invocation as the read. It must never cache a viewer in storage or
accept one as an argument. The `a_viewer_must_be_rebuilt_after_a_role_change`
test documents this.

## Recovery

Revoking a role is enough to remove access; nothing has to be cleaned up on the
records themselves. Only a maintainer can undo a redaction, by rewriting the
record. Deletion is permanent apart from the audit entry.
