---
name: monarch-categorization-review
description: Review how recent Monarch Money transactions are categorized. Finds uncategorized transactions, flags likely miscategorizations, and proposes auto-categorization rules to stop them recurring. Use when the user asks to review categories, clean up categorization, find uncategorized or miscategorized transactions, or asks why spending in a category looks wrong.
license: MIT
allowed-tools: mcp__monarch-money__get_transaction_categories mcp__monarch-money__get_transaction_rules mcp__monarch-money__get_transactions mcp__monarch-money__get_transactions_needing_review mcp__monarch-money__get_transaction_details mcp__monarch-money__search_transactions
---

# Categorization review

Read-only. You cannot recategorize anything or create rules — the server withholds
mutating tools. Produce findings the user applies in Monarch's UI.

## Gather

1. `get_transaction_categories` — the full category list with ids. Note the id of
   `Uncategorized`, and keep the list to hand so every suggestion names a category
   that actually exists.
2. `get_transaction_rules` — existing auto-categorization rules. A rule you propose
   that already exists is noise, and an existing rule may itself be the cause of a
   miscategorization.
3. Uncategorized transactions: `get_transactions(category_ids=["<Uncategorized id>"],
   start_date=..., end_date=...)`.
4. Recent transactions for the review window: `get_transactions(start_date=...,
   end_date=...)`, default the last 30 days unless the user says otherwise.

Default to the last 30 days. Ask only if the user's phrasing is genuinely ambiguous
about the period.

## Gotchas

- **`get_transactions` needs both dates.** Passing only `start_date` returns
  `{"error": true, "message": "You must specify both a startDate and endDate..."}`.
- **Do not use `get_transactions_needing_review(uncategorized_only=True)` to find
  uncategorized transactions.** That filter is applied locally to one fetched page,
  so it returns `count: 0, truncated: true` even when plenty exist. Filter by the
  `Uncategorized` category id instead. `get_transactions_needing_review` is still
  the right tool for Monarch's own review queue.
- **Amounts are signed**: expenses negative, income positive. A positive amount in an
  expense category is usually a refund, not a miscategorization — check before flagging.
- Paging is by `limit`/`offset`, default limit 100. If a result looks truncated, page
  until exhausted rather than reporting the first page as the whole answer.

## Judge

For each uncategorized transaction, propose a category from the real list, based on
merchant, description, amount, and how similar past transactions were categorized.
Say when you are guessing.

For miscategorization, the signal is inconsistency, not your opinion of where a
merchant belongs:

- The same merchant categorized differently across transactions — one of them is wrong,
  and the majority treatment is usually right.
- A transaction that does not fit its category's pattern: a $900 charge in Coffee Shops,
  a grocery-store charge in Home Improvement.
- Transfers and credit card payments categorized as spending, which double-counts them.
- Anything in a catch-all (`Miscellaneous`, `Other`, `Check`, `Cash & ATM`) that has an
  obvious real category.
- Business vs personal categories on the same merchant, where the split looks accidental.

Do not flag a merchant that legitimately spans categories (Amazon, Costco, Target)
just for spanning them. Flag it only when a specific transaction is clearly wrong.

## Propose rules

Only where a rule would have prevented a real finding above and no existing rule covers
it. A good rule is a merchant pattern that maps to one category with no exceptions in
the data you just read.

State each as: merchant/description match → category, plus how many transactions in the
window it would have fixed. Note that the user creates these in Monarch under
Settings → Rules, and that a rule applies going forward — existing transactions still
need recategorizing by hand or by Monarch's "apply to existing" option.

Skip rules for merchants that legitimately vary by transaction. A wrong rule is worse
than no rule: it miscategorizes silently, forever.

## Report

Lead with counts: how many transactions reviewed, how many uncategorized, how many
flagged. Then:

1. **Uncategorized** — date, merchant, amount, account, suggested category.
2. **Likely miscategorized** — same fields plus current category and why it looks wrong.
3. **Suggested rules** — pattern → category, transactions fixed.

Order each list by dollar amount, largest first — that is the order in which fixing
them changes the user's reporting. Tables are fine; keep them scannable. If a section
is empty, say so in one line rather than padding it.
