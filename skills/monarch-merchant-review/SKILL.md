---
name: monarch-merchant-review
description: Find merchants split across several spellings in Monarch Money: the same business recorded under two or more names, which fragments merchant-level reporting and recurring-subscription detection. Use when the user asks about duplicate or split merchants, why a merchant's totals look low, why one subscription appears twice, or wants to tidy up merchant names.
license: MIT
allowed-tools: mcp__monarch-money__check_auth_status mcp__monarch-money__get_transactions mcp__monarch-money__get_merchant mcp__monarch-money__get_recurring_transactions mcp__monarch-money__get_transaction_rules mcp__claude_ai_Monarch__GetMerchants mcp__claude_ai_Monarch__GetCashFlow mcp__claude_ai_Monarch__GetTransactions mcp__claude_ai_Monarch__GetRecurring mcp__claude_ai_Monarch__ListRules mcp__monarch__GetMerchants mcp__monarch__GetCashFlow mcp__monarch__GetTransactions mcp__monarch__GetRecurring mcp__monarch__ListRules
---

# Merchant review

Reports first. Merges only when the user asks, and only with writes enabled. See
`## Apply`.

Monarch creates a separate merchant record per distinct name its providers send. The
same business arrives hyphenated and unhyphenated, in title case and in caps, with and
without a leading `The`, and every merchant-level view treats each spelling as its own
business.

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

## What this does and does not fix

Categories live on the transaction, not the merchant, so splits do **not** corrupt
category spending, budgets, or cash flow. Say this plainly, because the user may assume the
problem is bigger than it is.

What splits do break:

- Merchant-level totals and "top merchants". A split merchant is understated by
  however much sits under its other name.
- Recurring detection. Each merchant record carries its own recurring stream, so one
  subscription can appear twice, and `monarch-subscription-manager` will double-count it.
- Rules keyed to a merchant with the `eq` operator, which silently stop matching when a
  new spelling appears.

## Gather

There is no tool that lists merchants. Build the list from transactions.

1. `get_transactions(start_date=..., end_date=..., limit=2000, offset=N)`, paging on
   `offset` until you have `total_count`. Default to the full history unless the user
   names a window. Splits accumulate over years, and a 30-day window finds almost none.
2. Group by the `merchant` field and count transactions per name.

## Gotchas

- **Pass `limit` to `get_recurring_transactions`.** Any value returns the whole window once.
  Without one, a window of 100 rows or more comes back repeated up to 50 times.
- **`get_transactions` includes hidden transactions by default.** Rows carry
  `hide_from_reports`, and Monarch's reports leave those out, so sums taken from rows can run
  high. Pass `include_hidden=False` to leave them out.
- **`get_transactions` needs both dates.** Passing only `start_date` returns
  `{"error": true, "message": "You must specify both a startDate and endDate..."}`.
- **The response is an envelope, not a list.** Transactions are under `data`, with
  `count`, `total_count` and `truncated` alongside. Treat `truncated: true` as "keep
  paging", never as the whole answer.
- **A full page will exceed the tool-output limit and be written to a file instead.**
  At `limit=2000` a page is roughly 2 MB. Do not try to read those files into context.
  Extract `.result`, parse it, and do the grouping in a script.
- **`get_transactions` returns `merchant_id` on every row; `search_transactions` does
  not.** Collect each variant's id from the same scan that finds the splits, since the
  stream check and the merge confirmation below both need it.
- **`get_merchant` takes an id, not a name.** Use it to confirm a merge landed by
  `transaction_count`.
- **`rule_count` does not cover merchant-criteria rules.** It reads `0` on a merchant
  that a rule matches by name. Search `get_transaction_rules` for the spelling instead.

## Judge

Normalize names before comparing (lowercase, strip a leading `The`, drop punctuation
and whitespace, drop a trailing `Inc`/`LLC`/`Ltd`/`Corp`), then group names that collide.
Then throw most of the groups away. On a real account the raw collision list is mostly
noise, and reporting it unfiltered buries the few that matter.

Discard:

- **Statement lines that are not merchants.** Transfers, ACH deposits, ATM withdrawals,
  autopay and payment lines, dividends and reinvestments, buy/sell lines, and
  installment counters like `Monthly Installments (12 of 24)`. These collide constantly
  on capitalization alone and merging them changes nothing. Match the keyword anywhere
  in the name, not just at the start: a bank label like `eDeposit` carries a one-letter
  prefix and slips a start-anchored pattern, and it can rank first by dollars while
  being no merchant at all. The surer tell is the category spread. A real merchant sits
  in one or two categories, while a deposit or transfer label scatters across unrelated
  ones, so check the categories of any large group before trusting its name.
- **Groups whose transactions all sit in transfer-ish categories** (`Transfer`,
  `Credit Card Payment`, `Balance Adjustments`, `Buy`, `Sell`, `Investments`).
- **Store-number variants**, unless the user asks for them. A name carrying a branch
  number against the same name without one is a real split, but collapsing per-location
  records is a bigger decision than fixing a spelling, and some people want the
  locations kept apart.

Keep a group when the names are plainly one business and at least one variant moves a
total, judged in dollars rather than transaction count.

Pick the canonical name by **which spelling is still arriving**, not by which has the
most transactions. Providers rename merchants, so the bulk of the history often sits
under a name the bank stopped sending years ago; merging into that dead spelling leaves
every future charge to split off again. Compare the latest date per variant, and only
fall back to transaction count when they are still both in use. Among equals, prefer the
properly capitalized spelling with no store number or provider prefix.

State the last-seen date for each variant in the report. It is what justifies the
canonical choice when it contradicts the counts, and it also marks the groups where both
spellings are dormant, which are worth merging only for tidier history.

Flag separately, because it is the case with a real cost: a split where more than one
variant has a recurring stream. Check with `get_merchant(merchant_id)` on every variant
of the groups you keep, reading `recurring_stream`. `get_recurring_transactions` is not
enough: it left out `is_active: true` streams on merchants with
`has_active_recurring_streams: false`, so it can report one stream on a split that has
two.

## Report

Lead with the shape of it: transactions scanned, the date range they cover, distinct
merchant names, how many groups survived filtering, and how many transactions they
cover. Then one table, ordered by total dollars descending. Dollars, not transaction
count, is what decides whether a merge changes anything a reader cares about: a
41-transaction group worth $776 belongs below a 25-transaction one worth $8,690.

| Canonical | Variants (count, last seen) | Transactions | Total | Recurring? |

Name the noise you filtered out in one line, with its count, so the user can ask for it
if they disagree with the cut. Do not paste the unfiltered list.

If nothing survives filtering, say so in one line. A clean merchant list is a normal
result, not a failure to look hard enough.

## Apply

Only when the user asks, and only for groups they name or confirm. Merging is a write,
so it needs `MONARCH_MCP_READ_ONLY=0`. If the tools below are absent, that is why.
Say so and stop rather than working around it.

`merge_merchants(source_merchant_id=..., target_merchant_id=...)` is the "Merge & delete"
action in Monarch's app. It moves the source's transactions to the target and **deletes the
source**. It has no dry run and cannot be undone. Renaming with `update_merchant` is not a
shortcut: a name already in use fails with `A merchant with this name already exists`.

Work one group at a time, one source at a time:

1. Tell the user the source, the target, and how many transactions move, and get a yes.
2. Record what the merge might disturb: `get_merchant` on both for `transaction_count` and
   `recurring_stream`, `get_recurring_transactions`, and any `get_transaction_rules` entry
   naming the source.
3. Merge. If `success` is false, stop and report the result. Treat an error as unknown rather
   than failed, and re-read before retrying.
4. Confirm: `get_merchant(source)` returns not found, and the target's `transaction_count` grew
   by exactly the source's earlier count. Verified live: transactions keep their ids, dates,
   amounts, categories, and notes.

No merge with a rule or a recurring stream has been run through this tool, so what happens to
them is unverified. Compare against step 2 and tell the user what you find:

- **Recurring streams.** Re-read `get_recurring_transactions` and `get_merchant(target)`, and say
  if a subscription disappeared or now shows twice. If a stale stream remains, offer
  `review_recurring_stream(stream_id=..., review_status="ignored")`. Do not call it without asking,
  since it is a second write on a different object.
- **Rules.** Re-read `get_transaction_rules`. A rule still keyed to the old spelling will re-split
  future charges. A merchant rule using `eq` only matches that exact name, and switching it to
  `contains` on the distinctive part of the name covers every variant. `update_transaction_rule`
  is non-destructive: it re-reads the rule and merges your change over its current state, so
  passing only `merchant_criteria` keeps the existing category and tag actions.

Report what changed, per group, with the counts you verified. Never report a merge you
did not confirm with `get_merchant`.
