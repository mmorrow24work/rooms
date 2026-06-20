# Room Broker — optimising meeting-room allocation (PoC)

A constraint-based **broker** that allocates 15 meeting rooms by resources,
duration and priority, and — crucially — **re-optimises an existing schedule**:
it re-rooms and re-times bookings to free up whole rooms (so full-day bookings
stop failing) and right-sizes meetings into the smallest adequate room. It then
**quantifies the gain** (rooms freed, seat-hours reclaimed, full-day-bookable
rooms, admin time saved).

Runs as two Docker containers under WSL2. No external DB, no cloud dependency.

```
┌──────────────┐   /api (proxied)   ┌─────────────────────────────┐
│  frontend     │ ─────────────────▶ │  backend (FastAPI)           │
│  nginx + SPA  │ ◀───────────────── │  • auth (token + override)   │
│  :8080        │      JSON          │  • greedy allocator (1 req)  │
└──────────────┘                    │  • CP-SAT broker (re-opt)    │
                                     │  • metrics engine            │
                                     │  • scenario simulator  :8000 │
                                     └─────────────────────────────┘
```

## 1. Run it (WSL2)

```bash
cd roombroker
# optional: set a real secret
echo "BROKER_SECRET=$(openssl rand -hex 16)" > .env
docker compose up --build
```

* App:        http://localhost:8080
* API + docs: http://localhost:8000/docs

**Demo sign-ins** (password `demo`): `mick` / `paul` are admins and hold the
override code `OVERRIDE-7` (queue-jump + force-cancel + apply proposals);
`asha` / `tom` are standard users.

## 2. The demo loop

1. Sign in. The board opens on a deliberately **messy, un-optimised day**
   (rooms scattered with short bookings — the real-world pain).
2. **Request a room** (right panel): set seats, duration, priority, resources,
   time window. The greedy allocator right-sizes you into the smallest free room.
3. **Run broker → propose changes.** The solver returns a proposal: a list of
   moves plus before/after metrics. Freed rooms glow amber on the board.
4. **Apply changes** (admins only) to commit the moves.
5. **Load a fresh un-optimised day** with a different size/seed to repeat.

Standard users can request a proposal but cannot apply it — mirroring the
"users must agree to the changes" requirement (in production each affected owner
approves their own move; here an admin applies the agreed set in one step).

## 3. Batch evaluation (no UI)

Quantify the uplift across many scenarios:

```bash
cd backend
pip install -r requirements.txt
python -m app.evaluate --n 26 --seeds 5
```

Typical output (15-room estate, 26 meetings/day, averaged over 5 seeds):

```
rooms freed          8.0
waste reduction      37.6 pts
capacity reclaimed   1.61 avg-room-days
```

i.e. the broker consistently frees the equivalent of **~8 whole rooms** and
cuts wasted seat-capacity by **~38 percentage points** versus how the same
meetings were booked by hand.

## 4. How the broker works (the interesting bit)

The day is discretised into 30-minute slots (08:00–18:00 → 20 slots). The
re-optimisation is a **CP-SAT model** (`app/optimizer.py`):

* **Decision vars** — `x[meeting, room, start] ∈ {0,1}`, created only for
  feasible combinations (room big enough, has the required whiteboard / VC /
  60" monitor / format, start inside the meeting's window).
* **Constraints** — each meeting placed once; no two meetings overlap in a room;
  time-locked meetings (e.g. fixed client calls) keep their start but may still
  be re-roomed; pinned meetings don't move at all.
* **Objective** (weighted) —
  1. minimise the number of rooms that have *any* booking → frees whole rooms;
  2. minimise wasted seat-slots `(room.seats − meeting.seats) × duration` →
     right-sizing;
  3. a disruption penalty for moving a booking, **scaled by priority** so a
     client meeting (P1) is far less likely to be shuffled than a focus block (P5).
* **Warm start** — the incumbent (hand-booked) schedule is fed in as a hint, so
  the broker can only improve on it and proves quality fast.

`app/metrics.py` turns a before/after pair into the headline numbers, including
the "equivalent of N average-room-days" figure and a tunable admin-time model.

## 5. Priority model

| Code | Meaning | Behaviour |
|---|---|---|
| 1 | Client meeting | most protected from moves; often time-locked |
| 2 | Recruitment interview | protected |
| 3 | Team meeting | freely movable |
| 4 | One-to-one | freely movable |
| 5 | Solo / quiet time | most movable; bumped first under override |

Override-code holders can **queue-jump** (bump a strictly lower-priority booking
when nothing is free) and **force-cancel** another user's booking.

## 6. Off-the-shelf comparison

See `MARKET-SCAN.md` for vendors, pricing and the key finding: mainstream tools
do *suggestions, auto-release and manual drag-and-drop rescheduling*, but none
do active, constraint-based **re-optimisation of the whole schedule** — which is
exactly the gap this PoC fills.

## 7. Productionising (notes)

* Swap the in-memory store for Postgres; persist proposals + an approval table so
  each owner consents to their own move.
* Replace token auth with Entra ID / OIDC; map the override code to an AD group.
* Two-way sync with Microsoft Graph (Places / room mailboxes) so the broker reads
  real availability and writes back accepted moves as calendar updates.
* Multi-day horizon: the slot model already generalises; widen `SLOTS_PER_DAY`
  and add a day index to the vars.
* Add no-show auto-release (check-in webhook) to feed real utilisation back in.
```
