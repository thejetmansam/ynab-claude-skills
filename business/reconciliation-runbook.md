---
title: Business YNAB Reconciliation System
status: template
tags: [finance, ynab, reconciliation, profit-first, runbook]
---

# Business YNAB Reconciliation System

The system of record for keeping a company's YNAB truthful: every transaction
categorized, every account matched to the bank, and the Profit First allocation
running on real numbers.

Three artifacts, three jobs:

- **This runbook is the law.** Rules, mapping, protocols, invariants, log.
- **`skills/ynab-profit-first/SKILL.md` is the brain.** It reads and allocates.
- **`commands/ynab-reconcile.md` is the hands.** It cleans the books.

The skill reads this file every run. If the skill and this file ever disagree, **this
file wins.** Fork it and fill in the bracketed values. Once populated it describes your
actual finances, so treat the filled-in copy like any other financial record.

## 1. Strategic decision: repair in place, no Fresh Start

YNAB's own guidance says being months behind can justify a Fresh Start. **Do not take
it on a business budget.** A Fresh Start archives transaction history out of the live
plan, and on a business the history *is* the tax record.

YNAB's official middle path, **Plan Reset**, keeps transactions and reports but
re-zeroes every assignment. That flattens category balances you may badly need to keep;
a funded tax envelope is the obvious one.

**Repair in place**, using the sequence that actually works:

1. One account at a time.
2. Categorize everything first.
3. Then reconcile.
4. Then fix the plan: cover overspending, fund categories, assign RTA to zero, **in the
   current month only.** Never backfill past months.

## 2. Cadence

| Rhythm | When | What | Time |
|---|---|---|---|
| Weekly pass | A fixed weekday, never inside the quiet window | `/ynab-reconcile` WEEKLY: categorize and approve everything new, flag questions | ~15 min |
| Allocation day | `[your split days]`, or the prior business day if that falls on a weekend or in the quiet window | Weekly steps, then the split of period deposits, RTA to $0 | ~20 min |
| Month close | First week of the month, once statements post | Categorize then reconcile each account against its statement, then a human presses Reconcile in the UI | ~30 min |
| Quarterly | First business day of the quarter | Profit distribution decision, CAP ratchet review, subscription sweep, payee cleanup | ~60 min |

YNAB's guidance is to reconcile at least weekly. **Small habit-anchored sessions are
what prevent relapse**, not heroic monthly catch-ups. Profit First prescribes a
twice-monthly allocation rhythm. If your bookkeeper already runs different days, keep
theirs: an existing habit beats a textbook date.

## 3. The categorization rulebook

### Confidence tiers

- **HIGH.** In the payee mapping table, or an exact payee-history match with a
  consistent category. Approve in bulk.
- **MEDIUM.** Unknown payee, identified by research. Propose with a one-line reason.
  Spot-check before approving.
- **LOW.** Fits two categories, amount far outside normal, or matches a personal-spend
  rule. One clarifying question with two or three candidates. **Never guess past LOW,
  never auto-approve it.**

### Payee mapping

Keep a living table here. Extend it on every run.

| Payee pattern | Category | Tier | Note |
|---|---|---|---|
| `[processor]` | Income | HIGH | Inflows only. If a lender skims deposits, see the financing gross-up in section 5 |
| `[ad platform]` | Advertising | HIGH | |
| `[marketplace / everything-store]` | inspect | LOW | Never blanket-map. See rule 3 below |
| `Transfer : *` | none | n/a | Transfer legs never take a category |

### Hard rules

1. **Credit card payments are transfers**, never categorized spending.
2. **Refund inflows get the ORIGINAL purchase's category**, matched to the source
   transaction, with an audit memo. Payee memory will assign the wrong one.
3. **Everything-store payees are inspected per transaction**, never blanket-mapped.
   Only let payee memory auto-categorize payees that are roughly 90% consistent to one
   category. Processors and multi-purpose merchants stay inspect-every-time.
4. **Categorize an account completely before reconciling it.** Reconciled rows lock:
   category edits stay safe afterward, amount and date edits do not.
5. **Assignments change in the current month only.** Past months' assignments are
   read-only history. Categorization fixes to past transactions are allowed and are the
   entire point of cleanup.
6. **Propose, wait, write.** No mutation without approval in-session. Bulk writes via
   `ynab_update_transactions_bulk`.
7. **Rename and consolidate payees in the YNAB UI** as names stabilize. Renaming rules
   plus payee memory then auto-categorize future imports. The API cannot rename payees,
   so queue these as UI tasks.
8. Honor the quiet window, if one is set.

## 4. Reconciliation procedure

Per account, in a fixed order. Operating checking first, then cards.

1. Get the statement for the period.
2. Confirm zero uncategorized or unapproved transactions in that account for the
   statement period (rule 3.4).
3. Compare the statement ending balance to the YNAB **cleared** balance as of the
   statement date. Pending items are excluded by definition.
4. If they differ, multiset-diff the statement lines against the YNAB register. Hunt in
   this order: missing transactions, posted-but-uncleared, phantom or stale transfers,
   duplicates. Fix with `ynab_create_transaction`, `ynab_delete_transaction`, or a
   `cleared` update, each proposed before applying.

   **The diff technique that works:** export the account's activity, then fuzzy-match
   on **amount with a plus or minus 7 day window.** Never match on (date, amount)
   exactly. Card issuers report the transaction date and YNAB imports the posting date;
   a strict date match **turns a couple of real differences into hundreds of false
   ones.** Also mind the sign flip: card exports list charges positive, YNAB stores
   them negative.

5. **Adjustments: hunt, do not adjust.** On business books a cash-account adjustment
   dumps into RTA and pollutes the income and spend reports that back the tax return.
   Two exceptions:
   - **Exception A, first-ever reconcile of an account.** Take one baseline adjustment
     rather than a multi-month forensic hunt. This is YNAB's own guidance for large
     gaps. Date it, memo it "initial reconciliation baseline", treat prior history as
     best effort.
   - **Exception B, ongoing months.** A residual under $10 after a full hunt.
6. Match to the penny, then **a human presses Reconcile in the YNAB UI.** The formal
   lock is UI-only; no API endpoint exists. It also arms YNAB's 3-day anti-duplicate
   import window.
7. Log it in section 7.

## 5. The allocation flow

Fill in from your own Profit First worksheet. **The worksheet is the source of truth,
not this file and not the skill.**

```
TOTAL INCOME = all deposits since the last split
             = RTA inflows
             + customer payments received outside the processor
             + any active financing gross-up

Profit       = INCOME x [current Profit CAP]
Owner's Comp = [fixed scheduled amount]   (surplus -> Owner's Comp Hold)
Tax          = INCOME x [tax rate]
OpEx         = remainder

The four lines must sum back to INCOME. If they do not, stop.
```

**Financing gross-up.** Only if you use revenue-based financing that takes a cut of each
deposit before it reaches you. Add the withheld amount back to income so the split runs
on what the business actually earned.

**Landing order.** Profit to the profit envelope. Tax to taxes. Owner's Comp to the pay
categories, surplus to the Hold. OpEx to the card payment category first until Available
equals the card balance, then fixed bills, then ads, leftover to an OpEx buffer.

**Ratchet, never jump.** Start the Profit CAP low enough that it does not hurt, and
step it up one notch at each quarter start. Never ratchet backward.

**The distribution is a real transfer.** At each quarter start, profit leaves the bank.
This is the step operators skip, and skipping it turns the whole system into a savings
account with extra steps.

**Never cut owner pay to make the math close.** When OpEx does not fit, cut OpEx.

### Standing protocol notes

Document your own here. The two that most often get mis-booked:

- **A shareholder loan is a receivable**, never an expense or a draw. Recurring
  transfers to an owner's personal account are a draw, and they are a different line.
  Conflating them misstates the balance sheet and the allocation base.
- **Personal spend from business accounts gets flagged for the owner to tag**, never
  classified on their behalf. That line is a tax position, not a categorization guess.

## 6. Health invariants

Checked every run. Flag only what fails.

1. Uncategorized transactions older than 7 days: **0**.
2. After each allocation: Ready to Assign (`to_be_budgeted`) = **$0** in the current month.
3. Card payment category Available = the card's balance. **The paid-in-full invariant.**
4. **The trust check:** total Available across categories, plus RTA, equals total
   on-budget cash. **To the penny.** This is the cheapest correctness check you have.
5. Every account reconciled through the prior statement month.
6. Every personal-spend flag resolved by the owner within one weekly cycle.

## 7. Run log

One line per run that writes. This is what makes drift visible.

| Date | Mode | Categorized | Pending | RTA | Reconciled thru |
|---|---|---|---|---|---|
| | | | | | |

## 8. The one-time catch-up

If you are adopting this on a budget that has fallen behind, do it in one sitting with
two approval gates:

1. **Gate A, structure.** Create the groups and categories the mapping needs.
2. **Gate B, the backlog.** Categorize everything per section 3. Personal items get
   flagged, not classified. One bulk write after approval.
3. Fund the card payment category to the full card balance. Assign remaining RTA per
   your CAPs, **current month only.**
4. Month-close reconcile every account. First-ever baseline adjustment allowed
   (Exception A). A human presses Reconcile.
5. Queue the UI-only tasks: payee renaming rules, the Reconcile presses.

After the catch-up, the system runs on the section 2 cadence.
