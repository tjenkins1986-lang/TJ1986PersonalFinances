# TJ1986 Personal Finances

A private financial dashboard for tracking net worth, spending, income
vs. outgoings, and savings goals. Single static page
(`personal-finance-tracker.html`), gated behind Google sign-in, backed by
a private Firestore database. `index.html` is just a redirect stub to
that file — see **Why two HTML files?** below.

This started as a fork of [21ForthCresFinance](https://github.com/tjenkins1986-lang/21ForthCresFinance),
a joint household version of the same app, then went through a
single-person adaptation of that household layout. Both of those are
retired now — this is a from-scratch rebuild around how one person
actually uses it, not a household app with the second person removed.

## Widgets

1. **Net Position** — assets, liabilities, pensions, as accordion
   sections with a headline net worth figure and month-on-month deltas.
2. **Story of the Month** — sentiment-tagged narrative bullets, plus a
   standing open/completed actions list.
3. **What I Spend** — a category spend matrix (one row per category,
   one column per month) grouped into Set Spend / Discretionary /
   One-Off / To Be Reviewed, with a running average and double-click
   drill-down to individual transactions.
4. **Net Worth Trajectory** — a diverging bar chart of assets vs.
   liabilities, month on month.
5. **Income & Outgoings** — income vs. total spend per month, with
   pending (not-yet-landed) months shown explicitly rather than as zero.
6. **Savings** — goal cards, balance pulled from Net Position by
   matching account label, required monthly contribution for dated goals.
7. **Monthly Data Update** — the JSON paste/cloud-sync panel: Apply
   Locally (preview), Save to Cloud (persist), Copy Current Data as JSON.

See `DATA-FORMAT.md` for the exact JSON schema each widget reads.

## Why two HTML files?

GitHub Pages always serves `index.html` for the site root — that's a
fixed convention of static hosting, not something this repo's deploy
workflow controls. So `index.html` stays in place as a two-line redirect
stub, and the actual app lives in `personal-finance-tracker.html`. If
you're editing the app, that's the file to open — `index.html` should
essentially never need to change.

## Why there's no data in this repo

This repository is **public**, and the page is served as plain static
files (GitHub Pages) — anyone who can load the URL can also view-source it
or `git clone` this repo. So no real balances or transactions are ever
committed here. `personal-finance-tracker.html` ships as an empty shell;
every widget loads its data live from Firestore, only after a Google
sign-in that Firestore's own security rules approve.

## One-time setup

### 1. Create a Firebase project
Go to <https://console.firebase.google.com>, click **Add project**, and
follow the prompts (Google Analytics is optional, skip it if you like).
**Use a new project — don't reuse the household one**, so this data stays
fully separate.

### 2. Register a Web App
In the project's dashboard, click the `</>` (Web) icon to add a web app.
Give it any nickname. Firebase will show you a `firebaseConfig` object
that looks like:

```js
const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project",
  storageBucket: "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123"
};
```

Copy this whole object. In `personal-finance-tracker.html`, find the
`firebaseConfig` object near the bottom of the file (inside the last
`<script type="module">` block) and replace the placeholder values with
your real ones. These values are not secret — they're meant to be public
in client-side code — so it's fine to commit this change.

### 3. Enable Google sign-in
In the Firebase Console: **Authentication** > **Sign-in method** > enable
**Google**.

### 4. Add your GitHub Pages domain as authorized
Still under **Authentication** > **Settings** > **Authorized domains**,
add `<your-github-username>.github.io` (Firebase adds `localhost` by
default, which is handy for testing locally too).

### 5. Create the Firestore database
**Firestore Database** > **Create database** > pick a region close to you
> start in **production mode** (the rules below take over immediately).

### 6. Lock it down with security rules
Open `firestore.rules` in this repo, replace the placeholder email with
your real Google account email, then paste the whole file into
**Firestore Database** > **Rules** in the console and click **Publish**.
This is what actually restricts who can read or write your data — the
sign-in screen in the app is just UI.

Also fill in the same email in the `ALLOWED_EMAILS` array near the bottom
of `personal-finance-tracker.html`, so the app can show a friendly "not
authorized" message instead of a confusing permission error if the wrong
account signs in. This part is cosmetic; the Firestore rules are what
matters.

### 7. Enable GitHub Pages
This repo already has a GitHub Actions workflow (`.github/workflows/deploy-pages.yml`)
that deploys on every push to `main`. In this repo on GitHub: **Settings**
> **Pages** > under **Build and deployment**, set **Source** to
**"GitHub Actions"**. Your site will be live shortly after at:

```
https://<your-github-username>.github.io/TJ1986PersonalFinances/
```

(That URL loads `index.html`, which immediately redirects to
`personal-finance-tracker.html` — same effective page.)

### 8. Seed your real data
Once the site is live and Firebase is wired up:

1. Open the site and sign in with Google.
2. You'll see an empty dashboard with a note that there's no cloud data
   yet.
3. Scroll to **Widget 07 · Monthly Data Update**, paste your real data
   (kept locally, never committed here) into the box, and click
   **Apply Locally** to preview it.
4. If it looks right, click **Save to Cloud**.
5. Delete or securely store the local seed file — it's no longer needed
   day-to-day; the JSON textarea + "Copy Current Data as JSON" button in
   Widget 07 is how you'll read/write full snapshots going forward, and
   the data itself now lives in Firestore.

## Updating data each month

Widget 07 (**Monthly Data Update**) accepts a JSON object with any of
these top-level keys — only the keys you include get replaced (or, for
`spend`/`netWorthHistory`/`income`, merged in), everything else is left
as-is. See `DATA-FORMAT.md` for the full schema.

## Local testing

You can open `personal-finance-tracker.html` directly in a browser
(`file://`) to check layout and styling, but sign-in won't work over
`file://` — Firebase Auth needs `http://` or `https://`. Firebase's
authorized domains list includes `localhost` by default, so running any
static file server locally (e.g. `python3 -m http.server`) and visiting
`http://localhost:8000/personal-finance-tracker.html` will let you test
the full sign-in flow before it's live on GitHub Pages.

## Known limitation

Widget 03's per-transaction "Move" / "Delete" controls are a preview-only
UI stub — they don't write anywhere yet (see `DATA-FORMAT.md`'s note on
this). Recategorising a transaction today means editing your source data
and repasting that month's `spend` via Widget 07.

## Architecture notes

- **Hosting:** GitHub Pages, deployed via GitHub Actions, static files
  only, no build step. `index.html` is a redirect stub; the real app is
  `personal-finance-tracker.html`.
- **Auth:** Firebase Authentication, Google provider only.
- **Storage:** One Firestore document (`household/snapshot` — name kept
  from the household version, harmless to rename if you want to) holding
  the entire app state as JSON.
- **Security:** Enforced by `firestore.rules`, not by the app's UI. The
  sign-in gate only controls what's *displayed* — the rules control what
  Firestore actually *allows*.
- **No secrets in git:** The Firebase config values committed in
  `personal-finance-tracker.html` are safe to be public (that's how
  Firebase web apps are designed to work). Your real financial figures
  are not — they only ever exist in Firestore and in whatever local file
  you use to seed it.
