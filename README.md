# 🍽️ NextMeal

A single-file web app that looks at your grocery spending habits (bank statement CSV) and grocery
receipts, then anticipates and suggests your next meal. All analysis runs **locally in the
browser** — your financial data never leaves the page.

## How to run

No build step, no dependencies. Either:

- Open `index.html` directly in a browser, or
- Serve it: `python -m http.server 8095 --directory .` then visit http://localhost:8095

## How to use

1. **Bank statement** — upload or paste a CSV export from your bank. Column headers
   (Date / Description / Amount, or Payee / Merchant / Debit, etc.) are auto-detected;
   headerless CSVs are sniffed. Negative, positive, `$`, and `(parens)` amounts all work.
2. **Grocery receipts** — upload or paste receipt text, **or upload photos of receipts**
   (JPG/PNG/WEBP/BMP). Photos are read with on-device OCR (Tesseract.js — vendored in
   `vendor/`, so it works offline; falls back to CDN if the folder is missing). Extracted
   text is placed in the textarea for review before analysis. Lines like
   `CHKN BREAST 1.8LB   8.12` are parsed into pantry ingredients; totals/tax/tender lines
   are skipped.
3. **Household (optional)** — solo users can skip this entirely. Add family members to
   unlock per-person profiles: an "who were these groceries for?" panel appears after
   analysis where each receipt item can be assigned to a person (or left Shared), and a
   "who's eating this meal?" selector scopes every suggestion to the people at the table.
4. Click **Suggest my next meal** — or click **Try with sample data** first to see a demo.
   (`sample-receipt.png` is a test receipt photo you can upload to try the OCR.)
5. After you eat, click **"I made this / I ordered this / I ate out — save to history"**
   on any suggestion card. The log records who ate.

## What it does

- Windows analysis to the **last 30 days** (anchored at the newest transaction).
- Categorizes food spending into **groceries / restaurants / delivery** via merchant
  keyword dictionaries; non-food transactions (gas, Netflix, Amazon…) are ignored.
- Builds a **taste profile**: cuisine affinity scored by visit frequency, spend channel
  (restaurant > delivery > grocery hints), and recency (last 10 days weighted up).
- Builds a **working pantry** from receipt items (40+ ingredient patterns).
- Ranks a built-in database of ~20 meals by cuisine affinity + ingredient overlap and
  presents four lanes:
  - 🍳 **Cook it** — best full recipe match, with inline steps and "you already have X of Y ingredients"
  - ⏱️ **Keep it simple** — best low-effort match (10–15 min meals)
  - 🛵 **Order in** — favorite delivery cuisine, budgeted at your average delivery ticket, with Uber Eats / DoorDash search links
  - 🍽️ **Eat out** — affordable nearby option for your top cuisine, budgeted at your average restaurant ticket, with Google Maps / Yelp links
  - 🧪 **Try something new** — a meal you've never logged, from a cuisine adjacent to your
    favorite (e.g. Mexican → Thai via "lime, cilantro, and chile heat"), to push gentle
    experimentation
- Plus a spending snapshot (totals, trip counts, average tickets) and 5 runner-up meal ideas.

## Meal history (saved by default)

Every meal you log is stored in the browser's `localStorage` (key `nextmeal.history.v1`) and
feeds back into the suggestion engine:

- **Learned preference** — cuisines you actually log eating get a scoring boost.
- **Repeat cooldown** — a meal eaten in the last 7 days "rests" (strong penalty, shown as
  *"resting — you just had it"* in runner-ups); 7–14 days gets a milder penalty.
- **Experimentation nudge** — the 🧪 lane always favors never-logged meals from cuisines
  adjacent to your favorite.

The history panel shows your recent meals, cuisines explored, and experiments tried, with a
two-click **Clear history** button. Nothing syncs anywhere — it lives in this browser only.

## Household profiles (family vs. single meals)

Add household members (stored in `nextmeal.household.v1`) to differentiate family meals from
single meals:

- **Item assignment** — after analysis, assign each receipt item to a person or leave it
  Shared (`nextmeal.assignments.v1`, remembered by item name so future receipts auto-apply).
- **Who's eating** — a persistent selector (`nextmeal.eaters.v1`) scopes the pantry, cuisine
  hints, and meal history to just the selected eaters. Someone else's groceries won't be
  suggested for your solo dinner.
- **Sizing** — recipe cost estimates scale to the headcount ("~$13.50 for 3 people" vs.
  "~$4.50 for 1 person"), and the suggestion header says who the meal is for.
- **Per-person learning** — meal logs record who ate, so the repeat cooldown and cuisine
  boosts apply per person: tacos you had solo last night are still fresh for the kids.

With a single member the app behaves exactly like the solo version — all household UI stays hidden.

## Files

- `index.html` — the entire app (markup, styles, parsers, preference model, meal database, history engine).
- `vendor/` — Tesseract.js OCR engine (~40 MB: worker, WASM cores, English language data) for fully offline receipt-photo reading.
- `sample-receipt.png` — generated test receipt image for trying the OCR.
