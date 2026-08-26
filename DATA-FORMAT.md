# DATA-FORMAT.md — Ledger JSON Schema

This is the contract between the monthly data-entry conversation (see
PROJECT-INSTRUCTIONS.md in your Claude Project knowledge — it isn't
committed to this repo) and the app (`personal-finance-tracker.html`).
Every field name here matches a variable name in the app's script
exactly — if you rename something here, rename it in the code too.

## Top-level keys

| Key | Behaviour | Feeds |
|---|---|---|
| `netWorth` | **Full replace** | Widget 01, Widget 06 (balances by label) |
| `netWorthHistory` | **Merge** — new months only | Widget 01 (delta), Widget 04 |
| `spend` | **Merge** — new months only | Widget 03, Widget 05 (outgoings) |
| `income` | **Merge** — new months only | Widget 05 |
| `story` | **Full replace** | Widget 02 |
| `actions` | **Full replace** (carry forward + amend) | Widget 02 |
| `savingsGoals` | **Full replace** (rare) | Widget 06 |
| `meta` | **Full replace** | Widget 03's partial-month note |

"Merge" means: only the months present in the incoming JSON are touched;
every other month already stored is left exactly as it was. This is
implemented in `mergeIncoming()` in `personal-finance-tracker.html` — see
that function if the merge behaviour ever needs to change.

Any top-level key not in this table is ignored (and called out in the
status message after **Apply Locally**) rather than erroring — a guard
against typos silently doing nothing.

---

## `netWorth`
```json
{
  "asOf": "2026-09-24",
  "assets": [
    { "label": "Investment Account", "amount": 3200.00 },
    { "label": "Everyday Saver", "amount": 1500.00 }
  ],
  "liabilities": [
    { "label": "Credit Card", "amount": 1300.00 }
  ],
  "pensions": {
    "pots": [
      { "label": "SIPP", "amount": 6000.00 }
    ]
  }
}
```
Full replace — include every asset/liability/pension pot each time, not
just the ones that changed. Labels must match exactly what's used in
`savingsGoals` (see below) and what you use to refer to each account.
`asOf` drives the "AS OF" label in the app header. Net worth is computed
as `assets + pensions − liabilities` — there's no property/mortgage tile
in this version.

## `netWorthHistory`
```json
{
  "months": ["2026-09"],
  "entries": {
    "2026-09": { "assets": 4700.00, "liabilities": 1300.00, "pensions": 6000.00 }
  }
}
```
Merge. Only send the new month(s). `assets`/`liabilities`/`pensions` here
are the **totals**, matching the sums in that month's `netWorth`. Widget 01
compares the latest two entries for its delta; Widget 04 charts the full
series. (The app derives `months` from `entries`' keys itself after
merging, so an incoming `months` array is read but not required — send it
anyway for readability, since **Copy Current Data as JSON** includes it.)

## `spend`
```json
{
  "months": ["2026-09"],
  "monthly": {
    "Fishing/Angling": { "2026-09": 45.20 }
  },
  "transactions": {
    "Fishing/Angling": {
      "2026-09": [
        { "date": "2026-09-05", "desc": "EXAMPLE MERCHANT LTD", "amount": 45.20, "source": "amex" }
      ]
    }
  }
}
```
Merge, by category and month. `monthly[category][month]` must equal the
sum of `transactions[category][month]`. `source` should identify which
account/card the transaction came from (`current_account`, `barclaycard`,
`amex`, etc.) — it's shown in the app's drill-down panel in small italics
for reconciliation.

Category names should be one of the fixed list below, grouped in
`SPEND_GROUPS` (hardcoded in `personal-finance-tracker.html`):
```
Set Spend:       Insurance, Phone/Mobile, Gym & Fitness, Subscriptions,
                 Investments, Pension Contributions, Bank Fees
Discretionary:   Groceries, Dining Out/Takeaway, Fuel/Transport,
                 Retail/Shopping, Fishing/Angling, Kids' Activities
One-Off:         Travel/Trips, Entertainment & Family
To Be Reviewed:  Cash Withdrawals, To Be Reviewed
```
A category outside this list still shows up — it lands in a synthetic
**Uncategorised** group at the bottom of Widget 03 rather than being
silently dropped — but it won't have a "proper" home until it's added to
`SPEND_GROUPS` in the code (a code change, not a data change).

Do not include Credit Card Payments, Loan Repayments, a Joint Household
Contribution, Curve flip-pairs, cross-statement duplicates, or
transactions that net off elsewhere — see "Exclusions from spend" in
PROJECT-INSTRUCTIONS.md.

(As with `netWorthHistory`, the app derives its month list from
`monthly`'s keys directly rather than trusting the `months` array on
input — include `months` for readability, it's just not load-bearing.)

## `income`
```json
{
  "months": ["2026-09"],
  "monthly": { "2026-09": 3500.00 }
}
```
Merge, by month. One figure per month — total income credited that month.
If a month's income hasn't landed yet by the cutoff date, omit it rather
than guessing; the app shows it as "pending" in Widget 05 rather than zero.

## `story`
```json
{
  "month": "2026-09",
  "narrative": [
    { "type": "positive", "text": "..." },
    { "type": "negative", "text": "..." },
    { "type": "neutral",  "text": "..." }
  ]
}
```
Full replace. Each entry renders as one bullet in Widget 02 with a
coloured dot — green for positive, red for negative, amber/grey for
neutral. `type` reflects whether the point is good, bad, or purely
informational, not the size of the number. Written by hand (or by Claude,
during the monthly update chat) — nothing here is computed in-browser.
See "Story of the Month — in-depth analysis" in PROJECT-INSTRUCTIONS.md
for the process to follow before filling this in each month.

## `actions`
```json
[
  { "text": "...", "raisedMonth": "2026-08", "done": false }
]
```
Full replace — include the complete list every time, it does not append.
Each month: carry forward existing open items, mark resolved ones
`done:true`, and only add new items where that month's analysis genuinely
surfaced something worth acting on. Rendered in Widget 02 as two
accordions, Open and Completed. (Copy the current list via **Copy Current
Data as JSON** first if you want to amend rather than rebuild it from
memory.)

## `savingsGoals`
```json
[
  { "label": "Everyday Saver", "purpose": "Emergency fund", "targetAmount": 4000, "targetDate": "2027-02-28" },
  { "label": "Fixed Saver", "purpose": "Fixed savings buffer", "targetAmount": 2000, "targetDate": null },
  { "label": "Stocks & Shares ISA", "purpose": "Long-term savings — no fixed target", "targetAmount": null, "targetDate": null }
]
```
Full replace, changes rarely. `label` must exactly match an asset label in
`netWorth.assets` — that's how Widget 06 pulls the current balance in.
`targetAmount`/`targetDate` both `null` = tracked balance-only, no fixed
goal. If `targetDate` is set, Widget 06 shows the monthly amount needed to
hit it on time.

## `meta`
```json
{ "currentAsOfDay": 25 }
```
Full replace. Day-of-month the current month's statements go up to —
surfaces as a "partial, to day N" note under Widget 03 once both `meta`
and that month's `spend` are present. Omit once a month closes out fully
(the note just won't show).

---

## Example monthly submission
A typical month only touches a handful of keys — everything else is left
alone by the merge:

```json
{
  "netWorth": { "asOf": "2026-09-24", "assets": [ /* full list */ ], "liabilities": [ /* full list */ ], "pensions": { "pots": [ /* full list */ ] } },
  "netWorthHistory": { "months": ["2026-09"], "entries": { "2026-09": { "assets": 4700.00, "liabilities": 1300.00, "pensions": 6000.00 } } },
  "spend": { "months": ["2026-09"], "monthly": { "...": { "2026-09": 0 } }, "transactions": { "...": { "2026-09": [] } } },
  "income": { "months": ["2026-09"], "monthly": { "2026-09": 3500.00 } },
  "story": { "month": "2026-09", "narrative": [ /* 3-6 bullets */ ] },
  "actions": [ /* full current list */ ],
  "meta": { "currentAsOfDay": 24 }
}
```
`savingsGoals` is omitted here since nothing changed that month — the app
leaves it exactly as it was.

## A safe monthly workflow

1. Paste the result into Widget 07, click **Apply Locally**, and eyeball
   the numbers before trusting them — the app recalculates everything
   live, so a wrong figure will usually stand out. Check the status box
   for any unrecognised keys it flagged.
2. Click **Save to Cloud** once it looks right.
3. If you ever want to sanity-check exactly what's currently stored,
   **Copy Current Data as JSON** in Widget 07 gives you the full current
   state, history included.

## A note on scale

The whole app state lives in one Firestore document, which has a 1MB
hard size cap. A year of personal transactions (a few hundred, each just
date/desc/amount/source) is only tens of KB, so this is in no danger for
normal use — worth keeping in mind only if you keep every itemised
transaction forever across many years. If it ever got close, the fix
would be dropping itemisation on old months while keeping the monthly
totals — the totals are what every calculation actually uses; individual
transactions only feed the drill-down panel.

## Known limitation: per-transaction recategorise/delete

Widget 03's drill-down panel has a "Move" / "Delete — nets off elsewhere"
UI on each transaction, but it's currently a **preview-only stub** — it
logs what it would do to the console and shows a note, but doesn't
change any data. To actually recategorise or remove a transaction today,
edit it in your source data and repaste the affected month's `spend` via
Widget 07.
