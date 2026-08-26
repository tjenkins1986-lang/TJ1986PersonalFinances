# Monthly Snapshot — JSON Format Reference

This is the exact shape the app expects when you paste a snapshot into
**Widget 07 · Monthly Data Update** in `personal-finance-tracker.html`.
Give this file to whatever process generates your monthly JSON (e.g. a
Claude conversation fed your bank statements) so its output loads cleanly.

## How loading works

- Paste JSON into the Widget 07 box, click **Apply Locally** to preview
  in your browser only, then **Save to Cloud** to make it live.
- **Partial updates are fine, and work two different ways depending on
  the key:**
  - `spend`, `netWorthHistory`, and `income` are **merged**, not
    replaced. Include only the new month's data and it's added alongside
    everything already there — existing months/categories you don't
    mention are left untouched. You never need to resend the full
    history.
  - Every other key (`netWorth`, `savingsGoals`, `story`, `actions`) is a
    **full replace** if included at all — it represents current state,
    not a history, so send the complete current value of that key, not a
    delta. A typical monthly update only needs `netWorth`, `spend`,
    `netWorthHistory`, `income`, `story`, and `actions`; `savingsGoals`
    changes rarely enough to just omit.
  - Any key you omit entirely is left exactly as it was.
- Every number is a plain JSON number — no `£` signs, no commas
  (`1234.56`, not `"£1,234.56"`).
- Every month is `"YYYY-MM"` (e.g. `"2026-09"`). Every date is
  `"YYYY-MM-DD"` (e.g. `"2026-09-03"`).
- The **latest** month present across `spend.monthly` after merging is
  treated as "the current month" in Widget 03 (drives the running-average
  window), so just include the new month you want to add.
- Unrecognised top-level keys are ignored (and called out in the status
  message after **Apply Locally**), not an error — a small guard against
  typos silently doing nothing.

## Top-level keys

```json
{
  "netWorth": { ... },
  "spend": { ... },
  "income": { ... },
  "savingsGoals": [ ... ],
  "story": { ... },
  "actions": [ ... ],
  "netWorthHistory": { ... }
}
```

### `netWorth`
```json
{
  "asOf": "2026-09-24",
  "assets": [
    { "label": "Current Account", "amount": 185.18 },
    { "label": "Rainy Day Pot", "amount": 1531.00 }
  ],
  "liabilities": [
    { "label": "Credit Card", "amount": 1344.50 },
    { "label": "Personal Loan", "amount": 10363.00 }
  ],
  "pensions": {
    "pots": [ { "label": "Workplace Pension", "amount": 315820.00 } ]
  }
}
```
Full replace. `asOf` drives the "AS OF" label in the header — set it to
the date you pulled the figures. Asset `label`s matter beyond display:
**Widget 06's savings goals link to an asset by matching its `label`
exactly** (see `savingsGoals` below), so keep account labels stable month
to month rather than renaming them. `pensions.pots` and `liabilities` are
both flat lists — one entry per pot/debt, no split between people. Net
worth is computed as `assets + pensions − liabilities` (no property/
mortgage tile in this version).

### `spend`
**Merged, not replaced** — a monthly update only needs the new month:
```json
{
  "spend": {
    "monthly": {
      "Groceries": { "2026-09": 420.46 }
    },
    "transactions": {
      "Groceries": {
        "2026-09": [
          { "date": "2026-09-06", "desc": "TESCO STORES", "amount": 12.89, "source": "current_account" }
        ]
      }
    }
  }
}
```
Only mention the categories that actually had spend that month — any
category you don't include is left exactly as it was. To correct a past
month, include that month's key again under the relevant category — it
overwrites just that category/month combination, nothing else.

For each category, `monthly[category][month]` should equal the sum of
`transactions[category][month][].amount` for that same month — the
`monthly` figure is what every calculation actually uses; `transactions`
only populates the double-click drill-down panel. `source` on each
transaction is a free-text tag for which account/card it came from
(shown in the drill-down, e.g. `current_account`, `barclaycard`) — purely
informational, nothing keys off its exact value.

`spend.months`/`spend.categories` are **not** read on input — the app
derives the month list and category set from `monthly`'s keys directly,
so there's nothing to keep in sync. **Copy Current Data as JSON** still
includes a `spend.months` array for human readability, but you never need
to send one back.

**Category names are grouped in `SPEND_GROUPS`, hardcoded in
`personal-finance-tracker.html`:**
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
silently dropped — but it won't have a "proper" home until you add it to
`SPEND_GROUPS` in the code (a code change, not a data change). Ask for
that separately rather than inventing a new category name in the JSON if
you want it to land in a specific group from the start.

### `income`
**Merged, not replaced**, the same way as `spend` — a monthly update only
needs the new month. Feeds **Widget 05 · Income & Outgoings** alongside
total spend for that month:
```json
{
  "income": {
    "monthly": { "2026-09": 4574.04 }
  }
}
```
A month present in spend but missing from `income.monthly` (e.g. this
month's salary hasn't landed yet) is shown as **pending** in the cash
flow chart rather than as zero income — don't backfill a guess, just
leave the month out until it lands.

### `savingsGoals`
Full replace.
```json
[
  {
    "label": "Rainy Day Pot",
    "purpose": "Emergency fund",
    "targetAmount": 4000,
    "targetDate": "2027-02-28"
  },
  {
    "label": "Stocks & Shares ISA",
    "purpose": "Long-term savings — no fixed target",
    "targetAmount": null,
    "targetDate": null
  }
]
```
`label` must exactly match an asset `label` in `netWorth.assets` — that's
how the current balance gets pulled in. Use `null`/`null` for a goal
that's tracked but has no target (shown as balance-only).

### `story`
Full replace. Each entry is a bullet with a sentiment dot, not a
paragraph:
```json
{
  "month": "2026-09",
  "narrative": [
    { "type": "negative", "text": "Fishing/Angling hit £872 in August — the highest single month this year." },
    { "type": "positive", "text": "No Travel/Trips spend in August — the first quiet month since March." },
    { "type": "neutral",  "text": "Retail/Shopping was again the largest single category." }
  ]
}
```
`type` is one of `"positive"`, `"negative"`, `"neutral"` — anything else
still renders but without a recognised dot colour. Written by hand (or by
Claude, during the monthly update chat) — nothing here is computed
in-browser.

### `actions`
Full replace — same as every other full-replace key, it does not append.
If you want to add new rolling actions without losing old ones, copy the
current list first (Widget 07 → **Copy Current Data as JSON**) and append
the new item(s) before pasting back.
```json
[
  { "text": "American Express is carrying interest — prioritise paying this down.", "raisedMonth": "2026-08", "done": false },
  { "text": "Resolved item, kept for the record.", "raisedMonth": "2026-07", "done": true }
]
```

### `netWorthHistory`
**Merged, not replaced**, the same way as `spend` — a monthly update only
needs the new month. Feeds **Widget 04 · Net Worth Trajectory**, a
running month-by-month record separate from the point-in-time `netWorth`
snapshot above:
```json
{
  "netWorthHistory": {
    "entries": {
      "2026-09": { "assets": 14978.26, "liabilities": 16526.50, "pensions": 321735.00 }
    }
  }
}
```
`liabilities` here is the sum of `netWorth.liabilities` for that month —
the app computes net worth as `assets − liabilities + pensions`. Add one
entry each month (typically the same month you refresh `netWorth`) and
the trajectory chart/table extends automatically.

## A safe monthly workflow

1. Hand your new month's statements to whatever process builds the
   update (e.g. a Claude conversation), along with this document, and
   ask for a JSON object containing just the new month's `spend` and
   `income`, plus a refreshed `netWorth`, `netWorthHistory`, `story`, and
   `actions`. You do **not** need to extract or resend the existing
   history — `spend`, `income`, and `netWorthHistory` merge in
   automatically.
2. Paste the result into Widget 07, click **Apply Locally**, and eyeball
   the numbers before trusting them — the app recalculates everything
   live, so a wrong figure will usually stand out. Check the status box
   for any unrecognised keys it flagged.
3. Click **Save to Cloud** once it looks right.
4. If you ever want to sanity-check exactly what's currently stored,
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
Widget 07. Wiring the in-page buttons to a real update is a natural next
step, not yet built.
