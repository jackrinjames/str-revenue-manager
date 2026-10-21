# Revenue Rules — [Your Name]

*The decision logic behind the weekly rate review. Edit these thresholds anytime. The dashboard's Config tab holds the same numbers in machine-readable form — keep the two in step.*

**Last updated:** [date]
**Strategy profile:** [e.g. 0–90 day window, balanced]

---

## Portfolio

**Actively revenue-managed:**

| Unit | Market | Beds/Baths | Sleeps | Notes |
| :---- | :---- | :---- | :---- | :---- |
| | | | | Seasonality, event drivers, anything unusual |

**In PriceLabs but out of scope** — hide these in the dashboard so they don't generate recommendations:

| Unit | Why excluded |
| :---- | :---- |
| | e.g. mid-term rental, former client, duplicate record |

---

## The core metric: pace vs market

Each week, for each unit, compare **your occupancy** against the **comp-set occupancy** for the same forward window.

```
Pace gap = (Your occupancy % for window) − (Market occupancy % for window)
```

Run it across the windows separately — they call for different responses.

| Window | What it tells you | Response speed |
| :---- | :---- | :---- |
| Near | Perishable inventory | Act this week |
| Middle | Whether current pricing is competitive | Adjust, re-check next week |
| Far | Early demand signal, mostly noise | Watch only |

**Size these windows to how your market actually books.** A market where guests book five days out can't be judged on a 60-day window. The dashboard derives them from PriceLabs' own monthly lead-time observation.

---

## Action thresholds

### Near window

| Pace gap | Action |
| :---- | :---- |
| Behind by 15+ pts | Cut 12–18% on unbooked nights. Drop min-stay. Clear orphan gaps. |
| Behind by 8–15 pts | Cut 8–12%. Check last-minute discounts are enabled. |
| Within ±8 pts | Hold. You're tracking market. |
| Ahead by 8–15 pts | Raise 5–8%. |
| Ahead by 15+ pts | Raise 10–15%. You priced too low earlier. |

### Middle window

| Pace gap | Action |
| :---- | :---- |
| Behind by 15+ pts | Cut 8–12%. Check min-stay first — a long minimum on soft midweek is often the real blocker. |
| Behind by 8–15 pts | Cut 5–8%. |
| Within ±8 pts | Hold. |
| Ahead by 8+ pts | Raise 5–10%. Consider tightening min-stay to protect weekends. |

### Far window

Act on only two things:

1. **A high-demand date is unpriced for it.** Events and holidays should carry a date-specific override well above base. An event date sitting at base price is the highest-value fix on the board.
2. **You're 20+ pts behind.** That's structural, not timing. Lower the base price rather than adding date overrides.

---

## Rules that override the table

- **Never change a booked night.**
- **Don't cut into a rising market.** If market occupancy is climbing while yours is flat, that's a listing problem — photos, reviews, min-stays, visibility — not a price problem.
- **Don't discount into a dead market.** Below roughly 35% market occupancy the demand isn't there to win back with price. Cap cuts and put the effort into peak dates.
- **Don't cut when your rate is already below market.** If RevPAN trails the comp set and your ADR is already under it, price isn't what's costing you.
- **Orphan nights are worth clearing.** A 1–2 night gap between bookings should be priced to move, 20–30% off with a 1-night minimum. Revenue on an unsellable night is zero.
- **Floor discipline.** Never discount below your minimum. If the rules say cut and you're at floor, the answer is min-stay or a listing fix.
- **One change per unit per week**, unless it's urgent. Constant tinkering makes it impossible to tell what worked.
- **Seasonal transitions get a manual look.** Algorithms lag the turn. Check base prices going into shoulder season.

---

## Success metric

**RevPAN — revenue per available night — not occupancy.** Occupancy alone can be bought with rate. Winning means meeting or beating the market's RevPAN.

Watch the two levers together: occupancy up while ADR falls is buying volume, and only a win if RevPAN holds.

---

## Weekly review checklist

1. Pull occupancy vs market for each unit across all windows.
2. Flag any window outside ±8 pts.
3. Check the next 90 days for event dates without an override.
4. Scan for orphan nights.
5. Confirm PriceLabs actually synced last week's changes.
6. Make the calls. Log what changed and why.

---

## Change log

*Record each move so pace changes can be attributed to a decision rather than guessed at.*

| Date | Unit | Change | Reason | Result (checked next week) |
| :---- | :---- | :---- | :---- | :---- |
| | | | | |
