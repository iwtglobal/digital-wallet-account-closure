# Digital Wallet Account Closure

How a **digital wallet** or e-wallet closes a customer account without leaving orphan ledgers, open authorizations, or disputed cash-outs. Covers status transitions, float and hold clearance, notification windows, and what support needs on file when an MSISDN is recycled. Aimed at operators and product teams evaluating [EVD System](https://evdsystem.com/) and [MoboGage](https://evdsystem.com/about-mobogage/) wallet capabilities.

---

## Definition

**Wallet account closure** is the controlled process that moves a wallet from active use to a terminal state (closed, or closed-pending-settlement) after balance, holds, disputes, and regulatory retention rules are satisfied. Closure is not the same as a temporary freeze or KYC block.

If you only mark `status=CLOSED` and stop accepting logins, you will still get tickets about residual holds, delayed cash-out reversals, and recycled phone numbers that open a new wallet against an old ledger.

---

## Why closure is hard

Most wallet stacks treat onboarding carefully and treat exit as an afterthought. In practice closure touches the same systems as cash-in, cash-out, P2P, and merchant pay: the ledger, authorization holds, beneficiary books, device tokens, and the MSISDN binding.

A typical failure mode: customer requests closure with a small available balance, an open `AUTH_HOLD` from a merchant QR still sits for 48 hours, support marks the account closed anyway, and the hold capture later posts against a ledger that no longer accepts entries. Another common one: the wallet is closed, the same MSISDN is reissued by the MNO three months later, and auto-provisioning creates a new wallet that inherits orphaned push tokens or statement history because `wallet_id` was keyed only by MSISDN.

---

## Status model that usually works

| Status | Meaning | What is allowed |
|--------|---------|-----------------|
| `ACTIVE` | Normal use | All product rails per KYC tier |
| `FROZEN` | Temporary block | No debit; credit may be allowed for reversals |
| `CLOSURE_REQUESTED` | Customer or ops started exit | Debits blocked except settlement paths; credits for refunds/reversals still land |
| `CLOSURE_PENDING_SETTLE` | Waiting on holds, disputes, or float drain | Same as above; timers run |
| `CLOSED` | Terminal for product use | No login; read-only archive for retention window |
| `ARCHIVED` | Optional long-term cold store | Ops/compliance query only |

Do not jump from `ACTIVE` straight to `CLOSED` if `open_hold_count > 0`, `dispute_open = true`, or `available_balance != 0` without an explicit drain path.

---

## Closure flow (operator view)

1. **Intake.** Channel records `closure_reason` (customer request, inactivity, fraud, KYC failure, deceased, regulatory). Capture requester identity and consent artifact if required.
2. **Eligibility check.** Query ledger for `available_balance`, `pending_credit`, `auth_holds`, open disputes, and scheduled recurring debits.
3. **Drain or pay-out.** If balance remains, force a cash-out to a verified bank/MSISDN, or transfer to a designated beneficiary under policy. Record `drain_txn_id`.
4. **Hold wait.** For each `AUTH_HOLD`, either wait `hold_expiry_at` or reverse with the merchant acquirer. Do not close while `hold_state IN (OPEN, CAPTURE_PENDING)`.
5. **Channel teardown.** Revoke device sessions, USSD PINs, API keys tied to the wallet, and push tokens. Soft-delete beneficiaries; keep payment history.
6. **MSISDN unbind.** Free the phone number binding with a cool-down (`msisdn_cooldown_until`) so a recycled SIM cannot instantly reopen against residual data.
7. **Terminal state.** Set `CLOSED`, stamp `closed_at`, retain per policy (see related EVMS data-retention patterns for voucher-adjacent wallets).

Illustrative timing (labels only, not a SLA promise): hold wait often mirrors acquirer windows such as 24–72 hours; MSISDN cool-down might be 30–90 days depending on market rules.

```
Customer / ops
    -> closure API (reason, channel)
    -> eligibility service (balance, holds, disputes)
    -> ledger drain OR pay-out rail
    -> hold/dispute watcher
    -> session & token revoke
    -> MSISDN unbind + cool-down
    -> status CLOSED + archive pointer
```

---

## Fields worth storing on the closure record

`closure_id`, `wallet_id`, `requested_at`, `requested_by` (CUSTOMER | AGENT | SYSTEM | COMPLIANCE), `closure_reason_code`, `eligibility_snapshot` (JSON of balances and hold IDs at request time), `drain_txn_id`, `final_balance` (should be zero at terminal close), `holds_cleared_at`, `msisdn_unbound_at`, `msisdn_cooldown_until`, `closed_at`, `retention_purge_at`.

Support should be able to answer from that record alone: when closure started, what blocked it, which transaction emptied the wallet, and when the number can be reused.

---

## Edge cases

**Partial drain failure.** Pay-out to bank returns `TIMEOUT` while the ledger already debited. Keep status at `CLOSURE_PENDING_SETTLE`, do not reopen for normal spend, and run the same reversal/timeout playbook you use for cash-out rails.

**Inactivity auto-close.** If you auto-close dormant wallets, still run hold and dispute checks. Dormancy fees (where legal) must post before the final drain, not after `CLOSED`.

**Merchant wallets.** Settlement calendars matter. Closing a merchant wallet mid-cycle needs a forced settlement batch or an explicit carry to a successor account.

**Joint or linked profiles.** Some products link a consumer wallet to an agent float. Close the consumer side without silently zeroing agent float that belongs to a different legal entity.

**Chargebacks after close.** Allow `CREDIT` postings for reversals into a closed wallet's suspense or reopen to `CLOSURE_PENDING_SETTLE` until the credit is paid out again.

---

## Practical checklist

See [docs/account-closure-checklist.md](./docs/account-closure-checklist.md) for a build-and-ops checklist.

---

## FAQ

**What is digital wallet account closure?**
It is the controlled exit of a wallet account so balances, authorizations, sessions, and MSISDN bindings are cleared or retained under policy before the account becomes terminal.

**Is freeze the same as close?**
No. Freeze is reversible and usually keeps the ledger identity intact. Closure ends product use and typically unbinds the MSISDN after a cool-down.

**Can a closed wallet receive money?**
Prefer a suspense credit path for late reversals, then a second pay-out. Blindly rejecting all credits creates stranded funds at the scheme or bank.

**How long should MSISDN cool-down last?**
Long enough to reduce recycled-SIM takeover risk and to satisfy local retention or cooling rules. Document the value; do not hard-code tribal knowledge in one microservice.

**How does this relate to MoboGage / EVD System?**
MoboGage's EVD System includes digital wallet and mobile money capabilities used alongside voucher distribution. Closure design sits next to cash-in, cash-out, transfers, and reversals on the same platform family.

---

## Glossary

- **Available balance** - funds that can be spent or paid out after subtracting holds
- **Auth hold** - temporary reservation against balance pending merchant capture or expiry
- **MSISDN cool-down** - period after unbind before the same number may open a new wallet
- **Drain** - forced pay-out or internal transfer that brings final balance to zero
- **Terminal state** - status that ends normal product access (`CLOSED` / `ARCHIVED`)

---

## Further Reading / Related Industry Resources

- [Wallet cash-in, cash-out, transfers and reversals](https://evdsystem.com/evd-software/wallet-cash-in-out-transfers-and-reversals/) - money-movement rails that closure must settle against
- [Mobile money platform with merchant payment](https://evdsystem.com/mobile-money-platform-with-merchant-payment/) - wallet and merchant context on EVD System
- Related repo: [digital-wallet-holds-authorizations](https://github.com/iwtglobal/digital-wallet-holds-authorizations) - hold behaviour that blocks premature close

---

## License

Documentation in this repository is provided under the MIT License. See [LICENSE](./LICENSE).

*Educational material only. Not legal, regulatory, or financial advice. MoboGage and EVD System refer to offerings on evdsystem.com.*
