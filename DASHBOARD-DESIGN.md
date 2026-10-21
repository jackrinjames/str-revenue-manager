# Revenue Manager — Backend Design

*Reference for editing the dashboard in a fresh session without re-deriving context. Written August 2026.*

Read this before changing anything. Several of the design choices below look arbitrary and are not — they're fixes for bugs that took a while to find, and reverting them reintroduces the bug.

---

## What and where

A single self-contained HTML artifact, ~2,700 lines, registered in Cowork as artifact id **`str-revenue-manager`**.

| | |
| :---- | :---- |
| Working copy | `outputs/revenue-dashboard.html` in the session scratch directory |
| Registered via | `mcp__cowork__update_artifact` with `id: "str-revenue-manager"` |
| Data source | PriceLabs MCP only — no Hospitable, no other connector |
| Decision rules | Mirror `Revenue Management/revenue-rules.md` |
| Companion | `Revenue Management/Web App/` holds an untested Cloudflare Worker port |

To edit: modify the HTML file, then call `update_artifact` with the same id and the file path. Editing the file alone does nothing.

### Connector id is per-account

```js
const PL_DEFAULT='mcp__PRICELABS_CONNECTOR_ID__';
let PL=localStorage.getItem('rm_connector')||PL_DEFAULT;
```

Every PriceLabs connection gets its own MCP server id. The default is the original owner's. Anyone else must set theirs in the Config tab. `main()` catches "tool not found" and says so explicitly rather than looking like an outage.

---

## Data sources

Only `window.cowork.callMcpTool` is available — the artifact cannot make arbitrary HTTP requests. This rules out BNBCalc, AirDNA and anything else needing a key or custom headers. Those require the Worker.

### Fast path — blocks first render, 3 calls per property

| Tool | Used for |
| :---- | :---- |
| `get_listing_data` | base/min/max, `push_enabled`, snapshot occupancy vs market at 7/30/60 |
| `get_listing_performance_metrics` | occupancy, ADR, RevPAR, MPI, revenue by days-from-today; `market_level.occupancy` |
| `get_listing_prices` | daily calendar — price, `uncustomized_price`, `user_price`, min_stay, booking status |
| `get_listing_date_overrides` | committed DSOs — the truth about whether a push landed |

### Background enrichment — never blocks render, 2 calls per property

| Tool | Used for |
| :---- | :---- |
| `get_neighbourhood_data` | comp-set size, price percentiles, surge dates, seasonal positioning |
| `get_listing_health_and_recommendations` | per-month season labels and booking lead time |

### On demand

`get_bookings_report` (ALOS, booking window), `diagnose_no_bookings`, `get_available_nudges`, `accept_nudge`.

### Writes

`update_listing_data` (base/min/max), `update_listing_date_overrides`, `delete_listing_date_overrides`, `refresh_listing_pricing`.

### Known dead ends — don't rebuild these

- **`reason` breakdown.** The Customer API's `listing_prices` accepts `reason: true` and returns the full factor chain. **The MCP tool does not accept that parameter** — it takes only listing_id, pms_name, date_from, date_to. The flag is silently ignored. A price-breakdown feature was built on this and had to be removed. `reasonText()` is a stub returning `''`.
- **Individual competitor listings.** `include_competitors: true` returns the same payload as without it. PriceLabs' docs point at a separate endpoint the MCP doesn't expose.
- **Listing Optimizer.** Returns 404 unless subscribed.
- **Market ADR as a field.** Doesn't exist. See the derivation below.
- **`get_neighbourhood_data` with `mode:'detail'` + `include_prices`** returns ~240k characters and blows the context window. `fields:'monthly_averages'` returns an empty `additional_details`.

---

## The four decisions that must not be reverted

### 1. Recommendations anchor to `uncustomized_price`, never `price`

```js
d.anchor = Number(d.uncustomized_price)>0 ? Number(d.uncustomized_price) : d.price;
d.eff    = override_price_if_fixed ?? d.price;
d.vsAnchor = (d.eff - d.anchor) / d.anchor * 100;
```

`price` absorbs your own overrides. Computing "raise 8%" from it means the next push computes 8% off the already-raised number — an unbounded ratchet that looks correct every single time. This was a live bug: one night climbed 158 → 167 → 180 and would have kept going.

`uncustomized_price` is PriceLabs' own figure and doesn't move when you push. Recommendations target a *position* relative to it ("sit 6% above PriceLabs' $154"), so applying the same recommendation twice is a no-op.

**Any new recommendation logic must use `d.anchor`, not `d.price` or `d.eff`.**

### 2. Date overrides are pushed as `percent`, not `fixed`

```js
{date, price: String(pct), price_type: 'percent'}
```

Fixed-dollar overrides freeze a night against PriceLabs' seasonality and demand adjustments. Percent overrides ride on top, so their engine keeps working underneath, and re-pushing the same percentage replaces rather than compounds.

Exception: manual per-date entry uses `fixed`, because the user typed a specific dollar amount.

### 3. "Already done" compares the resulting price, not percentages

```js
const already = Math.abs(p - Math.round(d.eff)) < 1;
```

Comparing target percentages misses floor and ceiling clamps. A target of −15% off $101 computes to $86, gets clamped up to a $95 floor, and equals the current price — a genuine no-op that rendered as a green "raise 0%" button. Clamped nights carry a `FLR`/`CAP` marker and explain themselves.

---

### 4. The anchor is pinned at push time

`uncustomized_price` is *not* fully stable: PriceLabs recomputes it on every
refresh, and on nights carrying your override it has come back lower than it was
when you pushed. Measuring the next recommendation off the new figure produces a
downward ratchet — push $126, refresh, recommendation drops to $108, push, repeat.

Fix: every staged price change carries `pins` ({date, anchor, price, type}).
After a successful push they're saved to localStorage (`rm_pins`). In
`finishUnit`, while the pin is younger than `coolDays` and the live override is
still the one we pushed, `d.anchor` uses the pinned value (`d.anchorLive` keeps
PriceLabs' current figure for display). Pins are dropped when they expire or a
different override replaces ours. Nights pushed before pins existed are held via
the change log for the cooling period instead.

## Data model

`UNITS[]`, one object per property, built by `loadUnit()` → `finishUnit()`, later augmented by `enrichUnit()` → `finishUnit()`.

```
id, pms, name, city, beds, cur
min, base, max, recBase, push, lastPush, stale
profile          resolved market profile (see Config)
cum{dfd}         your occupancy by days-from-today
mkt{dfd}         market occupancy — the core signal
adr{}, revpar{}, rev{}, mpi{}
stlyRevpar{}, stlyAdr{}, stlyOcc{}
trail{}, year{}  trailing 90d and 12mo actuals
days[]           daily calendar, each with .eff .anchor .vsAnchor
ovr{date}        committed date overrides
wins[]           per-window {mine, mk, gap, d:decision}
surge{date}      market surge dates from neighbourhood insights
seasons[]        per-month {month, season, bwMin, bwMax}
derivedWindows   booking windows sized from lead time
winOverride      user's manual windows, persisted per listing
comp             comp-set summary_table
stays[]          reservations, loaded on demand
enrich{market,health}   'pending' | 'ok' | error message
_open{}, _acts   memoisation — cleared whenever config changes
```

### Derived market RevPAN

PriceLabs publishes market occupancy but no market ADR. The comp-set median asking price stands in:

```js
marketRevpan(u,dfd) = marketAdr(u) * u.mkt[dfd] / 100
```

Validated against PriceLabs' own Market Dashboard: 59% × $182 = $107.38 published, versus 62.2% × $173 = $107.61 derived. Within 0.2%. It remains an asking price rather than achieved ADR, and the UI says so.

---

## Decision engine

`decide(win, gap, C, wins, mkOcc, rev)` returns `{act, sev, pct, range, why}`.

Order of application — this sequence matters:

1. **Pace band** on `gap` (your occupancy minus market, in points) sets direction and magnitude.
2. **`cap0`** — thin-market guard. Market occupancy below `lowDemandPct` caps cuts at `lowDemandCutCap`. Discounting into a dead market costs rate without creating demand.
3. **`revAdjust`** — RevPAN brake and boost:
   - RevPAN index < 1 **and** your ADR already below market → cut capped at `revpanBrakeCap`. Price isn't the problem, so cutting deepens the revenue gap.
   - RevPAN index < 1 on a **raise** → add `revpanBoostPct`. Leading occupancy while trailing revenue is the underpricing signature.
   - Index ≥ 1 → note added, no change.

`cap = d => revAdjust(cap0(d))`. Every cut and raise branch returns through `cap()`. Hold and watch return bare — they have no percentage to adjust.

`revPicture(u,w)` supplies `{idx, adrBelow}`. Returns null when market data is missing, in which case both adjusters no-op and behaviour matches the pre-RevPAN engine.

### Booking windows

Derived per property from PriceLabs' observed monthly lead time, snapped to the days-from-today points occupancy is actually reported for (`DFD_POINTS = [7,15,30,45,60,90,120]`). Occupancy can only be compared at those points, so arbitrary boundaries don't work.

A market booking 0–5 days out yields 0–15 / 15–30 / 30–60. One booking 4–25 days out yields 0–45 / 45–90 / 90–120. Precedence: `winOverride` → `derivedWindows` → `CFG.windows`.

---

## Configuration

`CFG` in localStorage under `rm_cfg`, seeded from `CFG_DEFAULTS`. `CFG_FIELDS` drives the Config UI — adding a field there makes it editable automatically.

**Market profiles** match by city or state; last match wins, `default` is the fallback. `cfgFor(u)` returns base config with that profile's overrides layered on.

**Everything that reads a threshold must use `cfgFor(u)`, not the global `CFG`.** A previous audit found the portfolio table using global config while unit cards used per-profile, so the same property read differently in two views. Several MPI and floor thresholds were also hardcoded literals while the Config tab advertised them as editable.

`recalc()` re-derives windows, orphans and decisions after a config change without refetching. It must clear `_acts` and `_open`.

---

## Runtime architecture

**Progressive load.** Properties paint as they arrive rather than after all complete. With the rate limiter, twenty properties take over a minute end to end; first paint is ~20ms.

**Rate limiter.** 50 requests per rolling 60s against PriceLabs' 60/min ceiling. A full load is `1 + 5N` calls, so twenty properties is ~101 requests. Retries re-enter the gate.

**Lazy rendering.** `renderAll()` builds only the tab in `CUR`. Rebuilding all five on every state change was wasteful at scale.

**Cache.** Last good load in `rm_cache`, trimmed to the eleven day-fields actually read (2.85MB → 1.37MB at 20 properties × 365 days). Degrades to a 120- then 60-day horizon on quota errors rather than losing the fallback.

**Failures must be visible.** `enrichUnit` records `u.enrich.{market,health}` as `'pending' | 'ok' | error`. Every section that depends on enrichment states which of those it's in and offers `retryEnrich(id)`. Silent empty returns caused a reported bug where market metrics vanished with no explanation anywhere.

---

## Testing

No browser is available — **Playwright cannot install its browsers in this sandbox** (host dependencies, no sudo). Don't spend time rediscovering that.

The approach that works: extract the `<script>` block, stub `document` / `window.cowork` / `localStorage`, and drive functions directly in Node.

```js
const html = fs.readFileSync('revenue-dashboard.html','utf8');
let js = html.match(/<script>([\s\S]*)<\/script>\s*<\/body>/)[1];
js = js.replace(/main\(\)\.then\([^;]*\);?/,'');   // strip boot
const api = new Function(js + '; return {main, decide, recPrice, getUnits:()=>UNITS};')();
```

Two harness traps to avoid:

- Stubbing `setTimeout` to fire immediately makes `rateGate` spin forever, because time never advances. Use real timers and shrink `RATE_WINDOW` instead.
- Assertions like `(x || '') !== undefined` are true by construction. One was written and reported as a pass. Break the thing deliberately and confirm the test goes red.

Always verify after a change: syntax, no dead functions, no undefined inline handlers, no `getElementById` for ids that don't exist, no config field declared but never read, and column counts matching between `<th>` and `<td>` (remembering that `gapCell()` emits cells a naive regex won't count).

---

## Open items

- `Web App/worker.js` is written from documentation and **has never been executed**. It also mirrors the MCP's one-listing-at-a-time shape rather than using the Customer API's batched `listing_prices`, which is the main reason the Worker would be faster at scale. Rewrite that before deploying.
- BNBCalc comp-set panel — $0.20 per report, fills the named-comp-set gap in `revenue-rules.md`. Requires the Worker; unreachable from the artifact.
- Market RevPAN uses asking price as an ADR proxy. Fine, but labelled as an approximation.
- Same-time-last-year columns stay empty until a listing has a full year of synced history.
