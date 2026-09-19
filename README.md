# Deal Evaluator

Single-file, client-side web tool: paste a price table, enter an offer, get **TAKE / COUNTER / PASS** with reasoning. No backend, no accounts. Everything lives in the browser's `localStorage`.

Standalone — not part of JED-I Dispatch or JED-I Recover.

## Files

- `index.html` — the whole app (HTML + CSS + JS inline). That's it.

## How it works

| Tab | What it does |
|---|---|
| **Evaluate** | Item search (autocompletes from your price table), tracked value, offer per unit, quantity → verdict card with 2–4 reasoning bullets. Every evaluation is appended to the log. |
| **Prices** | Paste a Collectr CSV export *or* manual `Item, 123.45` lines → preview with column mapping → **Replace** or **Merge**. Shows last-updated time. |
| **Thresholds** | The four numbers the verdict depends on, editable in place. Saved automatically. |
| **Log** | Every evaluation (timestamp, item, qty, offer, tracked, Δ%, verdict, suggested counter). **Export CSV** for the outcome dataset. |

### Verdict rules

`discount = (tracked − offer) / tracked`, positive = under tracked. Checked in this order:

1. `discount ≥ floorPct` → **PASS** (sanity floor — too good to be true; verify authenticity/condition)
2. `discount ≥ takePct` → **TAKE**
3. `discount ≥ −counterMaxOverPct` → **COUNTER**, suggested counter = `tracked × (1 − counterTargetPct)`
4. otherwise → **PASS** (offer too far over tracked)

Shipped defaults (placeholders, marked as such in the UI): take 15% · counter band up to 5% over · counter target 15% under · floor 60%.

## Changing the CSV format expectation

Column detection is header-based. Edit `COLUMN_HINTS` near the top of the `<script>` block in `index.html`:

```js
const COLUMN_HINTS = {
  name:  ['product name', 'product', 'item name', ...],
  value: ['market value', 'market price', 'tcgplayer market', ...],
  set:   ['set name', 'set', ...],
  qty:   ['quantity', 'qty', ...],
};
```

Headers are lower-cased and stripped of punctuation before matching; exact matches are tried first, then substring matches, in list order. If Collectr renames a column, add the new header name to the front of the relevant list. Users can also override the mapping in the UI after **Preview**, so a mismatch is never fatal.

Other parser notes:
- Quoted fields, embedded commas, `$` and thousands separators are handled.
- If the pasted text has no detectable value column, it falls back to manual mode: one item per line, value last, separated by `:`, `,` or a tab.
- Duplicate `name + set` rows (e.g. multiple conditions) keep the **last** row. Change `dedupe()` if you want max/avg instead.
- If Collectr's value column is a *total* rather than per-unit, tick **"Value column is a total"** in the mapping card and pick the quantity column.

## Changing default thresholds in code

Edit `DEFAULT_THRESHOLDS` at the top of the script:

```js
const DEFAULT_THRESHOLDS = {
  takePct: 15,          // TAKE when discount >= this % under tracked
  counterMaxOverPct: 5, // COUNTER band reaches up to this % OVER tracked
  counterTargetPct: 15, // suggested counter = tracked * (1 - this %)
  floorPct: 60,         // sanity PASS when discount >= this %
};
```

These are only the *defaults*. A user's edits are stored under `de.thresholds.v1` in `localStorage` and win over the code values until they hit **Reset to defaults**. If you change the semantics (not just the numbers), bump the storage key (`STORE_KEYS.thresholds`) so stale saved values don't apply.

## Storage keys

| Key | Contents |
|---|---|
| `de.prices.v1` | `{ updatedAt, items: [{ name, value, set? }] }` |
| `de.thresholds.v1` | user-edited thresholds |
| `de.log.v1` | evaluation log, newest first, capped at 5,000 entries |

Data is per browser, per device. Clearing site data wipes it — export the log first.

## Deploying (static hosting)

It's one file; any static host works. Pick one:

**Render (matches the rest of the stack)** — a `render.yaml` Blueprint is checked in: Dashboard → New → Blueprint → select this repo → Apply. (Or manually: New → Static Site → build command blank → publish directory `.`.) Free tier is fine.

**Netlify / Vercel** — drag the folder onto the dashboard, or connect the repo with no build step and output directory `.`.

**GitHub Pages** — repo Settings → Pages → source: `main` / root. Served at `https://<user>.github.io/deal-evaluator/`.

**Local / offline** — open `index.html` directly in a browser. Works from `file://`. On a phone, "Add to Home Screen" from the browser share menu gives it an app icon.

The only external dependency is the Inter font from Google Fonts; if that's blocked it falls back to the system font with no other change.

## Testing

No test harness is checked in. The pure functions (`evaluate`, `parseCSV`, `parseManual`, `detectCol`) can be exercised by extracting the `<script>` block; the initial build was verified with a jsdom smoke test covering paste → map → load → search → evaluate → threshold edit → export → clear.

## Out of scope (deliberately)

No live pricing APIs, no lifecycle/archetype classification, no reprint-risk signals, no accounts or sync. The log export is the only hook toward a future outcome-tracking dataset.
