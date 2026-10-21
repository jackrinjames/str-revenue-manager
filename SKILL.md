---
name: revenue-manager
description: Set up a live short-term rental revenue management dashboard backed by the user's own PriceLabs account. Use when someone asks to build, install, or set up a revenue dashboard, STR pricing dashboard, or wants to manage their own Airbnb/Vrbo rates instead of leaving it entirely to PriceLabs' automation.
---

# Revenue Manager

Installs a live revenue-management dashboard for short-term rentals. It reads the operator's own PriceLabs account, compares each property's booking pace against its comp set, and recommends specific rate and minimum-stay changes that can be staged and pushed back to PriceLabs.

Everything runs against **their** PriceLabs connection. Nothing in this skill contains or reaches anyone else's data.

## Before you start

The dashboard needs the **PriceLabs MCP connector**. Check whether tools named `*get_listings`, `*get_listing_data` and `*get_listing_prices` are available.

If they aren't, walk the person through connecting it and stop until they confirm:

1. In PriceLabs: **Account Settings → AI Connector (MCP)**, copy the MCP URL and Client ID.
2. In Claude: briefcase icon → **Connectors** → **+** → **Add custom connector**.
3. Name it `PriceLabs MCP`, URL `https://mcp.pricelabs.co/mcp`, then under **Advanced settings** paste the Client ID.
4. **Connect**, sign in, approve **read and write** access.

Read-only access makes the dashboard's push controls fail with a 403. Say so up front.

If they're a team member rather than the account admin, the admin must enable MCP for them under **Account Settings → Team Settings → edit member → MCP Access → Allow**. Their listing-level permissions carry over automatically.

## Setup

### 1. Find their connector id

Every PriceLabs connection has its own MCP server id. Look at the actual tool names available to you — they take the form `mcp__<id>__get_listings`. The prefix you need is everything up to and including the trailing double underscore, e.g. `mcp__a1b2c3d4-...__`.

Confirm it works by calling `get_listings` before going further. If that fails, the connector isn't properly authorised.

### 2. Build their copy

Read `revenue-dashboard.html` from this skill directory. Replace the single occurrence of:

```
mcp__PRICELABS_CONNECTOR_ID__
```

with their actual prefix. That string appears once, in `const PL_DEFAULT`.

Write the result to the outputs directory, then register it:

```
create_artifact
  id: "str-revenue-manager"
  html_path: <the file you just wrote>
  mcp_tools: [ every PriceLabs tool listed below, using their prefix ]
```

The `mcp_tools` list must use their connector id, not the placeholder:

`get_listing_data`, `get_listing_performance_metrics`, `get_listing_prices`, `get_listing_date_overrides`, `get_listings`, `get_bookings_report`, `update_listing_data`, `update_listing_date_overrides`, `delete_listing_date_overrides`, `refresh_listing_pricing`, `get_listing_health_and_recommendations`, `get_neighbourhood_data`, `diagnose_no_bookings`, `get_available_nudges`, `accept_nudge`

### 3. Ask about their portfolio before interpreting anything

**Do not treat every listing in their PriceLabs account as a property to manage.** Accounts routinely contain duplicates, mid-term rentals where nightly pricing doesn't apply, and former clients. Call `get_listing_data` and ask them, listing by listing, which are actively revenue-managed as short-term rentals.

Properties they don't manage should be hidden in the dashboard. Getting this wrong produces confident recommendations about properties that shouldn't be touched.

### 4. Set their strategy

Ask two questions before they start using it:

- **Booking window and posture.** The default is a balanced 0–90 day profile: hold inside ±8 points of market, cut when behind, raise when ahead. Aggressive fill and ADR-protective variants are reasonable alternatives.
- **Markets.** If they operate in more than one, create a profile per market in the Config tab. Thresholds travel between markets; seasonality does not. A desert market's April cliff is wrong for a coastal one.

Then copy `revenue-rules-template.md` into their workspace as `Revenue Management/revenue-rules.md` and fill in their properties and markets. The dashboard's Config tab and that file should agree — the file is what a human reads, the Config tab is what the code reads.

### 5. Offer the weekly brief

Offer to create a scheduled task that runs Monday mornings, pulls their pace data, and writes a short brief with the specific moves to make that week. The dashboard recommends continuously; the brief is what makes it a habit.

## How to explain it

Frame it accurately: **it's a rules engine, not judgement.** It compares occupancy to the comp set and applies thresholds. That's arithmetic, and the person should know it — both because it's honest and because it means the thresholds are theirs to change.

The three things worth showing them first:

- **Portfolio tab** ranks properties by what needs attention. Start every session here.
- **Staging tray.** Nothing reaches PriceLabs until they review and push. Booked nights are never touched.
- **Config tab** holds every threshold, with export and import so a tuned profile can be shared.

## Known limitations — state these plainly

- **Market ADR** isn't published by PriceLabs. Market RevPAN is derived from the comp set's median asking price. It matches PriceLabs' own Market Dashboard closely but is an approximation.
- **Individual competitor listings** aren't exposed through the connector — only the aggregate comp set.
- **Pricing factor breakdowns** exist in PriceLabs' REST API but not the MCP tool.
- **Same-time-last-year** figures stay empty until a listing has a full year of synced history.
- **Scale.** A full load is roughly `1 + 5N` API calls against PriceLabs' 60/minute ceiling. Comfortable to about 15 properties, workable to 30. Past that, a server-side deployment using the Customer API's batched endpoints is the better shape.

## Editing it later

`DASHBOARD-DESIGN.md` in this skill directory documents the architecture, the data model, and — importantly — three design decisions that look arbitrary but are fixes for real bugs. Read it before changing the recommendation logic. Reverting any of the three reintroduces a bug that looks correct every time it fires.
