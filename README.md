# 📈 Orderflow Lab

A self-contained, offline web app for **studying, backtesting, journaling, and ranking** orderflow / Smart-Money / ICT / Volume-Profile trading strategies. Built as a personal learning resource (default instrument: **HYPE — Hyperliquid**, but works for any symbol on TradingView).

> ⚠️ **Not financial advice.** This is an educational tool. The illustrations are *schematic* (idealized), and none of these concepts "predict" the market — they locate asymmetric (good risk:reward) setups. Edge comes from repetition + strict risk management, which is exactly what the Journal and Leaderboard are for.

---

## What it does

| Tab | Purpose |
|-----|---------|
| 📘 **Learn** | 8 core concepts (liquidity, sweep/MSS, FVG, order block, volume profile, CVD/footprint, key levels, market structure) — each with mirrored **long + short** illustrations. Plus a **Volume Profile Shapes** section (D / P / b / B / I). |
| 🎯 **Strategies** | 16 rule-based setups in 5 categories. Each shows a **long + short illustration** and a **Trigger → Entry → Exit → Invalidation** block for both directions, a confluence checklist, and common mistakes. Category filter at the top. |
| 🧠 **Quiz** | 18 multiple-choice questions with instant feedback, explanations, scoring, and shuffle. |
| 🧭 **Plan** | Build & save reusable trade plans (symbol, TF, strategy, entry/stop/target, risk, invalidation). |
| 📓 **Journal** | Record backtested or live trades. Auto-calculates **R** from entry/stop/exit. Shows win rate, expectancy, profit factor, max drawdown, and an equity curve. |
| 🏆 **Leaderboard** | Ranks strategies by an **Edge Score** (expectancy × sample-size confidence) so you scale winners and cut losers. |
| ⚙️ **Data** | Export/Import JSON backups, export CSV, wipe data. |

---

## Running it

There is **no build step and no server required**. It's a single HTML file with everything inline (no external dependencies, works fully offline).

```bash
# just open it
open index.html            # macOS
xdg-open index.html        # Linux
start index.html           # Windows
```

Or double-click `index.html` in a file browser.

> 💡 Optional: serve it locally (`python3 -m http.server` then visit `http://localhost:8000`) if you want a stable URL or to avoid any `file://` quirks. Not required.

---

## Data & privacy

- **Local-first:** data is stored in your browser's `localStorage` (instant, offline). Keys: `ofl_plans_v2`, `ofl_trades_v2`.
- **Optional cloud sync (multi-device):** turn on **Cloud Sync** in the Data tab to also save to **Supabase**, so your journal follows you across devices. Setup is a one-time ~5-min job — see **`SYNC_SETUP.md`**. Until configured, the app is local-only (default).
- With sync on, data is private to your account (protected by Supabase Row Level Security). Nothing is shared publicly.
- **Back up anyway:** Data tab → *Export JSON*. Restore with *Import JSON*.

> ⚠️ Because data lives in `localStorage`, the storage keys are versioned (`_v2`). If a future change alters the trade/plan data shape, bump the version (see `ARCHITECTURE.md`) — but know that bumping the key hides old data until migrated.

---

## Suggested workflow loop

1. **Learn** a concept (and identify the volume-profile *shape* — it decides which strategy is valid).
2. Pick a **Strategy**.
3. Write a **Plan** for it.
4. Open TradingView **Bar Replay** on HYPE/BTC and run **30–50 sample trades**, logging each in the **Journal** (Mode = *Backtest*).
5. Read the **Leaderboard** — which setup has positive expectancy with enough samples?
6. Trade the winner **small and live** (Mode = *Live*).
7. Compare **Live vs Backtest** expectancy. Repeat monthly. Back up weekly.

---

## TradingView setup (for studying the real thing)

- Chart: `BYBIT:HYPEUSDT.P` or `BINANCE:BTCUSDT` (learn on BTC first — cleaner structure).
- Indicators: **Volume Profile (Visible Range + Fixed Range)**, **Session Volume Profile**.
- For real orderflow (Premium): **Cumulative Volume Delta** + the **Footprint** chart.
- Draw horizontal lines: **PDH/PDL, PWH/PWL, daily & weekly open**, and **Asia/London/NY** session ranges.

---

## Project files

```
orderflow-lab/
├── index.html        # the entire app (HTML + CSS + JS inline)
├── README.md         # this file
├── ARCHITECTURE.md   # how the code is organized + how to extend it
└── SYNC_SETUP.md     # one-time Supabase setup for multi-device cloud sync
```

To **add a strategy, quiz question, illustration, or profile shape**, see `ARCHITECTURE.md` — it has copy-paste templates for each.

---

## Roadmap ideas

See the **Backlog** section in `ARCHITECTURE.md`. Highlights:
- Position-size calculator (account + risk% + entry/stop → quantity).
- Tag each journal trade with the **volume-profile shape** → Leaderboard by shape.
- Printable one-page cheat-sheet of all 16 setups.
- Per-strategy detail pages.
- Optional cloud sync (so data follows you across devices).

---

## Credits / accuracy

Strategy rules were grounded in published ICT/SMC/orderflow/volume-profile sources and deliberately corrected for common myths (e.g. "all FVGs must fill" — false; "wick = sweep, body close = break"; "VA mean-reversion only works in D-shape profiles"; "reset CVD at session, never use it standalone"). See concept caveats in the Learn tab.
