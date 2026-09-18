# /ynab-reconcile

The business YNAB maintenance pass: categorize the backlog, verify account truth
against the bank, and on allocation days run the Profit First split. This is the
business counterpart to the personal `/ynab-daily`.

**Read `business/reconciliation-runbook.md` FIRST.** It is the system of record: the
payee mapping table, the hard rules, the special protocols, and the confidence tiers
all live there. This command is only the run procedure.

## Modes

Pick by calendar, or by what the owner asks for:

- **WEEKLY PASS** (default, any day): steps 1 to 4. Target: under 15 minutes.
- **ALLOCATION DAY** (the configured split days, or the prior business day if that
  falls on a weekend or inside the quiet window): steps 1 to 5.
- **MONTH-CLOSE RECONCILE** (first week of the month, once statements post): steps 1
  to 4 plus step 6 for the prior month.

## 1. Pull state

In parallel:

- `ynab_list_transactions(transaction_type="uncategorized", limit=80)`
- `ynab_list_transactions(transaction_type="unapproved", limit=80)`
- `ynab_list_categories()` for exact names and ids. Never guess.
- `ynab_get_month(month="current")` for RTA and allocation state
- `ynab_list_accounts()` for balances, cleared versus uncleared, last reconciled date

Cap at 80 to stay under the 200 requests per hour limit. Note in the summary if older
items remain.

## 2. Categorize per the mapping table

Apply the runbook's payee mapping section. For each uncategorized or unapproved item:

1. **Mapped payee**, in the table or an exact payee-history match: assign, mark HIGH.
2. **Unknown payee**: search the name, identify the business type, propose the closest
   category, mark MEDIUM with a one-line reason.
3. **Ambiguous or out of pattern**: fits two categories, amount far outside normal, or
   matches a personal-spend rule. Mark LOW, queue one clarifying question with two or
   three candidates. **Never guess past LOW.**

Always check for: transfer pairs (autopay legs match each other and never take a
category), refunds (match the original purchase's category, not payee memory),
duplicates within 48 hours, customer payments received by bank transfer (income, with
the customer reference in the memo), and anything touching an owner-draw account or an
active financing protocol.

## 3. Propose

Plain text, three blocks, then wait:

```
HIGH CONFIDENCE (N) -- approve all?
  YYYY-MM-DD  Payee  $amount  -> Category
MEDIUM CONFIDENCE (N) -- spot-check then approve?
  YYYY-MM-DD  Payee  $amount  -> Category  (reason)
NEEDS CLARIFICATION (N)
  YYYY-MM-DD  Payee  $amount
    Q: [a] or [b]?
```

## 4. Apply

On approval, one `ynab_update_transactions_bulk` call carrying `category_id`,
`approved: true`, and a memo only where research added context. Confirm the result
count matches the approved count. Never write anything that was not approved in this
session.

## 5. Allocation day only: run the split

The formula and landing rules live in the runbook. Read them live; do not reproduce
them from memory, because the CAPs ratchet.

1. **TOTAL INCOME** equals all deposits since the last split: RTA inflows, plus
   customer payments categorized in step 4, plus any active financing gross-up.
2. **Run the formula:** Profit equals income times the current Profit CAP. Owner's
   Comp is the fixed scheduled amount. Tax equals income times the tax rate. OpEx is
   the remainder. The four must sum back to income.
3. **Propose the landing:** Profit to the profit envelope, Tax to taxes, Owner's Comp
   to the pay categories with surplus to the Hold, OpEx to real spending categories in
   priority order (card payment first, then fixed, then variable), leftover to the
   OpEx buffer. On approval apply with `ynab_update_month_category`, **current month only.**
4. Record any active protocol entries per the runbook.
5. **Verify:** RTA returns to $0 and the card payment category covers the full card
   balance. Both are hard checks. Flag loudly if either fails.

## 6. Month-close only: reconcile each account

For every real account in the budget:

1. Get the statement for the period.
2. Compare the statement ending balance to the YNAB **cleared** balance as of the
   statement date.
3. If they differ, multiset-diff the statement lines against the YNAB register. Look
   for, in this order: missing transactions, posted-but-uncleared items, phantom or
   stale transfers, duplicates. Fix with `ynab_create_transaction`,
   `ynab_delete_transaction`, or a `cleared` update, each proposed before applying.
4. Only if a residual difference survives the diff **and is under $10**, propose a
   reconciliation adjustment. **Anything larger gets hunted, not adjusted.** A plug
   entry hides the actual problem and it will resurface next month, bigger.
5. When it matches to the penny, tell the owner to press Reconcile in the YNAB UI. The
   formal lock is UI-only and no API can do it. Then log the reconcile.

## 7. Summarize and log

Under 100 words, plain text: categorized N, pending clarification N, RTA $X, card
funded yes or no, accounts reconciled through [month]. Append one line to the runbook's
run log:

```
| YYYY-MM-DD | mode | categorized=N | pending=N | RTA=$X | reconciled-thru=YYYY-MM |
```

## Hard rules

- Propose, wait, write. No mutation without approval in-session.
- Historical months' **assigned amounts are read-only.** Categorization fixes to
  history are the whole point of cleanup and are allowed; allocation changes are not.
- A shareholder loan is a receivable, never an expense or a draw. Recurring transfers
  to an owner's personal account are a draw, and they are a different line.
- Personal spend from business accounts gets flagged and listed in the summary for the
  owner to tag. Do not classify it for them.
- Never send money-related messages to customers from this flow.
- Plain text summaries. No emojis.
- Honor the quiet window, if one is set.
