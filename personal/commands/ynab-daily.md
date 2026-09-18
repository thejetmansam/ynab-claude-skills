# /ynab-daily

The daily personal categorization pass. Keeps transactions flowing through the four
rules without a backlog, then sends a short summary wherever you read notifications.

Read `personal/skills/ynab-coach/SKILL.md` first for the framework grounding, the
household's money model, and the propose-then-write pattern. This command is only the
run procedure.

## 1. Pull state

- `ynab_list_transactions(transaction_type="uncategorized", limit=60)`
- `ynab_list_transactions(transaction_type="unapproved", limit=60)`
- `ynab_list_categories()` for live names, ids, and balances
- `ynab_list_payees()` for payee history, used to infer the past category
- `ynab_get_month(month="current")` for Ready to Assign and the Age of Money snapshot

Cap at 60 to stay well under the 200 requests per hour limit. If more remain, say so
in the summary rather than silently truncating.

## 2. Categorize each transaction

1. **Known payee.** An exact match in payee history with a clear past category, or a
   recognizable national brand. Assign the most recent category, mark HIGH.
2. **Unknown payee.** An unfamiliar LLC or merchant code with no history. Search the
   payee name, identify the business type, assign the most probable category, mark
   MEDIUM so it gets a spot-check. Cite the finding: "[Payee]: search shows
   [business type], so [Category]".
3. **Genuinely ambiguous.** Fits two categories, or the amount is far outside normal.
   Mark LOW and queue one clarifying question with two or three candidate answers.
   **Never guess past LOW.**

Also flag:
- **Transfer legs.** Payees starting `Transfer :` need approval, never a category.
- **Refunds.** Match to the original purchase's category. Payee memory will assign the
  wrong one, and a refund landing in a low-traffic category can hide for months.
- **Duplicates.** Same date, amount, and payee within 48 hours.
- **Earmarked inflows.** A categorized inflow never shows up in an uncategorized-only
  pull. Check `Inflow: Ready to Assign` separately or you will miss it.

## 3. Show the proposal

Plain text, three blocks, no tables, no bold, no emojis:

```
HIGH CONFIDENCE (N) -- approve all?
  YYYY-MM-DD  Payee  $amount  -> Category

MEDIUM CONFIDENCE (N) -- spot-check then approve?
  YYYY-MM-DD  Payee  $amount  -> Category  (reason)

NEEDS CLARIFICATION (N)
  YYYY-MM-DD  Payee  $amount
    Q: Was this [a] or [b]? Or something else?
```

Wait for a reply.

## 4. Apply approved updates

Build a **single** `ynab_update_transactions_bulk` call covering every approved update.
One call handles roughly 100 transactions and keeps you under the rate limit; a loop of
single updates will exhaust it on a real backlog.

Each row carries `transaction_id`, `category_id` resolved from the live category list,
`approved: true`, and a `memo` only where research added context worth keeping.

Confirm the result count matches what was approved.

## 5. Daily summary

Under 100 words, plain text, no markdown:

- Categorized: N transactions across M categories
- Still pending clarification: N, listing payees if 5 or fewer
- Ready to Assign: $X
- Age of Money: N days, with a one-line read. Under 7 is paycheck to paycheck, 7 to 29
  is building a cushion, 30 or more is Rule 4 healthy.
- Top overspent category, if any
- One framework-anchored next move: "Rule 2: top up [category] $X so the next
  surprise does not break the budget"

Deliver it however you already get notifications. A Telegram bot works well:

```bash
curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d chat_id="${TELEGRAM_CHAT_ID}" \
  --data-urlencode text="${SUMMARY}"
```

Keep `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in the environment or a secret
manager, never in the repo. If they are unset, print the summary to stdout and say so
rather than failing the run.

## 6. Log the run

Append one line to your budget log, outside version control:

```
YYYY-MM-DD  categorized=N  pending=N  RTA=$X  AoM=Nd
```

That rolling history is what makes Age of Money trackable over time. YNAB's own Age of
Money number is unreliable on a budget that is still being cleaned up, so the proxy in
the skill plus this log is the more honest measure.

## Hard rules

- Hold the household's `MONEY_MODEL`. Never reopen it.
- Never auto-approve LOW confidence.
- Never write to YNAB without confirmation in the session.
- Cite the framework briefly when a rule is in play. No lectures.
- Honor the quiet window, if one is set. If it is that window, stop and say so.
