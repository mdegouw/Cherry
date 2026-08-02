# Pickup availability is an offerable-slot filter, not a cut-off that assigns a day

---

Status: accepted

---

Cherry is pickup-only. This ADR fixes how Cherry decides which pickup moments a customer may choose, what a chosen slot commits either side to, and what happens when the answer changes mid-checkout.

Decided in [#8](https://github.com/mdegouw/Cherry/issues/8), building on the branch-hours model from [#3](https://github.com/mdegouw/Cherry/issues/3).

## The reframe

The standing decision read: _same-day pickup if ordered at least one hour before that branch's closing time, otherwise the next business day._ That phrasing makes the cut-off a rule that **assigns a pickup day**, which cannot be reconciled with also letting a customer order for a later day — both cannot be primary.

Cherry instead computes the set of **offerable slots** and lets the customer choose from it. Same-day-versus-next-day is then an emergent consequence — _does today still have an offerable slot?_ — rather than a stored rule. Holidays, closures, odd exception hours and the mid-checkout straddle all fall out of the one filter instead of each needing a special case.

## Configuration

All Cherry-owned and staff-editable, extending [#3](https://github.com/mdegouw/Cherry/issues/3)'s model:

| Setting | Scope |
| --- | --- |
| Weekly pickup-hours pattern | per branch, per weekday: open+close, or closed |
| Dated branch exception | per branch, per date: closed, or different hours |
| Company-wide closure | per date, all branches |
| `picking_lead_time` | per branch, one company default (30 min) |

**Pickup hours are not the branch's trading hours.** They are Cherry-owned config describing when Cherry pickups may be collected. A branch that trades on Saturday but does not want Cherry pickups configures Saturday closed — no second flag is needed.

### The company-closure layer refines #3

[#3](https://github.com/mdegouw/Cherry/issues/3) modelled exceptions as per-branch only, one mechanism covering both holidays and one-offs. That elegance costs a re-entry of every national holiday for every branch every year, and a forgotten row puts a customer at a locked gate. A company layer is added, resolved by precedence:

1. **Branch dated exception** — wins outright, in either direction
2. **Company-wide closure** — closed
3. **Branch weekly pattern** — open or closed

Christmas is entered once; a branch trading that day overrides at level 1.

## The rule

**Open day** — a date whose resolution yields an open window.

**Slot grid** — from that date's **opening time**, 30-minute steps, while `slotStart + 30min <= close`. Anchoring to opening rather than to wall-clock `:00`/`:30` wastes no time at a branch opening at `06:15` and makes odd-houred exception days work without special handling. Where opening falls on a half hour — the normal case — the two are indistinguishable.

**Offerable** — a slot is offerable at instant `now` iff

```
openTimeElapsed(branch, now -> slotStart) >= picking_lead_time
```

where `openTimeElapsed` counts only minutes falling inside open windows.

**Horizon** — every offerable slot on the **first two open days containing at least one**. The scan is bounded to 14 days ahead; finding fewer than two qualifying days offers what exists, and finding none disables ordering for that branch and alerts staff.

**Clock** — `Europe/Amsterdam`, a Cherry-wide constant, not a per-branch field. Hours and slots are stored as wall-clock (date + local time); the absolute instant is always derived, never stored as truth.

## One knob, and why it is a lead time

The only configured number is how long the branch needs to pick an order. "One hour before closing" is not configured anywhere — it is a **consequence**:

```
last slot of the day starts at   close - 30min
offerable iff  now + leadTime <= close - 30min
with leadTime = 30min:  now <= close - 60min
```

which is exactly the agreed rule. Raising the lead time to 60 moves the effective cut-off to 90 minutes before close, honestly reflecting that picking got slower. A second, independently-configured cut-off was rejected: it can be set into contradiction with the lead time, and staff would have to hold both in their heads.

### Measured in open time, not wall-clock time

The lead time accrues **only while the branch is open**. Wall-clock arithmetic over-promises: an order placed at 23:00 for a 06:00 opening has seven hours of nominal lead time, none of it staffed, and Cherry would commit the branch to an order picked before anyone arrived.

In open time, that order's clock starts at 06:00 and its earliest slot is 06:30. The first slot of a day therefore goes to orders that arrived while the branch was previously open — precisely those that could have been picked for.

## The horizon is a price constraint, not a UX preference

Binding price is the live price at confirmation, and prices move intraday. A slot three days out locks today's price onto produce picked three days from now, with Aartsen carrying the move — on a commodity where that move is the margin. The stock check has the same problem: the availability band and shortfall are a snapshot of _now_, and a future day's stock does not exist to check against.

Two open days is the distance a confirmed price and a stock check can honestly be projected.

The horizon is anchored on **offerable open days, not calendar days**. Calendar anchoring punishes late orderers — at 23:00 on Tuesday, today has no slots left, so the customer would see Wednesday alone with no way to plan Thursday. Open-day anchoring yields a single explicable invariant: **you always see exactly two open days, whenever you look.** The worst case is a Friday-evening order reaching Tuesday.

## DST is made impossible rather than handled

DST transitions occur at 02:00/03:00 local on a Sunday. No Aartsen branch is open then, so an ambiguous or non-existent slot time cannot be constructed.

Rather than rely on that, a **validation rule refuses to save an exception day whose open window overlaps a DST transition**, and the test suite asserts it. Resolution of a wall-clock slot to an instant is then always unambiguous, with no ambiguity-preference or shift-forward rules to write, test or get wrong.

A per-branch IANA timezone was rejected as a column holding the same value in every row, which every date calculation must nonetheless remember to load.

## What a slot commits

The slot is **Aartsen's ready-from commitment**: the order will be picked and waiting from the slot's start time.

It places **no obligation on the customer**. Arriving later the same day is fine. The 30-minute window is grid granularity for the customer's own planning; the slot's end matters only as the guarantee that at least 30 minutes of opening remain in which to collect.

Slot capacity is out of scope, so nothing competes for a slot — which is also why a soft hold during checkout would be meaningless. Nothing is being reserved; the only thing expiring is the picking lead time, and time cannot be held.

## Re-validation at submit

The chosen slot is re-validated at submit against the same filter. If it no longer qualifies, **the order does not go through**: the customer sees the current slot list and re-picks.

This uses the same accept-the-change gesture Cherry already has for a binding-price delta and for a shortfall reveal — one mechanic, three causes. Silently rolling the order forward to the next valid slot was rejected as exactly the bait-and-switch to avoid: a customer who planned around 15:30 should not discover on the receipt that they are collecting tomorrow.

The slot is chosen **on the confirm step**, alongside the price and shortfall acceptance, so the straddle window is seconds rather than minutes.

The three-cause gesture itself is specified in [#18](https://github.com/mdegouw/Cherry/issues/18).

## "Business day" is retired

The term has no referent in Cherry. There is no business-day calendar object — a date is open iff that branch's resolved pickup hours say so, which makes "does Saturday count?" a per-branch configuration question rather than a rule. `CONTEXT.md` uses **open day**.

## Worked examples

Branch `06:00–16:00` Mon–Fri, closed Sat/Sun, `picking_lead_time` 30 min, unless stated.

| Placed | Offered |
| --- | --- |
| Mon 14:00 | Mon 14:30→15:30, Tue 06:00→15:30 |
| Mon 15:00 | Mon 15:30, then Tue — the last same-day moment, exactly 1h before close |
| Mon 15:01 | Tue, Wed — 15:31 rounds up to 16:00, which does not fit before close |
| Fri 15:30 | Mon, Tue |
| Tue 23:00 | Wed 06:30→15:30, Thu 06:00→15:30 |
| Wed 06:05 | Wed 07:00→15:30, Thu — 06:35 rounds past the 06:30 slot |
| Wed 24 Dec 09:00, company closures on 25 + 26 | Wed 09:30→15:30, Mon 29 |
| Fri 15:30, branch exception opens Sat 06:00–11:00 | Sat 06:00→10:30, Mon |
| Thu 10:01, branch exception 06:00–11:00 that day | Fri, Mon — 10:31 rounds to 11:00, which does not fit |
| Branch with `picking_lead_time` 60, Mon 14:31 | Tue, Wed — effective cut-off 90 min before close |
| Slot list drawn Mon 14:55, submit Mon 15:05 | 15:30 is stale → re-pick required |

These are written to be usable directly as Pest cases.

## Consequences

- The order carries a pickup slot as **branch code + date + local start time**, and must snapshot it — passed to [#10](https://github.com/mdegouw/Cherry/issues/10).
- `openTimeElapsed` is the one non-trivial calculation and deserves its own unit tests independent of the slot filter.
- Branch hours, company closures and lead times need a staff back-office surface, alongside the band thresholds from [#5](https://github.com/mdegouw/Cherry/issues/5) and the content tooling from [#15](https://github.com/mdegouw/Cherry/issues/15).
- Real per-branch hours remain **seed data, not specification** — everything here is testable without a single real opening time.
