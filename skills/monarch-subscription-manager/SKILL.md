---
name: monarch-subscription-manager
description: Find every recurring subscription and membership in Monarch Money, total what they cost per month and year, and flag candidates to cancel, downgrade, or question. Use when the user asks about subscriptions, recurring charges, what they pay monthly, or wants to cut recurring costs.
license: MIT
allowed-tools: mcp__monarch-money__check_auth_status mcp__monarch-money__get_recurring_transactions mcp__monarch-money__get_transactions mcp__monarch-money__search_transactions mcp__monarch-money__get_merchant mcp__claude_ai_Monarch__GetRecurring mcp__claude_ai_Monarch__GetTransactions mcp__claude_ai_Monarch__GetMerchants mcp__claude_ai_Monarch__GetCashFlow mcp__monarch__GetRecurring mcp__monarch__GetTransactions mcp__monarch__GetMerchants mcp__monarch__GetCashFlow
---

# Subscription manager

Read-only. You cannot cancel anything — produce a list the user acts on.

## Pick the server

Do this first, and name the server you used in the first line of the report.

1. **Community server** (robcerda/monarch-mcp-server): if `check_auth_status` is available
   and reports a session, follow this file as written. If it reports no session, or the
   first read fails on auth, treat this server as not connected.
2. **Official connector** (Monarch's hosted MCP): otherwise, if a server with "monarch" in its
   name, in any case, has a tool ending in `__GetTransactions`, read `references/official.md` in
   this skill's directory.
   It replaces Gather, Gotchas, and Apply below, and everything else still applies using the
   field names it gives. Without sign-in the connector only exposes `authenticate` tools,
   so it is not connected.
3. **Neither:** stop. Tell the user no Monarch server is connected and that either one works:
   - Official connector: `claude mcp add --transport http monarch https://api.monarch.com/mcp`,
     or add it as a claude.ai connector, then sign in through `/mcp`. It has fewer features.
   - Community server: supports everything, and needs `uv`.

   Link both to <https://github.com/rlorenzo/Monarch-Skills#connect-a-monarch-server>.

If both are connected, use the community server. Do not switch servers once a read succeeds.

## Gather

1. `get_recurring_transactions(start_date=..., end_date=..., include_liabilities=False, limit=100)`
   — Monarch's known recurring streams. Each row has `date`, `amount`, `is_past`,
   `transaction_id`, `category`, `account`, and a `stream` with `id`, `name`,
   `frequency`, `amount`, `is_approximate`, `merchant`, `merchant_id`. This is the
   starting list, not the whole list.
2. `get_transactions(start_date=..., end_date=...)` over the last 6-12 months to catch
   recurring charges Monarch has not flagged as streams. Page with `limit`/`offset`.
3. `search_transactions(search="<merchant>", start_date=..., end_date=...)` per candidate
   to get its full charge history, confirm cadence, and find price changes.

## Gotchas

- **`get_transactions` includes hidden transactions by default.** Rows carry
  `hide_from_reports`, and Monarch's reports leave those out, so sums taken from rows can run
  high. Pass `include_hidden=False` to leave them out.
- **Monarch's stream list is incomplete.** Anything it has not learned yet is missing, so
  a subscription audit that only calls `get_recurring_transactions` will understate the
  total. Always sweep raw transactions for merchants charging a similar amount on a
  similar day each month. A merchant can also hold an `is_active: true` stream that this
  list leaves out, when its `has_active_recurring_streams` is `false`, so a regular
  charge missing here may already be a stream. `get_merchant` shows it.
- **Liability bills are included unless you opt out.** The default adds synced credit
  card and loan statements as recurring rows. A card statement is the sum of charges
  already counted on their own, so pass `include_liabilities=False` or the card bill
  double-counts every subscription billed to it.
- **Always pass `limit`, though its value does nothing.** With any `limit`, Monarch returns the
  whole window once, ignoring `limit` and `offset`, so make one call per window and do not page.
  Without one, the tool pages for you, but since `offset` is ignored, a window of 100 rows or more
  comes back repeated up to 50 times. Verified live.
- **The window includes future predictions.** `is_past: false` rows are forecast
  occurrences, and `transaction_id: null` means no actual transaction. Never count a
  prediction as money already spent.
- **`stream.amount` is the expected amount, not what was charged.** A row can show
  `amount: -144.07` against `stream.amount: -289.71` — a partial or changed bill. Use
  actual transactions for real cost; treat `is_approximate: true` as a variable bill.
- **Not every stream is a subscription.** Transfers between own accounts, mortgage, auto
  loan, paychecks, credit card payments, and utilities all recur. Separate true
  discretionary subscriptions from fixed obligations and from transfers — the user asked
  about things they could cancel.
- **Amounts are signed**: expenses negative, income positive. Positive streams are income
  or inbound transfers.
- Some streams have `category: null` and `account: null`. Look them up by merchant rather
  than dropping them.

## Normalize

Convert every subscription to monthly and annual cost, using the real charge history:

- monthly → amount; annual → amount ÷ 12; quarterly → ÷ 3; weekly → × 4.33;
  biweekly → × 2.17.
- Show both the monthly figure and the annual figure. Annual is what makes a $14/mo
  service feel like the $168/yr decision it is.
- Flag any subscription whose amount rose over the window, with old → new and the annual
  impact. Silent price increases are the single most actionable finding here.

## Assess

For each, give a recommendation with the evidence for it:

- **Duplicate or overlapping** — two services doing the same job.
- **Possibly unused** — a charge that recurs with no related activity, or a category the
  user shows no other spending in. You cannot see usage, so ask rather than assert.
- **Downgrade candidate** — a premium tier where a cheaper tier plausibly suffices.
- **Price increased** — worth re-deciding at the new price.
- **Annual-billing candidate** — monthly plans that typically cost less billed annually.
- **Keep** — say so briefly; a list where everything is a problem is not credible.

Never assert the user does not use something. Ask: "Still using X at $Y/mo ($Z/yr)?"

## Report

Open with the totals: number of subscriptions, total monthly, total annual, and how that
compares to their income or discretionary spending if you have it.

Then a table sorted by annual cost, largest first: merchant, amount, frequency, annual,
account, category, recommendation. Fixed obligations and transfers go in a separate
short section, clearly labeled as not-cancellable, so the headline total means what it
says.

Close with the cut list: what to cancel or downgrade and the annual saving if they do.
