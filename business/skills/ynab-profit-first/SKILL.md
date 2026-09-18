---
name: ynab-profit-first
description: A daily business financial review. Michalowicz's Profit First money system running on top of YNAB. Use whenever the owner wants to look at business finances: "run the business money review", "how is the business doing", "where's the money", "morning financial review", "run profit first", "run the split", "is profit funded", "run the allocation", "did we clear the card", or on an allocation day. Two modes: a fast daily status-and-allocate (LIGHT) and a full Instant Assessment / TAP-vs-CAP diagnostic (DEEP). This is the BUSINESS counterpart to the personal ynab-coach skill; the two are deliberately separate.
---

# YNAB Profit First

The business counterpart to the personal `/ynab-daily`. It answers "where does the
business money stand, and what should move", and on allocation days it runs the
Profit First split.

Three pieces, three jobs. This skill is the read-and-allocate brain.
`/ynab-reconcile` is the books-cleaning hands. The runbook is the law.

**Read `business/reconciliation-runbook.md` first, every run.** It holds the
allocation formula, the current CAPs, the special protocols, the health invariants,
and the run log. This skill is only the run procedure. Never re-derive a rule that
lives in the runbook, and if the two disagree, **the runbook wins.**

## Configuration

Fill these in for the company this skill serves. Nothing here should be hardcoded
into the skill body:

- `BUDGET_ID`: the YNAB budget id, or `"last-used"`.
- `CHECKING`, `CHARGE_CARD`: the operating account and the card that autopays.
- `ALLOCATION_DAYS`: the split rhythm. The book uses the 10th and 25th; keep an
  existing rhythm if one already runs.
- `QUIET_WINDOW`: optional. A recurring period when no finance work happens.
- `REVENUE_SOURCE`: optional, for DEEP mode. Wherever real revenue lives outside YNAB:
  a data warehouse, your payment processor's reports, or your accounting system.

## The two gates that never move

1. **The quiet window**, if one is set. No finance work inside it. If it is that
   window, say so and stop. Do not negotiate.
2. **Propose, wait, write.** Never mutate YNAB without the owner's approval in the
   same session. Every allocation, every transaction, every category move is proposed
   as plain text and applied only on an explicit yes. Reads are free; writes are gated.

## Pick the mode

- **LIGHT** (default, any morning): YNAB only, about 3 minutes. Daily money status,
  the health invariants, and, if it is an allocation day or cash is sitting
  uncommitted in Ready to Assign, the proposed split.
- **DEEP** (on demand, weekly, month close, quarter start): LIGHT plus the full
  Instant Assessment, with revenue from `REVENUE_SOURCE`. Run it when the owner asks
  for the full analysis, at month close, at a quarter start for the distribution and
  the ratchet, or when a number looks wrong and you need the real revenue picture.

If the owner does not specify, run LIGHT and offer DEEP at the end.

## LIGHT mode

### 1. Pull state (parallel, read-only)

- `ynab_get_month(month="current")` for RTA, category balances, activity, assigned
- `ynab_list_accounts()` for cleared versus uncleared and balances
- `ynab_list_categories()` for exact names and ids. Never hardcode or guess an id.
- `ynab_list_transactions(transaction_type="uncategorized", limit=40)` and
  `(transaction_type="unapproved", limit=40)`, only to detect dirty books

### 2. Dirty-books check (hand off, do not fix here)

**Transfer legs correctly show as uncategorized. Exclude them.** Payees beginning
`Transfer :`, meaning the card autopay pairs and payment legs, never take a category.
They are not dirty books.

Count only real uncategorized *spend* and *unapproved imports*. If those exist, say so
plainly and tell the owner to run `/ynab-reconcile` first. **An allocation on top of
dirty books allocates fiction.** Unapproved imports are usually already categorized by
payee memory and only need approval, but still hand off: approval and personal-flagging
belong to reconcile, not here. Do not categorize in this skill.

### 3. Read the money and check the invariants

Compute these, and for the invariants flag only what FAILS:

- Checking cleared balance.
- Card balance and the **payment-category gap**: the card's payment category Available
  against the card balance. That is the paid-in-full invariant. Report the next autopay
  date and whether the gap closes before it.
- **RTA is the month's `to_be_budgeted` value.** It should be zero after the last
  allocation; a positive RTA on a non-allocation day is uncommitted cash worth
  surfacing. Do NOT read the "Inflow: Ready to Assign" category *balance*: it carries
  a lifetime artifact that can grow enormous and is not spendable cash.
- Profit reserved against the current CAP target. Tax reserved.
- Any red or overspent categories. A category the owner is deliberately holding red is
  a decision, not a bug. Name it, do not auto-fix it.
- Personal-spend items awaiting the owner's own tag. List them, never classify them.

### 4. Allocation trigger

Run the split if **today is an allocation day** OR **RTA is above zero with real
uncommitted income**. Otherwise report status and stop.

### 5. LIGHT output

Plain text, scannable, honest:

```
BUSINESS MONEY - YYYY-MM-DD - LIGHT
STANDING
  Checking $X cleared - Card -$Y (payment gap $Z, autopay MMM D)
  RTA $X - Profit $X (target $Y) - Tax reserved $X
INCOMING / UNCOMMITTED
  $X in RTA since last split
NEEDS YOU (N)          [omit the block entirely if none]
  - ...
HEALTH - failures only [omit the block entirely if all pass]
  - ...
[ALLOCATION DUE -> see proposed split below]   [only when triggered]
```

Challenge, do not flatter. If nothing needs action, say the business money is quiet
and stop.

## The allocation engine

Propose, approve, apply, log. **Never re-encode the formula or the CAPs from memory.**
Read them live from the runbook each run: CAPs ratchet over time and owner
compensation changes.

1. **TOTAL INCOME** equals all deposits since the last split: RTA inflows, plus any
   customer payments received by bank transfer, plus the financing gross-up if you use
   revenue-based financing (defined in the runbook).
2. **Compute the split:** Profit equals income times the Profit CAP. Owner's Comp is
   the fixed scheduled amount, with any surplus going to an Owner's Comp Hold. Tax
   equals income times the tax rate. OpEx is the remainder. **The four lines must sum
   back to income.** If they do not, stop and say so.
3. **Propose the landing**, in the runbook's priority order: Profit to the profit
   category, Tax to taxes, Owner's Comp to the pay categories then the Hold, OpEx to
   the card payment category first until Available equals the card balance, then fixed
   bills, then ads, and any leftover to an OpEx buffer. Show it as a table that foots
   to income, and name what each move accomplishes.
4. **Apply on approval**, current month only, via `ynab_update_month_category`.
   Never touch past-month assignments.
5. **Verify the hard checks:** RTA returns to zero, and the card payment category
   Available covers the full card balance. Flag loudly if either fails.
6. **Record protocol entries when due**, propose-then-apply like everything else.

**Quarter-start extras** (Jan, Apr, Jul, Oct 1): the profit distribution is a real
transfer out of the bank, not a budget move. Many operators run Profit First for years
and never actually take the distribution, which defeats the point. Also ratchet the
Profit CAP one step at the quarter start, **never backward.**

## DEEP mode, the Instant Assessment

LIGHT plus the real diagnostic. Pull revenue from `REVENUE_SOURCE`; do not eyeball it
from YNAB. If `REVENUE_SOURCE` is not set, say so, and use YNAB inflows labeled as an
approximation.

1. **Real Revenue** over the window the owner names, defaulting to the trailing 12 full
   months, the book's Instant Assessment window. Revenue from outside YNAB is usually
   gross of processor fees and any financing deductions, so it runs above YNAB's net
   deposits. Say so rather than reconciling the two silently.
2. **The Instant Assessment** (Profit First ch. 3): the four buckets as actual
   percentages of Real Revenue, against both the current CAPs and the book's Target
   Allocation Percentages for the revenue band. Percentages are downstream of revenue:
   the fixed-dollar buckets heal as revenue recovers, so frame it that way rather than
   as a spending problem.
3. **The levers:** where the gap actually is, the organic versus paid revenue mix, the
   collection rate on booked revenue, and the position against break-even and against
   fully-funded Profit First.
4. **Findings and what needs the owner**, ranked, each with a status: fixed, watch, or open.
   Is a distribution due? Is a ratchet due? Is the Owner's Comp Hold funded to one month?
5. Offer to refresh the written Profit First review if the numbers have moved materially.

Paraphrase the book, cite chapters, never quote it at length.

## Logging

Append one line to the runbook's run log for **every run that writes**:

```
| YYYY-MM-DD | LIGHT/DEEP | categorized=n/a (see reconcile) | pending=N | RTA=$X | reconciled-thru=YYYY-MM |
```

For allocation runs also log the split total, the Profit amount taken, any category
held red by decision, and the card gap before and after. A run that only reads and
proposes nothing needs no log line. A run that writes always does.

## Standing rules

Honor these. Do not restate them to the owner every run.

- **A shareholder loan is a receivable, never an expense or a draw.** Recurring
  transfers to an owner's personal account are a draw, and they are a different line.
  Confusing the two misstates both the balance sheet and the allocation base.
- **Personal spend from business accounts gets flagged for the owner to tag**, never
  classified on their behalf.
- **Never cut owner pay to make the math close.** When OpEx does not fit, cut OpEx.
  That is the entire point of the system.
- Never send money-related messages to customers from this flow.
- Plain text. No emojis. Nothing that would embarrass the owner if forwarded.
