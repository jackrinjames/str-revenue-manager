# Installing

Three ways to run it. Pick by how much setup you want.

| | Option A — Skill | Option B — Standalone artifact | Option C — Hosted website |
| :--- | :--- | :--- | :--- |
| Runs in | Claude Cowork | Claude Cowork | Any browser |
| Data source | PriceLabs MCP connector | PriceLabs MCP connector | PriceLabs Customer API |
| Setup effort | ~5 min, guided | ~10 min, manual | Developer work |
| Extra cost | None | None | PriceLabs API: $1/listing/month, plus hosting |
| Status | **Supported** | **Supported** | **Not included — build guide only** |

All three need a PriceLabs account with listings syncing.

---

## Step 0 — Connect PriceLabs to Claude (Options A and B)

1. PriceLabs → **Account Settings → AI Connector (MCP)**. Copy the MCP URL and Client ID.
2. Claude → **Connectors → + → Add custom connector**.
3. Name `PriceLabs MCP`, URL `https://mcp.pricelabs.co/mcp`, paste the Client ID under **Advanced settings**.
4. **Connect**, sign in, approve **read and write** access.

Read-only access lets the dashboard load but makes every push fail with a 403.

Team members: the account admin enables MCP under **Account Settings → Team Settings → edit member → MCP Access → Allow**.

---

## Option A — Install as a skill (recommended)

Claude does the setup and asks about your portfolio and strategy along the way.

1. **Code → Download ZIP** on this repo, and unzip it.
2. In Claude Cowork, select the unzipped folder and say:

   > Follow SKILL.md in this folder to set up my revenue manager dashboard

Claude finds your connector id, builds your copy, registers it as an artifact,
asks which listings you actively manage, and offers a weekly Monday brief.

---

## Option B — Install the artifact by hand

Same dashboard, no guided setup.

1. **Find your connector prefix.** In Cowork, ask:

   > What is the full tool name prefix for my PriceLabs MCP connector?

   It looks like `mcp__a1b2c3d4-...__` — everything up to and including the
   trailing double underscore.

2. **Edit the file.** Open `revenue-dashboard.html` and replace the single
   occurrence of `mcp__PRICELABS_CONNECTOR_ID__` with your prefix. (You can skip
   this and set it later in the dashboard's **Config** tab.)

3. **Register it.** Put the edited file in a Cowork folder and say:

   > Create an artifact with id `str-revenue-manager` from revenue-dashboard.html,
   > allowing these PriceLabs tools: get_listing_data, get_listing_performance_metrics,
   > get_listing_prices, get_listing_date_overrides, get_listings, get_bookings_report,
   > update_listing_data, update_listing_date_overrides, delete_listing_date_overrides,
   > refresh_listing_pricing, get_listing_health_and_recommendations,
   > get_neighbourhood_data, diagnose_no_bookings, get_available_nudges, accept_nudge

   Claude needs to call each tool once first to confirm it works — let it.

4. **First run.** Open the artifact, then in the **Portfolio** tab hide any listing
   you don't actively manage (duplicates, mid-term rentals, former clients). Review
   thresholds in **Config**.

Opening `revenue-dashboard.html` directly in a browser **will not work** — it reads
data through Cowork's connector bridge (`window.cowork.callMcpTool`), which only
exists inside Cowork.

---

## Option C — Host it as a website (PriceLabs Customer API)

**This repo does not include a working hosted version.** What follows is how to
build one. It needs someone comfortable deploying a small server.

### Why it needs a server

The Customer API authenticates with a secret API key. That key can read and
change live prices on every listing in your account, so it **must never be put in
browser code**. The page talks to your own small backend; only the backend holds
the key.

```
Browser (dashboard)  →  your backend (holds API key)  →  api.pricelabs.co
```

### What you need

- A PriceLabs **API key**: Account Settings → **API Details** → **Get PriceLabs API Key**.
  PriceLabs charges $1 per syncing listing per month for API access.
  API docs: https://developers.pricelabs.co
- Somewhere to run the backend — e.g. a Cloudflare Worker, Vercel/Netlify function,
  or any small Node server — with the key stored as a secret/environment variable.
- **Access control** on the site (Cloudflare Access, basic auth, or a login). An
  open URL would let anyone change your prices.

### Build steps

1. **Backend.** Expose one endpoint per operation the dashboard uses — list
   listings, listing settings, prices, overrides (read / write / delete), refresh.
   Each one adds the API key header and forwards to the PriceLabs Customer API.
   Use the batched price endpoint so one call covers many listings.
2. **Swap the data layer.** Every PriceLabs call in the dashboard goes through one
   function, `call(tool, args)`. Replace the `window.cowork.callMcpTool(...)` line
   inside it with a `fetch()` to your backend, and map each MCP tool name to the
   matching backend route. Check response shapes — the REST API and the MCP
   connector don't return identical field names.
3. **Remove Cowork-only features** or reimplement them server-side: the AI
   diagnostics panel (`askClaude`) and the neighbourhood / health / diagnose calls
   are MCP-only and have no direct Customer API equivalent.
4. **Persistence.** The dashboard stores config, change log and anchor pins in
   browser localStorage. That's per device. For a shared or multi-device site, move
   them to a database.
5. **Scale (optional).** For 30+ listings, sync on a schedule (e.g. every few
   hours) into a cache and serve the dashboard from that, instead of calling
   PriceLabs on every page load.
6. **Test with pushes off first.** Load real data read-only and compare against
   the Cowork version before enabling writes.

### Honest expectations

Expect a few days of developer time. Market data the MCP connector provides
(neighbourhood comp set, health checks) may not be available through the Customer
API, which would leave the market-comparison features thinner than the Cowork
version.
