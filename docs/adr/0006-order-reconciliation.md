# A stuck order is an outage, not a queue

---

Status: accepted

---

[ADR-0003](./0003-cherry-domain-model.md) made submission Cherry-first, which means Cherry can hold orders the ERP does not have: `submitted` ones whose retry has not landed, and `rejected` ones the ERP refused. This ADR fixes who sees them, what they can do, what the customer is told, and how the retry ever stops.

Decided in [#19](https://github.com/mdegouw/Cherry/issues/19), building on the submission sequence from [ADR-0003](./0003-cherry-domain-model.md), the staff principal and audit log from [ADR-0004](./0004-staff-principal-and-audit.md), and the ERP performance envelope from [#12](https://github.com/mdegouw/Cherry/issues/12).

## The reframe: a cohort, not a trickle

[#12](https://github.com/mdegouw/Cherry/issues/12) measured the ERP: **250 ms guaranteed, >1000 req/s**. An order call that times out is therefore almost never one unlucky order. It means the integration is **down or unreachable**.

So stuck orders do not arrive as a thin stream that a worklist grinds through one at a time. They arrive as a **cohort** — every order submitted during the outage window, all stuck for one reason, all resolving the moment that reason is fixed. Every decision below follows from that shape: the alert is about the outage rather than the order, no bulk action is needed because recovery is automatic, and on a normal day the surface holds nothing at all.

## Retry is unbounded, and "stuck" is a query

[ADR-0003](./0003-cherry-domain-model.md) left a timeout at `handover_state: submitted` with "a queued retry on the same key" and never said whether that retry gives up. It does not.

**Retry runs indefinitely with escalating backoff.** "Stuck" is therefore not a state but a **query**: `handover_state = submitted` and `submitted_at` older than a threshold — **15 minutes**, long enough that a cold start or a blip resolves itself unobserved, short enough that a branch still has the working day to act.

A bounded retry with a `failed` state was rejected twice over:

- **The ceiling fails exactly where it is needed.** For a ninety-second blip, no human is required anyway. In a four-hour outage the ceiling expires while the ERP is still dead, and a person must then re-fire every order in the cohort by hand. The policy breaks in precisely the scenario that justifies it.
- **`failed` asserts what Cherry cannot know.** The idempotency key makes retrying safe forever, so a state meaning "this will never reach the ERP" is a claim about the future of a system Cherry does not control. [ADR-0003](./0003-cherry-domain-model.md) refused the fulfilment lifecycle on the same grounds: do not name a state you cannot observe.

The cost is real and belongs on the record: **`submitted` stops meaning "terminal within minutes"** and becomes an interval with no upper bound. Every surface that reads `handover_state` must handle an old `submitted`. That is what this ADR is for.

## The alert is the load-bearing part

[#16](https://github.com/mdegouw/Cherry/issues/16) wrote the principle: _a worklist nobody opens is the same as no worklist._ For uncategorised articles that is tolerable, because [ADR-0001](./0001-cherry-owned-product-content.md) permits content to degrade. A stuck order is the inverse case — a customer is planning their morning around a slot for an order the branch cannot see — and the failure is silent by construction. Pull-only means it is discovered at the counter.

So Cherry **pushes**, and pushes about the outage:

| When | Mail |
| --- | --- |
| First order crosses 15 minutes | _"de ERP-koppeling faalt: 7 orders wachten, oudste 22 minuten"_ + link |
| Hourly while it persists | the same, with current counts |
| Queue drains | one resolution mail |

**To one shared staff address**, not to every staff account: a fan-out to ten people means ten people each assuming someone else is on it.

Per-order mail was rejected — forty emails describing one fact trains staff to filter the alert, and it misdiagnoses the work. Nobody needs to chase `C-26-4F27` individually; somebody needs to phone Thinkwise, after which the whole cohort clears itself.

**Rejections raise no alert.** A rejection is one customer, already told at submit to phone the branch — the customer *is* the notification channel. Alerting on both would put the routine case in the same inbox as the emergency.

## One page, two sections

`/beheer/orders` — _"Orders die de ERP niet heeft"_ — with **Vastgelopen** above **Afgewezen**. One entry in [ADR-0004](./0004-staff-principal-and-audit.md)'s `/beheer` nav, beside the configuration surfaces. There is no second back-office.

One undifferentiated list was rejected on a precedent this repo has already set: [ADR-0003](./0003-cherry-domain-model.md) splits repeat-order unavailability into _vandaag uitverkocht_ and _niet meer leverbaar_ because the two "demand opposite responses". Stuck and rejected are that asymmetry a level up — one needs the **integration chased**, the other needs a **conversation with a customer**, and only one of them resolves itself.

Two separate surfaces were rejected because on a normal day both queues hold zero rows. Two routes, two nav entries and two empty states is structure bought against volume that does not exist.

The section order encodes the urgency: the population where **the customer does not know** sits above the population where they do.

**Afgewezen may later migrate** to a general order-lookup surface — a rejection is what a customer phones about, and order lookup remains open on the map. **Vastgelopen never migrates**: it is an operational queue, not a customer-support view.

## What staff can do

### Retry, per order

_"Nu opnieuw proberen"_ enqueues an attempt immediately instead of waiting out the backoff. Safe by construction, audited under [ADR-0004](./0004-staff-principal-and-audit.md), disabled while an attempt is in flight. It is the only lever staff have and it is the one they will want the moment Thinkwise says they are back.

**No bulk retry.** Backoff drains the cohort automatically once the ERP returns, so "retry all 40" exists only to make a person feel busy — and it is the one action here that could hammer a half-recovered ERP.

### The idempotent call is the status query

[#19](https://github.com/mdegouw/Cherry/issues/19) expected to need a new ERP endpoint: query an order by Cherry reference to resolve the ambiguous timeout. **It does not.** If the order call is genuinely idempotent on `order.reference`, re-calling it *is* that query — either the ERP has no such order and now creates it, or it already has one and returns it. Cherry learns the truth either way. A separate read endpoint would be a second thing for the Thinkwise team to build to answer a question the first thing already answers, and the map warns explicitly against over-asking.

What this does require is sharpening the existing requirement into something that could otherwise be built useless: **a duplicate call must return the existing order's ERP reference, distinguishable as a duplicate rather than an error.** An implementation returning a bare `409` is idempotent in the "no double order" sense and worthless in the "resolve the ambiguity" sense. [ADR-0003](./0003-cherry-domain-model.md) asked for idempotency and never said the duplicate response must be informative.

### Abandon — the price of unbounded retry

`handover_state` gains a fourth terminal value: **`abandoned`**.

Because nothing ever stops trying, there must be a way to stop it. The case is already fixed by the map: self-service cancellation is out of scope, so cancellations arrive **by phone**. A customer whose order is stuck phones the branch to cancel it — and without this, Cherry injects that order into the ERP three hours later.

This does not contradict the refusal of `failed` above. `failed` was rejected because Cherry cannot know an order will never reach the ERP. `abandoned` is a **staff decision** — a fact Cherry owns outright, and the only fact that legitimately stops the queue. Model what can actually be asserted.

Three properties, each of which reads as surprising until stated:

- **It restores no cart.** A rejection restores the cart because that happens *at submit*, with the customer still on the page. An abandonment happens hours later, by which time the organisation has likely built a new cart — and [ADR-0003](./0003-cherry-domain-model.md)'s shared-cart rules exist precisely to stop one action overwriting a number a colleague set.
- **It requires a note**, shown on the row. _"Klant heeft telefonisch geannuleerd."_ The audit log holds who and when; the question a colleague asks is *why is this dead*, and audit-only means digging through `/beheer/logboek` for an answer that belongs on the row.
- **It does nothing to the ERP.** If the ambiguous timeout concealed a successful commit, the ERP holds an order Cherry now calls abandoned, and the staff member must cancel it ERP-side themselves. The button reads as more powerful than it is. Retrying first is the honest way to find out which world you are in.

## What the customer sees

The customer distinguishes **"we have your order"** from **"we don't"** — never *"we have it but the ERP doesn't"*.

| `handover_state` | Order history |
| --- | --- |
| `accepted` | bevestigd, with slot |
| `submitted` | **identical** — bevestigd, with slot |
| `rejected` | _niet verwerkt — neem contact op met je vestiging_ |
| `abandoned` | same treatment |

`submitted` stays invisible because the distinction is **unactionable**. The customer cannot make the integration work, so surfacing it converts a problem that usually self-heals within minutes into a phone call to a branch that also cannot see the order. It leaks Cherry's plumbing into a commercial surface, and the honest wording — _"we may or may not have your order"_ — serves the customer worse than either truth.

`rejected` and `abandoned` **must** be visible: an order presented as confirmed that nobody will pick is the most damaging thing this surface could permit. The **reason** is still withheld, per [#6](https://github.com/mdegouw/Cherry/issues/6) — Cherry says it did not go through and points at the branch.

**The alert buys the silence.** Withholding `submitted` from the customer is only defensible because staff are pushed at. Without the alert nobody would know, and hiding it from the customer as well would mean the failure surfaces at the counter. **If the alert is ever dropped, this decision reopens.**

The escalation channel for a stuck order approaching its slot is a **phone call from staff**, not a Cherry notification: the map keeps notifications beyond order confirmation out of scope, and every other exception on this map already routes through the branch.

## Rejection codes: no silence for staff

[ADR-0003](./0003-cherry-domain-model.md) set the customer-side precedent — `erp_status_code` renders through a Cherry-owned lookup and is **simply not shown** when unrecognised. The staff surface inverts it.

**The raw `MessageID` is always on the row. The Dutch label is a gloss on top of it.**

The two audiences want opposite failure modes. To a customer, `ERR_DEBT_BLK_02` is noise, so silence is kinder. To staff it is the **only actionable thing on the screen** — it is what they quote to Thinkwise, and it is what makes two rejections recognisable as the same problem. Degrading to silence would empty the surface exactly when something new has started happening, which is the one case a human is genuinely needed for.

So an unknown code renders as itself; a known code renders as _Debiteur geblokkeerd_ with the code still beside it. Where the ERP's response carries its own message text, Cherry **stores it and shows it verbatim** — no translation, no interpretation. That is a nullable field rather than a new contract requirement: [#11](https://github.com/mdegouw/Cherry/issues/11) deliberately asked for keys rather than translated text, and this does not reopen it.

The lookup joins the **same staff settings surface** as [ADR-0003](./0003-cherry-domain-model.md)'s `erp_status_code` labels — one screen, two code namespaces, separate tables — and it **ships holding only the codes already known to exist**, exactly as [ADR-0005](./0005-catalogue-search.md) shipped article-group aliases empty:

- **Debtor blocked** — [#12](https://github.com/mdegouw/Cherry/issues/12) lists this among the four ERP fields Thinkwise must build.
- **Insufficient stock** — and this one deserves attention. [#12](https://github.com/mdegouw/Cherry/issues/12) established that the ERP **refuses an over-order atomically**, so a stock race becomes a rejection. That is very likely the **dominant** rejection in production, and it is neither a chasing case nor a conversation case: it is the shortfall gesture arriving too late. [#12](https://github.com/mdegouw/Cherry/issues/12) named the cheapest fix — a live re-check at the commitment moment, now affordable — and that decision belongs to [#18](https://github.com/mdegouw/Cherry/issues/18). **If it does not land, Afgewezen becomes a busy queue for a fixable problem** rather than the rare-exception list this ADR assumes.

## The row

| | Vastgelopen | Afgewezen |
| --- | --- | --- |
| Identity | `reference` — the number the customer will say aloud | same |
| Customer | organisation name + **debtor number** | same |
| Placed by | user name + email | same |
| Pickup | date, slot start, **time until the slot** | same |
| Value | line count + estimated total | same |
| Why | attempts, last attempt, **next** attempt, last error | code + Dutch gloss + raw ERP text |
| Age | `submitted_at` + elapsed | `submitted_at` |
| Actions | _Nu opnieuw proberen_ · _Afbreken_ (note required) | none |

The **debtor number** is load-bearing rather than decorative. Cherry holds no phone number — [ADR-0003](./0003-cherry-domain-model.md) dropped address and VAT, [#6](https://github.com/mdegouw/Cherry/issues/6) left the ERP's contact persons unused — so the debtor number is how a staff member gets from this row to a phone call. It looks like a field with no reader; this is its reader.

**Vastgelopen sorts by time until the pickup slot, ascending — not by age.** A three-hour-old order for tomorrow afternoon is *less* urgent than a twenty-minute-old order for a slot at 11:00 this morning. Sorting by staleness puts the loudest row on top instead of the one about to hurt.

### How a row leaves

| | |
| --- | --- |
| Stuck → `accepted` | the retry succeeds; the row vanishes with no human action. **The common case, and why the list is normally empty.** |
| Stuck → `abandoned` | staff, with a note |
| `rejected` | nothing resolves it, so it **ages off after 7 days** — the section is *recent rejections*, not an unbounded log. Older ones live in order history. |
| `abandoned` | leaves both sections at once; note and causer survive on the order and in `/beheer/logboek` |

**No _afgehandeld_ tick and no assignment.** Two staff could in principle both phone the same customer, but this repo has now twice chosen visibility over prevention — the shared cart's _gewijzigd door Piet_ ([ADR-0003](./0003-cherry-domain-model.md)) and audit instead of roles ([ADR-0004](./0004-staff-principal-and-audit.md)) — the population is rare, and an acknowledgement flag is state that goes unmaintained and then lies. **Revisit if rejections become routine**, which is exactly what the unresolved re-check in [#18](https://github.com/mdegouw/Cherry/issues/18) would cause.

## Consequences

- **`handover_state` is four-valued**: `submitted → accepted | rejected | abandoned`. [ADR-0003](./0003-cherry-domain-model.md)'s three-valued definition and `CONTEXT.md` are amended.
- **`submitted` has no upper time bound.** Any surface reading `handover_state` must handle an old one; nothing may treat `submitted` as transient.
- **The order gains `abandoned_at`, `abandoned_by_staff_id` and `abandoned_reason`**, plus `erp_message_text` nullable alongside `erp_message_id`.
- **[#11](https://github.com/mdegouw/Cherry/issues/11) gains one requirement and loses one**: the duplicate order-call response must return the existing ERP reference distinguishably; and **no query-by-reference endpoint is needed**, which is one less thing to ask Thinkwise to build.
- **[#18](https://github.com/mdegouw/Cherry/issues/18) carries the live stock re-check.** Whether it lands decides if _Afgewezen_ is a rare-exception list or a daily queue.
- **A second mailer exists**, on a shared staff address, independent of [#20](https://github.com/mdegouw/Cherry/issues/20)'s customer confirmation. That address is configuration, and it is the one setting whose absence silently disables the control this ADR leans on.
- **The rejection-code lookup joins the settings surface** that [ADR-0002](./0002-pickup-slot-and-cut-off-rule.md), [#5](https://github.com/mdegouw/Cherry/issues/5) and [ADR-0003](./0003-cherry-domain-model.md) already need — now four staff-editable lookups sharing one screen.
- **The retry scheduler is a queued job with unbounded attempts**, which is not Laravel's default `tries` behaviour and must be written deliberately.
- **Nothing here adds a customer-facing notification channel**, so the map's notification fog is untouched.
