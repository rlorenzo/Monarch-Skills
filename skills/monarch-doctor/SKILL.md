---
name: monarch-doctor
description: Health check for a Monarch Money account — institution connections needing re-authentication, disconnected or stale accounts, sync-disabled accounts, and the data gaps they cause. Use when the user asks whether Monarch data is current, why balances or transactions look wrong or missing, or wants a general check-up of their account connections.
license: MIT
allowed-tools: mcp__monarch-money__get_account_sync_health mcp__monarch-money__get_accounts mcp__monarch-money__get_account_balance_history mcp__monarch-money__get_transactions
---

# Monarch doctor

Read-only. Reconnecting a bank happens in Monarch's UI or app; report what needs fixing
and what it is costing in data accuracy.

## Why this matters

A broken bank connection fails silently. Monarch keeps serving the last balances it saw
and stops importing transactions, so budgets, cash flow, and net worth drift without any
error appearing. Every other analysis in this repo is wrong in proportion to how long a
connection has been dead.

## Gather

1. `get_account_sync_health(stale_after_days=3)` — per-institution connection health.
   Returns a `needs_attention` list plus the full connection list. Empty `needs_attention`
   means every institution is syncing. Raise `stale_after_days` for accounts that
   legitimately update slowly (some 401k and mortgage feeds are weekly or monthly).
2. `get_accounts` — needed because `get_account_sync_health` does not report whether an
   account is active or hidden, which is what tells a broken connection apart from a
   closed one. Join the two by account name, and watch for near-duplicate names across
   people or accounts. Gives per-account `sync.state` (`ok`, `needs_reauth`, `disconnected`,
   `sync_disabled`, `manual`), `needs_reauth`, `disconnected_at`, `sync_disabled`,
   `data_provider`, `last_updated_at`, `is_active`, `is_hidden`, `balance`.
3. For an account whose balance looks suspicious, `get_account_balance_history(account_id=...)`
   to see when it flatlined — that dates the outage.
4. `get_transactions(start_date=..., end_date=...)` filtered to a suspect account when you
   need to show that transactions stopped arriving on a specific date. Both dates are
   required; passing one returns an error.

## Gotchas

- **Group by connection (`credential_id`), not by account or by institution name.** One
  dead link surfaces as many broken accounts, so per-account reporting buries the actual
  task. But one institution can also hold several separate connections — the same credit
  union can appear twice under different `credential_id`s, covering different accounts,
  each needing its own reconnection. Collapsing by name would under-report the work.
- **`sync.state: "manual"` is not broken.** Manual accounts never sync by design. So do
  accounts with `sync_disabled: true` that the user turned off deliberately.
- **Exclude closed accounts from the action lists.** `is_active: false`, or
  `is_hidden: true` with an old `last_updated_at`, means a closed or abandoned account,
  and there is nothing to reconnect. Drop a connection from "fix now" when *every* account
  under it is closed — but only then. A connection can cover a mix, and a live account
  sitting behind a dead link is the whole point of this check.
  Do not drop them silently: end the report with one line saying how many closed accounts
  were excluded, so a misfiled one is still visible. Monarch has no explicit closed flag
  here, so this is inferred from `is_active` and `is_hidden` — say so when it is the only
  reason you set something aside.
- **A recent `last_updated_at` with `needs_reauth: true` still means broken.** The
  timestamp records the last attempt, not the last successful import. Trust the state.
- **Zero balances are ambiguous.** A $0 balance on a broken connection may be a real zero
  or a failed read. Say which you cannot distinguish rather than reporting a $0 balance
  as fact.
- **Never recommend deleting an account to silence a warning.** Deleting an account in
  Monarch deletes its transactions and balance history with it, which rewrites past net
  worth and category totals. A long-dead connection on a closed account is usually holding
  real history — check with `get_transactions` before suggesting anything destructive, and
  prefer marking the account closed, or raising `stale_after_days`, over removing it.
  Removing an institution connection may also offer to delete its accounts; say that the
  user should read that dialog rather than assuming it keeps them.
- Different providers (`PLAID`, `FINICITY`, `MX`, and others) break differently and
  reconnect differently. Name the provider — it tells the user what the flow will look like.

## Assess impact

For each broken connection, say what it costs, not just that it is broken:

- How long it has been out (from `last_updated_at` or a flatlined balance history).
- Which categories and budgets are now incomplete — a dead checking account means missing
  spending everywhere, a dead brokerage means only a wrong net worth.
- Whether the frozen balance is material to net worth.

Rank by impact: a dead primary checking account outranks a dormant store card, whatever
their balances.

## Report

Open with the one-line verdict: how many institutions need attention out of how many, and
whether the user's data can be trusted right now.

1. **Fix now** — institution, accounts affected, days stale, provider, what is missing
   because of it.
2. **Worth a look** — stale but plausibly normal cadence, or ambiguous.
3. **Working as intended** — manual, intentionally sync-disabled, closed. One line total,
   not a list, unless something looks misfiled. A closed account with a very stale
   connection belongs here, not in "fix now": there is nothing to reconnect.

End with the concrete next action per institution, in impact order. If everything is
healthy, say that in one line and stop.
