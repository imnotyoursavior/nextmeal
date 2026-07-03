# 🍽️ NextMeal

A single-file web app that looks at your grocery spending habits (bank statement CSV) and grocery
receipts, then anticipates and suggests your next meal. All analysis runs **locally in the
browser** — your financial data never leaves the page.

## How to run

No build step, no dependencies. Either:

- Open `index.html` directly in a browser, or
- Serve it: `python -m http.server 8095 --directory .` then visit http://localhost:8095

## How to use

1. **Bank statement** — upload or paste a CSV export, a **PDF statement**, or a **photo/
   screenshot** of your transactions. CSVs get column auto-detection (Date / Description /
   Amount, headerless sniffing, `$`/`(parens)`/negative amounts). PDFs are read on-device
   with pdf.js (scanned PDFs fall back to OCR); images go through OCR. Non-CSV text uses a
   loose line parser that finds date + description + amount, handles trailing
   running-balance columns, and infers missing years. Extracted text lands in the textarea
   for review before analysis. (`test-statement.pdf` / `test-statement.png` are demo files.)
2. **Grocery receipts** — upload or paste receipt text, **or upload photos of receipts**
   (JPG/PNG/WEBP/BMP). Photos are read with on-device OCR (Tesseract.js — vendored in
   `vendor/`, so it works offline; falls back to CDN if the folder is missing). Extracted
   text is placed in the textarea for review before analysis. Lines like
   `CHKN BREAST 1.8LB   8.12` are parsed into pantry ingredients; totals/tax/tender lines
   are skipped.
3. **Fridge / pantry photo (optional)** — snap your fridge or pantry shelves. Two on-device
   passes find food: COCO-SSD object detection (TensorFlow.js, vendored in `vendor/` +
   `vendor/coco-ssd/`) spots produce like bananas, broccoli, and carrots, while the OCR
   engine reads packaging labels ("WHOLE MILK", "TORTILLAS"). Detected items appear as
   checkboxes — uncheck misfires, add missed items manually — then confirm to merge them
   into your working pantry (shown with a 📷 marker, shared across the household, persisted
   with the photo date, re-scans merge rather than replace, one-click clear).
   Try it with `test-banana.jpg` (object detection) or `test-fridge-labels.png` (label reading).

   **The photo scanner learns.** Every confirmation is a training signal: checked items
   become positive visual examples (a MobileNet embedding of that image region, stored in
   IndexedDB), unchecked items become corrections, and **teach mode** lets you drag a box
   around any item in the photo and name it — including things no detector knows. Future
   scans compare every candidate region (including generic bottles/jars/boxes) against your
   personal example library by cosine similarity, surfacing "🧠 remembered" items. It learns
   *your* specific products — your milk carton, your salsa jar — and gets reliable at ~3
   examples per item. A "What I've learned" panel shows examples per item with thumbnails,
   per-item forget, and full reset. All learning stays on this device.
4. **Household (optional)** — solo users can skip this entirely. Add family members to
   unlock per-person profiles: an "who were these groceries for?" panel appears after
   analysis where each receipt item can be assigned to a person (or left Shared), and a
   "who's eating this meal?" selector scopes every suggestion to the people at the table.
5. Click **Suggest my next meal** — or click **Try with sample data** first to see a demo.
   (`sample-receipt.png` is a test receipt photo you can upload to try the OCR.)
6. After you eat, click **"I made this / I ordered this / I ate out — save to history"**
   on any suggestion card. The log records who ate.

## What it does

- Windows analysis to the **last 30 days** (anchored at the newest transaction).
- Categorizes food spending into **groceries / restaurants / delivery** via merchant
  keyword dictionaries; non-food transactions (gas, Netflix, Amazon…) are ignored.
- Builds a **taste profile**: cuisine affinity scored by visit frequency, spend channel
  (restaurant > delivery > grocery hints), and recency (last 10 days weighted up).
- Builds a **working pantry** from receipt items (40+ ingredient patterns).
- Ranks a built-in database of 60+ meals by cuisine affinity + ingredient overlap and
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

## Kitchen twin (predictive pantry)

NextMeal keeps a dated purchase log (`nextmeal.purchases.v1`) built from receipt dates and
infers what's probably in your kitchen *right now* — no photo needed:

- **Freshness**: per-ingredient shelf-life model (milk ~7d, spinach ~5d, rice ~forever)
  aged from the last purchase date. Items nearing the end show as **⏳ use soon**; expired
  ones as **❌ probably out**.
- **Rebuy rhythm**: once an item has been bought 2+ times, the median interval predicts
  when you'll run out even before it spoils.
- **Photo override**: a fridge-scan sighting is authoritative — it resets the clock.
- **Integration**: meals that rescue expiring items get boosted (with a "🥬 Uses up" note),
  "probably out" items stop counting toward pantry matches, and the out-list becomes a
  one-click copyable shopping list.

## AI Chef (on-device LLM, opt-in beta)

An actual language model running entirely in the browser via WebGPU (WebLLM/MLC). It sees
the pantry, kitchen twin, household, tastes, and meal history — and generates recipes
conversationally. Your data still never leaves the device; only the model weights are
downloaded (once, then cached).

- Opt-in with a size choice: Llama 3.2 1B (~900 MB, recommended) or SmolLM2 360M (~400 MB).
- Quantization is auto-selected per GPU (`q4f16_1` when the `shader-f16` extension exists,
  `q4f32_1` otherwise) so older/integrated GPUs work.
- Preset prompts ("Dinner from my kitchen", "Use up expiring items", "Surprise me") plus
  free-form questions; responses stream in. Requires WebGPU (recent Chrome/Edge/Safari);
  without it the card explains and the rest of the app is unaffected.

## Backup, durability & offline (PWA)

Everything NextMeal knows lives in browser storage, which browsers can evict (Safari purges
site data after ~7 days of disuse). Three defenses:

- **Export / Import** — the 💾 Backup card bundles all `nextmeal.*` localStorage keys plus
  the learned visual examples (IndexedDB) into a single JSON file, and restores from one.
  This is also how you move your data between devices. The file never leaves your hands.
- **Persistent storage** — the app requests `navigator.storage.persist()` and reports
  whether the browser granted it.
- **Service worker PWA** — `sw.js` precaches the app shell and runtime-caches everything
  else (including the ~60 MB of vendored ML models) on first use, so the app loads and works
  fully offline afterward. `manifest.webmanifest` + icons make it installable ("Add to Home
  Screen" on mobile — handy for the fridge-camera workflow).

Staleness nudges appear when the newest statement transaction is over a week old or a fridge
photo scan is aging out.

## Files

- `index.html` — the entire app (markup, styles, parsers, preference model, 60+ meal database, history engine, photo detection, backup).
- `sw.js` / `manifest.webmanifest` / `icon-*.png` — service worker, PWA manifest, and icons for offline use and home-screen install.
- `vendor/` — Tesseract.js OCR engine (~40 MB), TensorFlow.js + COCO-SSD model (~19 MB in `vendor/coco-ssd/`), and MobileNet v2 feature extractor (~2.7 MB in `vendor/mobilenet/`) for fully offline receipt reading, fridge-photo food detection, and personal food-appearance learning.
- `sample-receipt.png` — generated test receipt image for trying the OCR.
- `test-banana.jpg` / `test-fridge-labels.png` — test photos for the fridge/pantry feature (banana photo: Evan-Amos, public domain).
