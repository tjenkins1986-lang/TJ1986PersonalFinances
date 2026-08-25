# Monthly Snapshot — JSON Format Reference

This is the exact shape the app expects when you paste a snapshot into
**Widget 07 · Monthly Data Update**. Give this file to whatever process
generates your monthly JSON (e.g. a Claude conversation fed your bank
statement) so its output loads cleanly.

## How loading works

- Paste JSON into the Widget 07 box, click **Apply Locally** to preview
  in your browser only, then **Save to Cloud** to make it live.
- **Partial updates are fine, and work two different ways depending on
  the key:**
  - `spend` and `netWorthHistory` are **merged**, not replaced. Include
    only the new month's data and it's added alongside everything
    already there — existing months and categories you don't mention are
    left untouched. You never need to resend the full history.
  - Every other key (`netWorth`, `savingsGoals`, `story`, `actions`,
    `meta`) is a **full replace** if included at all — it represents
    current state, not a history, so send the complete current value of
    that key, not a delta. A typical monthly update only needs to
    include `netWorth`, `spend`, `netWorthHistory`, `story`, `actions`,
    and `meta`; `savingsGoals` changes rarely enough to just omit.
  - Any key you omit entirely is left exactly as it was.
- Every number is a plain JSON number — no `£` signs, no commas
  (`1234.56`, not `"£1,234.56"`).
- Every month is `"YYYY-MM"` (e.g. `"2026-09"`). Every date is
  `"YYYY-MM-DD"` (e.g. `"2026-09-03"`).
- The **latest** month present in `spend.months` after merging is treated
  as "the current month" throughout the app (drives the 6-month rolling
  average window in Widget 03, etc.), so just include the new month you
  want to add — no need to reorder or resend past ones.

## Top-level keys

```json
{
  "netWorth": { ... },
  "spend": { ... },
  "savingsGoals": [ ... ],
  "story": { ... },
  "actions": [ ... ],
  "netWorthHistory": { ... },
  "meta": { ... }
}
```

### `netWorth`
```json
{
  "asOf": "2026-09-24",
  "assets": [
    { "label": "Santander 1/2/3 Account", "amount": 120.50 }
  ],
  "liabilities": [
    { "label": "Barclaycard", "amount": 1240.50 },
    { "label": "Car Loan — MotoNovo", "amount": 6800 }
  ],
  "pensions": {
    "pots": [ { "label": "Workplace Pension", "amount": 316000 } ]
  }
}
```
Asset `label`s matter beyond display: **Widget 04's savings goals link to
an asset by matching its `label` exactly** (see `savingsGoals` below), so
keep account labels stable month to month rather than renaming them.
`pensions.pots` is a flat list — add one entry per pension pot/provider,
no split between people.

`liabilities` is where **loans and credit card balances live** — each
entry is `{ "label": "...", "amount": ... }`, same shape as `assets`.
These are subtracted from net worth, and shown in their own tile in
Widget 01. Like asset labels, keep them stable month to month. This
version of the app doesn't track a mortgage/property tile at all — net
worth is just assets plus pensions minus liabilities.

### `spend`
**Merged, not replaced** — a monthly update only needs the new month:
```json
{
  "spend": {
    "months": ["2026-09"],
    "monthly": {
      "Groceries": { "2026-09": 812.40 },
      "Mortgage/Loan": { "2026-09": 1308.57 }
    },
    "transactions": {
      "Groceries": {
        "2026-09": [
          { "date": "2026-09-06", "desc": "CARD PAYMENT TO TESCO STORES", "amount": 12.89 }
        ]
      },
      "Mortgage/Loan": {
        "2026-09": [
          { "date": "2026-09-02", "desc": "DIRECT DEBIT PAYMENT TO NATWEST BANK", "amount": 1308.57 }
        ]
      }
    }
  }
}
```
Only mention the categories that actually had spend that month — any
category you don't include is left exactly as it was. If you need to
correct a past month (e.g. a miscategorised transaction from July),
include that month's key again under the relevant category — it
overwrites just that category/month combination, nothing else.

The full shape, for reference (this is what **Copy Current Data as
JSON** will show you, and what a from-scratch `spend` object looks
like):
```json
{
  "months": ["2026-01", "2026-02", "...", "2026-09"],
  "categories": [
    "Mortgage/Loan", "Car Finance", "Credit Card", "Loan Repayment",
    "Council Tax", "Utilities",
    "TV/Subscriptions", "Insurance", "Groceries", "Fuel/Transport",
    "Dining Out/Takeaway", "Entertainment & Family Activities",
    "Kids' Activities", "Household Maintenance & Contractors",
    "Retail/Shopping", "Travel/Trips", "Cash Withdrawals", "Bank Fees",
    "Personal Transfer – Discretionary"
  ],
  "monthly": {
    "Groceries": { "2026-01": 725.44, "2026-02": 945.99 }
  },
  "transactions": {
    "Groceries": {
      "2026-01": [
        { "date": "2026-01-06", "desc": "CARD PAYMENT TO TESCO STORES", "amount": 12.89 }
      ]
    }
  }
}
```
**The category names above are fixed** — they're hardcoded into the
app's `SPEND_GROUPS` (the Mortgage / Ongoing Monthly / One-Offs / Needs
Further Analysis grouping in Widget 03). A category name that doesn't
match one of these exactly won't crash anything, but it also **won't
appear anywhere in Widget 03** — it'll just be silently invisible. If you
introduce a genuinely new category, it needs to be added to
`SPEND_GROUPS` in `index.html` (a code change, not a data change) — ask
for that separately rather than inventing a new category name in the
JSON.

`Credit Card` and `Loan Repayment` track the **monthly payment amount**
(same idea as `Car Finance`) — this is spend, separate from the
**balance owed**, which lives in `netWorth.liabilities` instead. Both
usually change together each month (a payment reduces the balance) but
they're two different numbers in two different places.

For each category, `monthly[category][month]` should equal the sum of
`transactions[category][month][].amount` for that same month — the
`monthly` figure is what every calculation actually uses; `transactions`
is only used to populate the click-to-drill-down modal. If they don't
match, the totals shown won't match what the modal shows when you click
into them.

### `savingsGoals`
```json
[
  {
    "label": "Everyday Saver 7427",
    "purpose": "Emergency slush fund",
    "targetAmount": 3800,
    "targetDate": "2027-02-27"
  },
  {
    "label": "Monzo General Investment Account",
    "purpose": "General investment — no fixed target, tracked for growth",
    "targetAmount": null,
    "targetDate": null
  }
]
```
`label` must exactly match an asset `label` in `netWorth.assets` — that's
how the current balance gets pulled in. Use `null`/`null` for a goal
that's tracked but has no target (shown as balance-only, excluded from
the required-monthly-savings total).

### `story`
```json
{
  "month": "2026-09",
  "narrative": [
    "First paragraph of the write-up.",
    "Second paragraph."
  ]
}
```
Each array entry becomes one `<p>` paragraph in Widget 02.

### `actions`
Including this key **replaces the whole list**, same as every other key
— it does not append. If you want to add new rolling actions without
losing old ones, copy the current list first (Widget 07 → **Copy Current
Data as JSON**) and append the new item(s) to it before pasting back.
```json
[
  { "text": "Check whether the emergency fund deadline should move.", "raisedMonth": "2026-09", "done": false },
  { "text": "Resolved item, kept for the record.", "raisedMonth": "2026-08", "done": true }
]
```

### `netWorthHistory`
**Merged, not replaced**, the same way as `spend` — a monthly update only
needs the new month. This is what feeds **Widget 06 · Net Worth
Trajectory** — a running month-by-month record, separate from the
point-in-time `netWorth` snapshot above:
```json
{
  "netWorthHistory": {
    "months": ["2026-09"],
    "entries": {
      "2026-09": { "assets": 12400, "liabilities": 8040.50, "pensions": 316000 }
    }
  }
}
```
`liabilities` here is the sum of `netWorth.liabilities` (loans, credit
cards) for that month — the app computes net worth itself as
`assets − liabilities + pensions`. Add one entry each month (typically
the same month you refresh `netWorth`) and the trajectory chart/table
extends automatically; existing months are left untouched.

### `meta`
```json
{ "currentAsOfDay": 24 }
```
The day-of-month the *current* (last) month in `spend.months` is partial
through — drives the "to the 24th" wording in Widget 03's header. Set
this to whatever day you pulled the bank statement data up to; if the
month is fully closed, use the month's last day number.

## A safe monthly workflow

1. Hand your new month's bank statement to whatever process builds the
   update (e.g. a Claude conversation), along with this document, and
   ask for a JSON object containing just the new month's `spend` data,
   plus a refreshed `netWorth`, `netWorthHistory`, `story`, `actions`,
   and `meta` (those five are the only keys that typically change month
   to month). You do **not** need to extract or resend the existing
   history — `spend` and `netWorthHistory` merge in automatically.
2. Paste the result into Widget 07, click **Apply Locally**, and eyeball
   the numbers before trusting them — the app recalculates everything
   live, so a wrong figure will usually stand out.
3. Click **Save to Cloud** once it looks right.
4. If you ever want to sanity-check exactly what's currently stored
   (e.g. to hand to a different tool, or just to look something up),
   **Copy Current Data as JSON** in Widget 07 gives you the full current
   state, history included.
