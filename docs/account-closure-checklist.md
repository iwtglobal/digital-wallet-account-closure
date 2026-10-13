# Account Closure Checklist

Practical checklist for product, engineering, and ops designing **digital wallet account closure**.

## Eligibility before terminal close

| Check | Pass criteria |
|-------|---------------|
| Available balance | Zero after successful drain, or explicit write-off under policy |
| Auth holds | None in OPEN or CAPTURE_PENDING |
| Disputes | No open cases, or cases moved to suspense with owner |
| Recurring mandates | Cancelled or transferred |
| Linked merchant settlement | Final batch posted or carried to successor wallet |
| Agent float link | Consumer close does not zero a separate agent entity |

## Process steps

- [ ] Capture `closure_reason_code` and requester channel
- [ ] Snapshot balances and hold IDs into `eligibility_snapshot`
- [ ] Block new debits; allow reversal credits
- [ ] Execute drain with stored `drain_txn_id`
- [ ] Wait or clear holds; record `holds_cleared_at`
- [ ] Revoke sessions, refresh tokens, USSD PIN, API keys
- [ ] Soft-delete beneficiaries; retain payment history
- [ ] Unbind MSISDN; set `msisdn_cooldown_until`
- [ ] Set status `CLOSED` and `closed_at`
- [ ] Schedule retention purge date; do not delete audit early

## Support playbook hooks

- Screen showing blockers (holds, disputes, pending drain)
- Ability to re-send drain to a verified destination
- Path for late reversal credit into suspense after close
- Clear customer message when cool-down prevents immediate reopen

## Anti-patterns

- Closing while `open_hold_count > 0`
- Keying wallet identity only by MSISDN with no stable `wallet_id`
- Hard-deleting statements on close day
- Auto-provisioning a new wallet for a recycled MSISDN during cool-down
