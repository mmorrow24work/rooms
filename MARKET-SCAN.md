# Off-the-shelf market scan (room booking + optimisation)

Pricing is list/indicative as of Q1–Q2 2026, mostly USD, and almost always
discounted 20–40% on enterprise contracts. Treat as ballpark; confirm on a quote
for ~15 rooms in the UK (£/€).

## The key finding

For your two specific pains there is a clear split:

* **Pain 1 — full-day booking fails because of a stray 1h booking.** No
  mainstream tool *automatically defragments* the day. They mitigate it with
  no-show **auto-release** (frees rooms when nobody checks in) and **manual
  drag-and-drop** rescheduling. The active "broker moves meetings to free a whole
  room/day" behaviour is not an off-the-shelf feature.
* **Pain 2 — no optimisation before/after booking.** Tools optimise at the
  *point of booking* (suggest the right-sized room, suggest alternative
  rooms/times) and give *analytics after the fact* (utilisation, ghost meetings).
  "AI scheduling" in this category means suggestions and predictive nudges — not
  constraint re-solving of the existing schedule.

So: buy an off-the-shelf tool for the booking UX, calendar sync, panels,
analytics and governance — and the re-optimisation broker remains a custom build
(this PoC). The two are complementary: the broker could sit on top of a tool's
data via Microsoft Graph.

## Vendors

| Tool | Model | Indicative price | Notes for your case |
|---|---|---|---|
| **Microsoft Places** | Per managed space (since 1 Apr 2026) | Rooms are bookable free in Outlook today; Places management/analytics needs **one Teams Shared Space licence per managed room** (MTR-licensed rooms already covered). The old Teams-Premium-per-user (~$10/user/mo) gate is gone. | Lowest friction — lives in Outlook/Teams you already run. Analytics are basic; no re-optimisation. Strong default to evaluate first. |
| **Skedda** | **Per space**, unlimited users | ~$99/mo Starter (15 spaces), ~$149 Plus, ~$199 Premier; ~$349/mo for ~45 spaces | Per-space pricing fits "many staff, ~15 rooms" far better than per-user tools. Deep rules engine, Outlook/Teams sync, custom floor plans. G2 #1 space-management 3 yrs running. |
| **Robin** | **Per employee/mo** | ~$3–5 Essentials, $5–8 Pro, $8–12+ Enterprise (annual) | Good room/desk UX + abandoned-meeting auto-release + 100+ KPIs. Per-employee model gets expensive when many people share few rooms. |
| **Envoy** | Per user / module | Quote-based | Workplace platform (rooms + desks + visitors). Analytics-led; rooms one module of many. |
| **Joan** | Per room (software) + hardware | Software roughly €10–15/room/mo; e-ink panels are a per-door hardware cost | Known for low-power e-ink room displays. Narrower software feature set; best where the physical at-door experience matters. |
| **Condeco (Eptura)** | Enterprise, per-resource | Quote-based | Heavyweight enterprise room booking + analytics; steeper learning curve and cost. Overkill for 15 rooms unless part of a wider Eptura IWMS rollout. |

**Market rules of thumb:** software-only room booking ≈ $15–40/room/month;
room panels ≈ $300–1,200/door + install; occupancy sensors ≈ $150–600/device +
$3–10/room/mo; implementation/services $2k–15k at scale.

## Recommendation shape

1. If "stay inside Microsoft" wins on cost/IT effort → **Microsoft Places**
   (licence the ~15 rooms), accept basic analytics.
2. If you want a better booking UX + rules + utilisation analytics without
   per-head cost → **Skedda** (per-space is the right model for you).
3. Keep the **custom broker** for the re-optimisation/defrag capability none of
   them offer, fed from whichever tool's data via Microsoft Graph.
