# 🏗️ Orderflow Lab — Architecture & Extension Guide

Everything lives in **one file**: `index.html` (~1300 lines). It's plain HTML + CSS + vanilla JavaScript with **no dependencies and no build step**. This doc explains how it's organized and gives **copy-paste templates** for adding content.

---

## 1. High-level layout of `index.html`

```
<head>
  <style> … design tokens (CSS variables) + all component styles … </style>
</head>
<body>
  <header><nav id="nav"> … tab buttons … </nav></header>
  <main>
    <section id="learn">       …  (active by default)
    <section id="strategies">  …
    <section id="quiz">        …
    <section id="plan">        …
    <section id="journal">     …
    <section id="leaderboard"> …
    <section id="data">        …
  </main>
  <script> … all logic … </script>
</body>
```

- **Tabs** are `<section>`s; only one has class `active` at a time. The nav handler (near the bottom of the script, `document.querySelectorAll('#nav button')…`) toggles them.
- Each section's content is **rendered by JS** from data arrays — you rarely edit the HTML body; you edit the **data arrays** and the renderers redraw.

### Where things are (approx. line numbers — may drift as you edit)

| Thing | Location |
|---|---|
| CSS design tokens (`:root`) | top of `<style>` |
| Storage keys + load/save helpers | `const LS_PLANS='ofl_plans_v2' …` |
| **Scene engine** (illustrations) | `WD/HD`, `flipScene`, `serialize`, `scene()`, `C()` |
| Scene builders | `function scn_*()` … |
| `SCENES` map | `const SCENES = { … }` |
| `LEARN` data | `const LEARN = [ … ]` |
| `CATS` + `STRATEGIES` data | `const CATS = […]`, `const STRATEGIES = […]` |
| `QUIZ` data | `const QUIZ = [ … ]` |
| Volume-profile shapes: `vpHist()` + `SHAPES` | near the end, before INIT |
| Plan / Journal / Leaderboard / Data logic | middle-to-end of script |
| **INIT** (calls all renderers once) | very bottom: `renderLearn(); renderShapes(); …` |

> Tip: search the file for the headings like `/* ===================== STRATEGY DATA` — the script is divided by banner comments.

---

## 2. The illustration "Scene Engine" (the clever bit)

All candlestick diagrams are **inline SVG generated from data**, authored **once for the LONG case** and **auto-mirrored** to produce the SHORT case. This is why every strategy gets a long *and* short diagram for free.

### Coordinate convention
- SVG canvas is `WD × HD` (default `380 × 210`).
- **Smaller `y` = higher price** (standard SVG, y grows downward).

### Primitives (array of objects)
A scene is an **array of primitive objects**. Types:

| `t` | Meaning | Key fields |
|---|---|---|
| `'c'` | candle | `x, o, c, h, l, col, w` (o/c/h/l are **y-values**; `col` = `G`/`R`) |
| `'z'` | zone (rect) | `x, y, w, h, col, op, label, lx, ly` |
| `'l'` | line | `x1, y1, x2, y2, col, dash, label, lx, ly, sw` |
| `'a'` | arrow | `x1, y1, x2, y2, col` (auto arrowhead) |
| `'tx'` | text | `x, y, s, col, size, anchor, weight` |

Colors are the constants `G` (green), `R` (red), `P` (purple), `Y` (yellow), `A` (accent blue), `M` (muted grey).

### Direction-aware labels
Any `label` or text `s` can be a **string** *or* an object `{long:'…', short:'…'}`. When rendering the short version, the engine picks the `short` text. Example:
```js
{t:'l', x1:38, y1:150, x2:300, y2:150, col:R, dash:'4 3',
 label:{long:'SSL (sell-side liquidity)', short:'BSL (buy-side liquidity)'}, lx:150, ly:165}
```

### How mirroring works
`flipScene(items)` vertically flips every `y` (`y → HD - y`) **and swaps green↔red** (a bullish setup mirrored becomes bearish). `scene(items, dir)` calls `flipScene` only when `dir === 'short'`, then `serialize()` turns primitives into SVG strings.

```js
scene(SCENES.fvg(), 'long')   // bullish FVG diagram
scene(SCENES.fvg(), 'short')  // same diagram, mirrored → bearish
```

### Candle helper
```js
const C = (x,o,c,h,l,col,w) => ({t:'c',x,o,c,h,l,col,w});
//        x   open close high low color width(optional)
```
Remember: because lower y = higher price, a **green/up candle has `c < o`** (close is higher on screen = smaller y).

---

## 3. Copy-paste templates

### ➕ Add a new STRATEGY

1. (Optional) pick or add a category in `CATS`. Existing: `ict`, `struct`, `vp`, `of`, `levels`.
2. Add an object to the `STRATEGIES` array:

```js
{
  id:'my_setup',                 // unique, lowercase_snake — also used as the journal key
  cat:'ict',                     // must match a CATS id
  name:'My New Setup',
  type:'Reversal',               // free label shown as a pill
  cls:'grn',                     // pill colour: grn|red|yel|pur|acc
  scene:'sweepMSS',              // a key in the SCENES map (reuse an existing diagram or add one)
  range:false,                   // true = range/mean-reversion (shows ONE diagram, no long/short mirror)
  blurb:'One-line description of the idea.',
  long:{
    trig:'What must be true to look (the context/trigger).',
    entry:'The exact entry condition.',
    exit:'Target / how you take profit.',
    inval:'When the idea is wrong — stand aside / exit.'
  },
  short:{ trig:'…', entry:'…', exit:'…', inval:'…' },   // omit if range:true
  conf:['Confluence A','Confluence B','Confluence C'],   // also auto-added to Journal checkboxes
  mistakes:'The errors that kill this setup.'
}
```

That's it — `renderStrategies()` will draw the card, and the strategy automatically appears in the Plan dropdown, Journal dropdown, Journal confluence checkboxes, and Leaderboard.

> **Note on confluences:** the Journal's confluence checkboxes are the **union of all `conf` arrays** across strategies, so new ones show up automatically.

> **Note on journal continuity:** trades are keyed by strategy `id`. If you **rename an `id`**, old trades referencing the old id will display the raw id as their name (still counted, just unlabelled). Prefer adding new ids over renaming.

---

### ➕ Add a new ILLUSTRATION (scene)

1. Write a builder that returns an array of primitives, **authored for the LONG case**:

```js
function scn_mything(){ return [
  C(50,150,120,156,116, R),                 // a down candle
  C(80,120, 90,124, 86, G),                 // an up candle
  {t:'l', x1:40,y1:96,x2:300,y2:96, col:A, dash:'4 3',
   label:'BOS', lx:250, ly:90},
  {t:'z', x:120,y:96,w:90,h:24, col:G, op:0.16,
   label:{long:'demand', short:'supply'}, lx:150, ly:130},
  {t:'a', x1:175,y1:150,x2:175,y2:118, col:A},   // up arrow (reversal/continuation)
  {t:'tx', x:150,y:200, s:'caption', col:M}
];}
```

2. Register it in the `SCENES` map:
```js
const SCENES = { …, mything: scn_mything };
```

3. Reference it from a strategy (`scene:'mything'`) or a Learn item.

**Authoring tips**
- Keep candles within `y` ≈ `40–185` so wicks/labels don't clip.
- For a clean mirror, author the long version so the *reversal/continuation goes UP* (smaller y). The flip turns it into a clean short.
- Don't hard-code green/red backwards — the flip swaps them; author the *bullish* meaning.
- Range scenes (`range:true` strategies, or symmetric concepts) are **not** mirrored — author them to read correctly as-is (e.g. `scn_vpRange`).

---

### ➕ Add a LEARN concept

Append to the `LEARN` array:
```js
{
  n:'9 · My Concept', pill:'short tagline', cls:'acc',
  scene:'mything',     // a SCENES key
  flip:true,           // true → shows side-by-side long + short; false → single diagram
  body:`HTML string. Use <ul class="tight"><li>…</li></ul> for bullets.`
}
```
`renderLearn()` handles the rest.

---

### ➕ Add a QUIZ question

Append to the `QUIZ` array:
```js
{
  q:'The question text?',
  o:['Option A','Option B','Option C','Option D'],   // any number of options
  a:1,                                               // index of the CORRECT option (0-based)
  e:'Explanation shown after answering.'
}
```
Scoring, shuffle, and the pass/fail verdict update automatically.

---

### ➕ Add / edit a VOLUME-PROFILE SHAPE

Shapes use a **separate renderer** (`vpHist`) — a horizontal histogram, *not* the candle scene engine. Append to the `SHAPES` array:

```js
{
  name:'X-shape — label', cls:'acc',
  widths:[24,46,78,120,162,120,78,46,24],  // bar widths TOP→BOTTOM (top=high price)
  poc:4,                                     // index of the Point of Control bar (highlighted yellow)
  va:[2,6],                                  // [VAH index, VAL index] — the value-area band (blue)
  poc2:null,                                 // optional 2nd POC (double distribution)
  lvn:null,                                  // optional Low-Volume-Node index (highlighted red)
  path:[[40,120],[60,90],[80,118]],          // optional schematic price path (left side), [x,y] points
  id:'How to identify it.',
  means:'What it implies about the market.',
  trade:'How to trade it (HTML ok).',
  uses:'Which strategies apply'
}
```
`renderShapes()` draws it. The histogram auto-colours: POC = yellow, in-value-area = blue, outside = grey, LVN = red.

---

## 4. Data model (localStorage)

### Plan object (`ofl_plans_v2` → array)
```js
{ id, name, sym, tf, strat, dir, entry, stop, tp, rr, risk, max, inval }
```

### Trade object (`ofl_trades_v2` → array)
```js
{
  id, date, mode,        // mode: 'Backtest' | 'Live'
  strat,                 // strategy id
  sym, dir,              // dir: 'Long' | 'Short'
  tf, out,               // out: 'Win' | 'Loss' | 'Breakeven'
  entry, stop, exit,     // strings (raw input)
  r,                     // number — the result in R (drives ALL stats)
  conf:[…],              // array of confluence strings ticked
  img, notes,
  updated                // ISO timestamp — used for cloud merge (last-write-wins)
}
```

### Cloud sync (optional — Supabase)
- Config: the `SUPABASE = { url, anonKey }` block at the top of the script. Empty = local-only (default).
- Library: loaded via `<script src="…@supabase/supabase-js@2">` before the main script (exposes `window.supabase`).
- The `cloud` IIFE wraps all sync: `init, signIn, signUp, signOut, syncNow, upsert, remove, removeMany, pushAll`.
- **localStorage-first:** mutations (`saveTrade`/`delTrade`/`savePlan`/`delPlan`) write localStorage first, then call `cloud.upsert/remove` (no-ops if not configured/signed in).
- **Merge:** `pull()` fetches the user's rows and merges by `id`, newest `updated` wins. `syncNow()` = pull + pushAll.
- **Schema:** Supabase tables `trades`/`plans` each store `{ id, user_id (default auth.uid()), payload jsonb, updated_at }` with Row Level Security scoping rows to the owner. Full SQL in `SYNC_SETUP.md`.
- **Known limit:** deletes have no tombstones, so an offline delete on one device can be re-pushed by another. Delete again while online to clear.

### Stats (computed, not stored) — see `statsFor(list)`
`n, winrate, avgWin, avgLoss, totalR, pf (profit factor), expectancy, maxDD`.

### Edge Score (Leaderboard) — see `renderLeaderboard()`
```
confidence = min(1, N/30)
edge = expectancy × √(min(N,100)) × (0.4 + 0.6 × confidence)
```
Rewards positive expectancy, scales with sample size, penalizes thin samples. Verdicts: `Scale up` (N≥10 & expectancy>0.1), `Keep testing` (N<10), `Cut/fix` (expectancy<0), else `Marginal`.

---

## 5. Conventions & gotchas

- **No framework, no bundler.** Keep it dependency-free and offline-capable — that's a feature. Don't add a CDN `<script>` unless you also inline it.
- **Renderers are idempotent**: each `render*()` rebuilds its container's `innerHTML` from data. To refresh after changing data in code, just call the renderer (or reload the page).
- **Editing data ≠ editing storage.** Changing the `STRATEGIES`/`QUIZ` arrays changes the app's *content*. User trades/plans live in `localStorage` and are untouched by code edits (unless you bump the `LS_*` keys).
- **Bumping storage version**: if you change the trade/plan shape incompatibly, change `'ofl_trades_v2'` → `'…_v3'`. Old data becomes invisible. Better: write a one-time migration that reads the old key, transforms, and writes the new key.
- **Colours**: use the CSS variables (`var(--grn)` etc.) in CSS and the JS constants (`G`, `R`, …) in SVG. Keep them in sync if you re-theme.
- **Mobile**: layouts collapse to one column under 920px via the `@media` rule. Test new wide tables/grids there.

---

## 6. Validating a change

After editing, sanity-check the inline JS parses (catches the most common breakage):

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');
const m=h.match(/<script>([\s\S]*)<\/script>/);
try{new Function(m[1]);console.log('JS OK')}catch(e){console.log('ERR',e.message);process.exit(1)}"
```

Then open `index.html` and click through the tab(s) you touched. Check the **browser devtools console** for runtime errors.

---

## 7. Backlog / improvement ideas

**Quick wins**
- [x] **Multi-device cloud sync** (Supabase, localStorage-first). See `SYNC_SETUP.md`.
- [ ] **Position-size calculator** (account balance + risk% + entry/stop → position size / qty). Add as a small widget in Plan or a new mini-tab.
- [ ] **Tag each trade with the volume-profile shape** (D/P/b/B/I). Add a `shape` field to the trade form + object, then a Leaderboard view "expectancy by shape" — reveals which conditions your edges actually work in.
- [ ] **Delete tombstones** so offline deletes propagate cleanly across devices.
- [ ] **Printable cheat-sheet** view (one page, all 16 setups' trigger/entry/exit/inval) with `@media print` CSS.
- [ ] **Filter Journal by date range / direction / outcome.**

**Medium**
- [ ] **Per-strategy detail pages** (deep-dive content + a gallery of saved screenshot URLs for that setup).
- [ ] **R-multiple distribution histogram** and **win/loss streak** stats in the Journal.
- [ ] **"Pre-trade checklist" modal** that must be ticked before logging a Live trade.
- [ ] **Embed a live TradingView chart widget** in a tab (note: needs an internet connection; keep the rest offline).

**Bigger**
- [ ] **Optional cloud sync** (e.g. a tiny backend or a hosted DB) so data follows you across devices — the current single biggest limitation.
- [ ] **CSV import** of trades (round-trip with the existing CSV export).
- [ ] **Strategy A/B comparison** view (two strategies' equity curves overlaid).
- [ ] **Split `index.html`** into `app.css` / `app.js` / `data.js` if it grows past comfortable single-file size (trade-off: loses the "one file, double-click to run" simplicity).

---

## 8. Changelog (keep this updated as you go)

- **v2.1** — Optional Supabase cloud sync (multi-device), localStorage-first with last-write-wins merge. Added `updated` field to trades/plans. New `SYNC_SETUP.md`.
- **v2** — Rebuilt: 16 categorized strategies with mirrored long/short illustrations + entry/exit/invalidation, 18-question quiz, volume-profile shapes (D/P/b/B/I). Storage keys bumped to `_v2`.
- **v1** — Initial lab: 6 strategies, concept explainers, plan/journal/leaderboard.
