# Revenue Manager — STR Revenue Dashboard

A live revenue-management dashboard for short-term rentals, running on your own
PriceLabs account. Compares each property's booking pace against its comp set,
recommends specific rate and minimum-stay changes, and pushes them back to
PriceLabs after you review them.

## Install

See **[INSTALL.md](INSTALL.md)** for all three options:

- **A — Skill** (recommended): Claude sets it up for you in Cowork.
- **B — Standalone artifact**: register the HTML in Cowork by hand.
- **C — Hosted website** on the PriceLabs Customer API: build guide only, not included.

| File | |
| :--- | :--- |
| `INSTALL.md` | Install options A, B and C |
| `SKILL.md` | Setup instructions Claude follows |
| `revenue-dashboard.html` | The dashboard itself, ~2,700 lines, self-contained |
| `revenue-rules-template.md` | Your strategy in human-readable form |
| `DASHBOARD-DESIGN.md` | Architecture reference — read before editing the logic |

## What it does

- **Portfolio** — every property ranked by what needs attention
- **This week's moves** — ranked recommendations with the reasoning and the numbers
- **By unit** — RevPAN vs market, pace by window, seasonal context, PriceLabs diagnostics
- **Calendar & pricing** — per-night prices, overrides, recommended price and minimum stay
- **Config** — every threshold the engine uses, with JSON export/import

Changes stage in a review tray. Nothing reaches PriceLabs until you push, booked
nights are never touched, and each pushed change pauses its own recommendation
for a week so you can see whether pace responds.

## Limitations — read before you rely on it

**What it is**

- **A rules engine, not judgement.** It compares your booking pace to the comp set
  and applies thresholds. That's arithmetic. It doesn't know about your reviews,
  photos, a renovation, a road closure or a local event PriceLabs hasn't flagged.
  The thresholds are a starting point, not tuned for your market — check them in
  the Config tab.
- **Built and tested on a small portfolio** in one market (Arizona desert). The
  seasonal defaults reflect that. Other markets need their own profile.
- **Not affiliated with or endorsed by PriceLabs.** It uses their public MCP
  connector, which can change without notice and break things.

**It changes live prices**

- Pushes go straight to PriceLabs and then to your channels. A wrong threshold or
  a bug can underprice real nights. Review every staged change before pushing.
- It respects your PriceLabs minimum and maximum, so **set a sensible floor** —
  that's your safety net.
- Overrides it creates stay in PriceLabs until removed. Use the "Clear overrides" control
  to hand pricing back to PriceLabs.

**Data gaps**

- **Market ADR isn't published by PriceLabs.** Market RevPAN is derived from the
  comp set's median asking price × market occupancy. It has matched PriceLabs'
  own Market Dashboard closely, but it's an approximation, and asking prices
  aren't achieved prices.
- **No individual competitor listings** — only the aggregate comp set.
- **No pricing factor breakdown** (seasonality %, last-minute discount, etc.).
  The connector doesn't expose it.
- **Same-time-last-year figures** stay empty until a listing has a full year of
  synced history.
- **PriceLabs' own base figure moves on refresh.** The dashboard pins the figure
  it priced from for 7 days after you push, so refreshes don't walk your price
  down. Recommendations on nights you pushed are held during that window.

**Where it runs**

- **Only inside Claude Cowork**, as an artifact. It's not a standalone web app;
  opening the HTML in a browser won't load data.
- **State is per browser.** Your change log, pins, hidden properties and config
  live in local storage. Clearing it or switching machines loses them — export
  your config from the Config tab.
- **Scale.** A full load is about `1 + 5 × properties` API calls against
  PriceLabs' 60/minute limit. Comfortable to ~15 properties, workable to ~30.
  Beyond that you need a server-side build on PriceLabs' batched REST API.

**Not advice**

This is a tool, not financial advice. You're responsible for your pricing
decisions. Provided as-is under the MIT license, with no warranty.

## License

MIT — see `LICENSE`.
