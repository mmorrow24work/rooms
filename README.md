# rooms ask part 1

At my workplace, we have around 15 meeting rooms which can be booked using MS Office, users can view the rooms availabilty and resouces such number of seats, whiteboard, video conferencing system, 60 inch monitor, room format ( lecture or round-table) - then invite the available room to the meeting, if all goes well, the user will recieve an email confirming the reservation.

This works, but i feel like it could be improved in a number of ways such as:

1/ if i want to book a meeting room for an entire day - it often fails because a room might already have a 1 hr booking already.

2/ there is no optimisation in place either before meeting rooms are booked or after

I envisage a solution which uses a broker to allocate rooms based on resources, duration, priority level e.g. 1=client meeting, 2=recruitment interview, 3=team meeting, 4=one to one meeting, 5=solo or quiet time - it also allows users with a special code to jump the queue or even cancel existing bookings in extreme cases.

The broker should be able to move meetings around if neccessary to optimise the resources, and scheduled times so long as the minimal meeting requirements are met and the users agree to the changes.

Use docker containers hosted by WSL2 to create a broker system with a web page frontend which allows authenticated users to make bookings based on the examples provided.

Also create a backend simulation of the meeting rooms to test and demo the solution. I want to be able evaluate how effective the broker is in terms of room optimation and time saved booking a room. I want to run scenarios where an un-optimsed schedule is analysed by the broker and then changes are made and the improvment is quantified in terms of percentage resources gained or made avaulable - e.g. with broker optimisation we have gained the equivalent of 3x 10 seat rooms.

Feel free to suggest ideas that might be worth considering.

Find off the shelf solutions that can meet the requirements, and get pricing.


---

This is a meaty one — a custom optimising broker plus a market scan. Let me research the off-the-shelf options and current pricing first (that's live data), then build you a runnable PoC and validate the optimiser with real numbers.

Good — Microsoft Places context is clear. Let me get current pricing on the main dedicated platforms.

Now the key differentiator — whether any off-the-shelf tool actually *re-optimises* (moves bookings), plus Joan/Condeco pricing.

I have what I need on the market. Now let me build and *validate* the broker — the optimiser is the core, so I'll get OR-Tools running on a real generated scenario before wrapping it in the app.

OR-Tools is good. Let me check the frontend design skill since I'm building a UI, then build the core.

Skill noted — I'll make the schedule board the signature element. Now let me build the backend core (domain, simulator, optimiser, metrics) and validate it on a real generated scenario.

Now the optimiser — the CP-SAT model that re-rooms and re-times bookings to free whole rooms while right-sizing and minimising disruption.

Now the metrics engine that quantifies the improvement (room-equivalents freed, seat-hours reclaimed, full-day availability, admin time saved).

Let me validate the core end-to-end on a generated scenario before building the API and UI.

Excellent — real numbers: incumbent uses 13 rooms (2 free), broker consolidates to 4 rooms (11 free), waste 58.6%→14.9%, full-day-bookable rooms 2→11. That's a compelling demo. The solver maxed its 10s budget at FEASIBLE though — let me warm-start it from the incumbent so it's fast and provably good.

Solution quality is stable from ~2s (4 rooms, ~27 moves); I'll default the API to a 4s budget. Core validated. Now the auth, API, and store.

Let me smoke-test the API in-process before wiring the frontend and Docker.

Full API works end-to-end (book → propose 22 moves → apply → 3 rooms used, 12 free). Now the frontend — a scheduling-board console where the board is the signature element and freed rooms light up when the broker runs.

Now the Docker/WSL2 packaging and docs.

Let me verify the stack runs over real HTTP (not just in-process) and the payloads match what the frontend expects.

All HTTP endpoints and RBAC work (standard user gets a proposal, 403 on apply). Let me add a standalone evaluation CLI (batch scenarios → quantified report), then the README, and package it.

Consistent across seeds: ~8 rooms freed, ~38pt waste reduction, ~1.6 room-days reclaimed. Now the README and packaging.

Built, validated, and packaged. Here's the summary.

## What I built

A two-container stack (`docker compose up --build`, runs on WSL2): an nginx frontend + FastAPI backend. The backend has a **CP-SAT constraint optimiser** (Google OR-Tools) as the broker core, a greedy single-booking allocator, a token-auth layer with the special-code privilege, a messy-schedule simulator, and a metrics engine. App on `:8080`, API docs on `:8000/docs`.

The broker does the thing none of the products do: it takes an existing schedule and **re-rooms/re-times bookings to free whole rooms**, weighted so it (1) frees whole rooms first → full-day bookings become possible, (2) right-sizes meetings into the smallest adequate room, (3) penalises moving a booking *scaled by priority* so a P1 client meeting is far more protected than a P5 focus block. Time-locked and pinned meetings are respected. It returns a *proposal* (moves + before/after metrics); nothing changes until applied — your "users must agree" requirement. Override-code holders queue-jump and force-cancel.

I validated it on generated scenarios (15 rooms, 26 meetings/day, 5 seeds). Consistent results: **~8 whole rooms freed**, wasted seat-capacity cut **~38 percentage points** (≈60% → ≈20%), full-day-bookable rooms going from 2–4 up to 10–12, and ~1.6 average-room-days of capacity reclaimed — i.e. your "equivalent of N rooms" headline, computed directly. There's also a CLI for batch sweeps (`python -m app.evaluate --n 26 --seeds 5`) and a tunable admin-time-saved model.

The demo loop and the optimisation model are written up in the README; the frontend board glows amber on the rooms the broker frees.

## Off-the-shelf — the key finding

I scanned the market (full table in MARKET-SCAN.md). The honest answer: **no mainstream tool does active re-optimisation/defragmentation.** They handle your pains only partially — via no-show **auto-release** and **manual drag-and-drop** rescheduling — and their "AI" is point-of-booking *suggestions* plus after-the-fact analytics, not constraint re-solving of the live schedule. So the sensible shape is: buy a tool for the booking UX/sync/panels/analytics, keep the broker custom (this PoC), and feed it via Microsoft Graph.

Indicative 2026 pricing (USD list, typically 20–40% off on enterprise deals):

| Tool | Model | Ballpark |
|---|---|---|
| Microsoft Places | per managed space | Rooms bookable free in Outlook; Places management/analytics needs one Teams Shared Space licence per managed room [Essential](https://www.essential.co.uk/blog/articles/new-microsoft-places-licencing-explanation/) (MTR rooms already covered) |
| Skedda | per space, unlimited users | ~$99/mo for 15 spaces, $149 Plus, $199 Premier [Capterra](https://www.capterra.com/p/132372/Skedda-Bookings/) |
| Robin | per employee/mo | ~$3–5 Essentials, $5–8 Pro, $8–12+ Enterprise [Vendr](https://www.vendr.com/marketplace/robin) |
| Joan | per room + e-ink hardware | ~€10–15/room/mo software + per-door panels |
| Condeco (Eptura) | enterprise, per resource | quote-based |

For ~15 rooms shared by many staff, **per-space pricing (Places or Skedda) beats per-employee (Robin)** by a wide margin. Given you're already on MS Office, the April 2026 switch to resource-based licensing [HANDS ON tek](https://handsontek.net/microsoft-places-licensing-changed-matters/) makes Places the lowest-friction thing to evaluate first.

## Ideas worth considering

- **Microsoft Graph integration** as the real productionisation path: read live room-mailbox availability, write accepted moves back as calendar updates. The broker becomes an optimisation layer over your existing estate rather than a parallel system.
- **Approval workflow per owner** rather than admin-applies-all — model genuine consent, with auto-apply for moves below a "disruption threshold" (e.g. same room, ≤30 min shift).
- **No-show auto-release** feeding real utilisation back in — that's where the biggest *actual* capacity sits (industry figures put unused-but-booked rooms very high).
- **Multi-day horizon** — the slot model already generalises; widen the day and add a day index to let the broker pull a fragmenting meeting to tomorrow to clear a full-day slot today.

One caveat on the metrics: the "admin time saved" figure rests on explicit per-booking assumptions (in `metrics.py`) — tune those to your reality before quoting them, as they're the softest number in the set. The capacity figures are hard (derived from the actual schedule).

Want me to wire up the Microsoft Graph read/write path next, or add the multi-day horizon to the optimiser?

---

# rooms ask part 2

In addition, if not already included - i need a way to modify the room names and resources.

I want you to create a new repo e.g. mmorrow24work/meetingroom-defrag and push the artifacts to it.

I want a sales-pitch.md and power-point doc created. 

I also want markdown docs and MS Word docs - a user guide for new users and a second for super users include screenshots of the frontend web UI.

I also need a QUICKSTART.md that walks through hosting the solution.

All code created should be saved to a code folder with accompanying markdown docs for each python script ( i assume we are using python ) doing a code review walk through for future code maintenance.

---