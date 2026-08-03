# Mirror what you query; fetch what you display

---

Status: accepted

---

[ADR-0003](./0003-cherry-domain-model.md) established that the mirror is advisory and the atomic order call is the only commitment. It did not say how the mirror is filled. This ADR fixes that: what Cherry polls, what it fetches on demand, what it never holds at all, and what happens at the one moment the ERP gets to disagree.

Decided in [#9](https://github.com/mdegouw/Cherry/issues/9), building on the Indicium capabilities from [#2](https://github.com/mdegouw/Cherry/issues/2), the installation facts from [#12](https://github.com/mdegouw/Cherry/issues/12), the band model from [#5](https://github.com/mdegouw/Cherry/issues/5), the domain model from [ADR-0003](./0003-cherry-domain-model.md), the catalogue query from [ADR-0005](./0005-catalogue-search.md) and the reconciliation surface from [ADR-0006](./0006-order-reconciliation.md).

## The two principles

**Mirror what you query; fetch what you merely display.** [ADR-0005](./0005-catalogue-search.md) made the availability band a predicate of the catalogue query — it filters and orders every customer-facing list — so stock has to be in Cherry's own database or the query cannot run. Nothing in Cherry sorts, filters or aggregates by price. Price is only ever rendered next to an article a query already found.

**Cherry never predicts a refusal.** The ERP's refusal *is* the validation. Nothing forecasts it, so nothing can disagree with it.

Almost every decision below is one of these two applied.

## Price is not mirrored, because price is never queried

[#12](https://github.com/mdegouw/Cherry/issues/12) handed this ticket what looked like its hardest problem: the ERP's price lives in a **view** over a base price and a discount group, and a view carries no trace column, so the cheap `modified_since` that tables get for free does not reach the most volatile datum on the map — prices move several times a day per article and customer.

That problem is **dissolved rather than solved**. A change stamp answers "what changed since T?", and Cherry never needs to ask. Price is fetched per organisation, wholesale, on demand:

```
price-bearing page render, organisation X
  rows fresh enough?  → serve
  stale?              → serve, dispatch background refresh
  absent?             → fetch synchronously

fetch = GET price_view?$filter=debtor_number eq <X>
        ~1000 rows, paged, upsert into article_price
```

The volumes make this comfortable. [#4](https://github.com/mdegouw/Cherry/issues/4) sized the assortment at hundreds to ~1000 sellable articles, and the ERP's article master is barely larger. One organisation's entire price list is therefore **one or two OData pages** against a 250 ms guarantee, and the rows Cherry holds scale with *how many organisations are actually shopping* rather than with Aartsen's debtor book. Price is keyed debtor × article, so a background mirror would have grown as debtors × 1000 whether or not those debtors ever logged in.

Three alternatives were rejected:

- **Asking Thinkwise to expose a derived change stamp on the view** — buildable, since both of the view's inputs carry trigger-filled trace columns, so `greatest()` of the two would work. Rejected because it spends the Thinkwise team's time on a mechanism that exists only to answer a question Cherry has stopped asking, and it brings a fan-out to reason about: one discount-group edit changes every debtor in that group across every article.
- **Mirroring price wholesale on a fixed cycle with no change stamp** — needs no Thinkwise work at all, but the arithmetic runs the wrong way. At fifty activated organisations a full pass is fifty pages; at two hundred it saturates. The cadence would degrade as Cherry succeeded.
- **Reading price live on every render with no local copy** — affordable per call, but it makes an ERP outage a total price outage and puts a hard ERP dependency on the most-loaded path in the application.

### The fetched prices live in a table, not a cache

`article_price` — `(debtor_number, article_code)` unique, plus `unit_price`, `price_unit`, `fetched_at`. The [ADR-0005](./0005-catalogue-search.md) catalogue query joins it, so price arrives with the articles rather than being merged onto them in PHP afterwards.

This is the same table a background mirror would have had. The only thing that changed is **who triggers the write**.

A Laravel cache entry per organisation was rejected on three counts. `CACHE_STORE` is `database`, so it would land in MySQL regardless and buy nothing structurally. Every surface that shows a price would hand-write the merge. And a cache miss during an ERP outage leaves *nothing* to fall back on, whereas a stale row is still a row — which is what makes the degraded behaviour below possible at all.

Because it is a table rather than a cache, staleness is a per-row `fetched_at` that [#16](https://github.com/mdegouw/Cherry/issues/16) can inspect, not an opaque TTL. Rows are pruned by `fetched_at`, so a dormant organisation holds none.

### Freshness is tiered by surface, because it is really a question about friction

Mirror staleness adds directly to the submit-time delta window: a confirm page rendering a price already a minute old, submitted thirty seconds later, is a ninety-second window in which the price may have moved. At roughly four moves per article per trading day and a fifteen-line cart, a two-minute window puts a delta on something like one order in seven, and ten minutes puts one on most of them.

So "how fresh?" is not a technical budget. It is **how often a customer is made to accept a price change** — and the answer differs by surface because the cost of freshness does too.

| Surface | Behaviour |
| --- | --- |
| Login | Queued warm-up for the organisation, under a lock so several users signing in together fetch once |
| Browse, search, cart | Serve immediately; dispatch a background refresh if older than ~5 minutes |
| **Confirm page** | **Synchronous refresh, always.** One page load per order, so 250 ms is invisible — and it makes the delta baseline fresh to the millisecond |
| Submit | No read at all; see below |

The delta window collapses to confirm→submit, which is seconds. Nothing pays for freshness on pages where it does not matter.

Because a stale row is *served* while its refresh goes to the queue, **no render ever blocks on the ERP** — and the login warm-up removes the only remaining cold case. The property [#2](https://github.com/mdegouw/Cherry/issues/2)'s constraint 8 asked for survives, which was not obvious when a read-through was first proposed.

A single tight budget everywhere was rejected as paying catalogue-wide ERP traffic for freshness that only the confirm page consumes, while still leaving that page a minute stale. A single loose budget, leaning entirely on [#18](https://github.com/mdegouw/Cherry/issues/18)'s gesture, was rejected because it makes accepting a price rise a routine click — which trains customers through the one screen that is supposed to carry real commercial weight.

This ADR delivers the **unit price and its price unit**. Whether those can be composed into a total is [#21](https://github.com/mdegouw/Cherry/issues/21)'s question, and nothing here depends on its answer.

## Articles and stock are full-walked, not delta-polled

~1000 articles and three branches means stock is ~3000 rows and the article master ~1000. Against the 1000-record response cap that is **four to six pages, roughly a second and a half, for a complete re-read of everything**.

At those volumes a delta mechanism optimises nothing worth having, and full-walking pays for itself twice over:

- **Deletion detection is free.** A code absent from a completed pass is absent. This closes [#2](https://github.com/mdegouw/Cherry/issues/2)'s constraint 6 — how de-listings are communicated — which [#12](https://github.com/mdegouw/Cherry/issues/12) recorded as *the one constraint still open*.
- **[ADR-0005](./0005-catalogue-search.md)'s `search_text` rebuild is free**, exactly as [#14](https://github.com/mdegouw/Cherry/issues/14) asked: idempotent upserts write nothing when nothing changed, so re-reading unchanged data costs reads and no writes.

Two passes, on the cadences the data earns:

| Pass | Every | Reads | Writes |
| --- | --- | --- | --- |
| **Stock** | minute | ~3000 rows, 3–4 pages | upsert quantity, recompute band |
| **Articles** | hour | ~1000 rows, 1–2 pages | upsert facets and units, rebuild `search_text`, diff key set |
| **Debtors** | hour | hundreds of rows | upsert home branch, blocked, on-account |

Stock earns the minute because the band is a query predicate and stock moves continuously. Article descriptions and facets change rarely, and [#14](https://github.com/mdegouw/Cherry/issues/14) explicitly asked for `search_text` to travel with the slow data. Debtor lag is harmless because [ADR-0003](./0003-cherry-domain-model.md) already made the ERP the gate on blocked status.

Passes never overlap; a run still in flight suppresses the next tick.

**There is no `modified_since` anywhere in Cherry.** Which incidentally deletes a problem [#12](https://github.com/mdegouw/Cherry/issues/12) went to some trouble to characterise: trace columns are in local time, so a watermark inside the autumn fold is ambiguous. With no watermark there is no fold to reason about — the same move [ADR-0002](./0002-pickup-slot-and-cut-off-rule.md) made in rendering DST impossible rather than handling it.

A `modified_since` delta was rejected despite the trace columns [#12](https://github.com/mdegouw/Cherry/issues/12) confirmed exist. It is genuinely cheaper per poll, and it buys back both of the problems the full walk gives away: deletes need a separate mechanism, and a pass that dies mid-walk becomes indistinguishable from a mass deletion — precisely the trap [ADR-0001](./0001-cherry-owned-product-content.md) flagged.

### Absence is a key-set diff, evaluated only on a completed pass

[ADR-0001](./0001-cherry-owned-product-content.md) requires "full pass completed" as a first-class signal, because with mandatory paging **a sync that dies on page 3 of 4 looks exactly like a quarter of the catalogue being deleted**.

The walk holds the codes it has seen in memory — a thousand strings, nothing — and *only after the last page succeeds* diffs that set against the mirror, setting `absent_since` on what is missing and clearing it on what returned. A pass that fails never reaches the diff, so "a partial pass marks nothing" is **structural rather than a rule someone has to remember**.

Stamping a `last_seen_at` on every row during the walk was rejected: it writes a thousand rows per pass whether or not anything changed, and it leaves the "only on completion" guarantee as a conditional a future edit can break.

`absent_since` stops orderability **immediately** — an article the ERP no longer has cannot be sold. [ADR-0001](./0001-cherry-owned-product-content.md)'s seven days govern only whether its **content** is orphaned.

A `sync_pass` row records every attempt — kind, `started_at`, `completed_at`, status, pages read, rows seen. That row is the data [#16](https://github.com/mdegouw/Cherry/issues/16) surfaces; whether staff are shown it, and who watches, remains that ticket's.

## No push

[#12](https://github.com/mdegouw/Cherry/issues/12) reopened this deliberately — nothing bypasses Indicium so process flows fire on every write, the platform is 2026.2 so message brokers exist, and an API and mail queue are already in place — and asked that it be weighed rather than dismissed.

Weighed, it loses on its own terms. [#2](https://github.com/mdegouw/Cherry/issues/2) measured realistic push latency at **tens of seconds** for the only shape available: trigger → outbox → scheduled system flow → HTTP POST. That is not better than a one-minute stock poll. And price — the datum push would most want to accelerate — is no longer mirrored at all, so **there is nothing left for it to push**.

It would add a custom Thinkwise build, at-least-once delivery and undocumented retry semantics, in exchange for no latency win. Recorded as decided-against so it stays shut. [#2](https://github.com/mdegouw/Cherry/issues/2)'s own strongest evidence still stands: Thinkwise's Universal GUI polls, because no real event channel exists.

## The commitment moment: the refusal is the validation

[#12](https://github.com/mdegouw/Cherry/issues/12) established two facts that together make this the sharp end of the map. The ERP **refuses** an over-order, and the order call is atomic. [ADR-0003](./0003-cherry-domain-model.md) writes the order in Cherry first, so whenever the mirror is stale by a single crate, Cherry holds an order the ERP has just refused — and with stock moving continuously that is routine.

Cherry submits anyway, and uses the refusal.

```
submit
  → INSERT order (submitted, reference C-26-4F27)
  → atomic ERP call, idempotency_key = reference
       accepted → accepted, erp_reference stored
       refused  → recoverable: #18's gesture, amended basket
                  resubmits under a NEW reference
       rejected → terminal: cart restored, phone the branch
       timeout  → submitted, retry SAME reference   (ADR-0006)
```

No pre-flight. No dry run. No basket-validation endpoint.

Two alternatives were rejected, and both were rejected for the same reason:

- **A `validate_only` dry-run mode on the order call**, returning per-line price, available quantity and a verdict. Attractive because validation and commitment would share a code path and could not disagree. Rejected as an extra round trip and an extra contract negotiation to *predict* an answer the next call gives for free.
- **Composing a pre-flight from the price and availability read endpoints.** Adds nothing to the contract, which is its appeal, but the quantity a read endpoint reports is not guaranteed to be the quantity the order call validates against — so it can clear a basket the ERP then refuses, and pinning that equivalence in prose is weaker than not needing it.

The principle underneath: **a mechanism that predicts a refusal can be wrong about it. A mechanism that reads the refusal cannot.**

What this does require is that the refusal be **usable**. [#2](https://github.com/mdegouw/Cherry/issues/2)'s constraint 13 forbids parsing `TranslatedMessage`, so _"slechts 12 beschikbaar"_ cannot be scraped out of prose. The refusal must carry **structured per-line detail** — article code, message code, current available quantity and current price. This is a real addition to [#11](https://github.com/mdegouw/Cherry/issues/11), and it is still a smaller ask than a dry-run mode. Without it, [#18](https://github.com/mdegouw/Cherry/issues/18) has no numbers to render and this decision reopens.

It also promotes one of [#12](https://github.com/mdegouw/Cherry/issues/12)'s residuals to load-bearing: **whether the ERP refuses when a submitted price diverges from its own**. If it silently overwrites instead, the binding-price promise has no enforcement point anywhere in the system.

### `refused` and `rejected` are separated by whether a customer is present

[ADR-0006](./0006-order-reconciliation.md) closed while this was being decided, and named the risk precisely: insufficient stock is *"very likely the dominant rejection in production"*, and if the live re-check does not land, _Afgewezen_ *"becomes a busy queue for a fixable problem"*. The re-check did not land. The state split is what protects that surface instead.

**Recoverability is not a property of the refusal code. It is a property of whether a customer is there to answer.**

| | `refused` | `rejected` |
| --- | --- | --- |
| When | The submit was synchronous — the customer is on the page | The refusal came back on a queued retry, after an outage |
| Cause | Over-order, price divergence | Blocked debtor, unrecognised code, *or* an over-order with nobody present |
| Customer sees | [#18](https://github.com/mdegouw/Cherry/issues/18)'s gesture, then their amended order | _niet verwerkt — neem contact op met je vestiging_ |
| In history | No — this is not a failed order | Yes, per [ADR-0006](./0006-order-reconciliation.md) |
| Back-office | **None.** Nothing to action | _Afgewezen_ on `/beheer/orders` |
| Resubmit | Amended basket, **new reference** | Nothing; the branch takes over |

A new reference on resubmission is safe precisely because a refusal is **unambiguous**: the ERP definitively did not commit, so there is nothing for the old reference to collide with. That is the opposite of the timeout case, where ambiguity is exactly why [ADR-0006](./0006-order-reconciliation.md) retries the *same* reference forever.

`refused` gets **no worklist**, and that is the point of separating it. The customer has already resolved it and submitted again; a queue of rows nobody needs to action is what [#16](https://github.com/mdegouw/Cherry/issues/16) warned against. It is kept because it is the only evidence of how often shortfalls actually bite — the same reason [ADR-0003](./0003-cherry-domain-model.md) stores the accepted shortfall — and that is a reporting question, not an operational one.

Discarding the row entirely was rejected for destroying that evidence. Leaving both populations in one `rejected` state was rejected because a customer who asked for one crate too many would get a permanent failed order in their history and a message telling them to phone the branch, which is the outcome this whole question exists to prevent.

## Degraded mode: the gate is an event, not a duration

Browsing degrades gracefully — the catalogue serves from the mirror with a staleness banner, and the shop stays useful through an ERP outage. Submitting is different, because the design above **depends on a customer being present when the ERP answers**.

When Indicium is unreachable, Cherry queues the submission only if **this order's price was ERP-verified**:

```
confirm page  synchronous price refresh
                OK   → price verified ✓ → submit may queue if the ERP is down
                FAIL → renders from the mirror with a banner,
                       submit disabled: "prijzen konden niet worden gecontroleerd"
```

The gate is a boolean derived from something that actually happened, not a threshold anyone has to calibrate. A customer who loaded the confirm page successfully and then hit an outage on submit has accepted a price the ERP confirmed seconds ago, so queuing is safe. A customer arriving *during* the outage is stopped at confirm — the earlier, honest place to stop them — rather than at the button.

A staff-tunable staleness ceiling in minutes was rejected: nobody can derive the right value, so it would be set once by guess and never revisited, and its real-world behaviour would drift silently as price volatility changed. Blocking submission outright during any outage was rejected as closing the shop for blips the event gate handles correctly.

Price, not stock, is what bounds this. A queued order carries the price the customer accepted, and prices move several times a day, so a queued submission is inherently a **blip absorber measured in minutes** rather than a way to trade through a long outage. The event gate encodes that without naming a number.

An order queued this way that comes back refused has nobody to show a gesture to, so it lands as `rejected` and joins [ADR-0006](./0006-order-reconciliation.md)'s _Afgewezen_ with a customer to phone. That surface's resolution mail should say how many of the drained cohort need calling.

## Accepted costs

Stated plainly, because both are visible to customers and neither is a bug.

**A shortfall reveal can be optimistic.** Cart-add reads the stock mirror, up to a minute old, so it can say _"nog 12 beschikbaar"_ when the truth is 8, and the customer is corrected a second time at submit. Accepted because [#5](https://github.com/mdegouw/Cherry/issues/5) defined the band as **redaction, not decision support** — the reveal is guidance and the refusal is the truth — and because cart-add happens fifteen times per order where submit happens once. Reading availability live at confirm was considered and rejected as the pre-flight returning by the back door: it would predict the refusal without sharing its code path.

**An outage-queued order can be refused with nobody present.** Bounded to one confirm-page-load's worth of orders per outage, and it resolves through a phone call rather than silently.

## Consequences

- **`handover_state` is five-valued**: `submitted → accepted | refused | rejected | abandoned`. [ADR-0006](./0006-order-reconciliation.md)'s four-valued definition and `CONTEXT.md` are amended. `refused` is terminal and never retried.
- **[ADR-0006](./0006-order-reconciliation.md)'s open risk is answered without the live re-check it expected.** _Afgewezen_ stays a rare-exception list because over-orders resolve as `refused`, not because refusals were made rare. Its two-section page, its alerting and its no-acknowledgement stance all stand unchanged.
- **[#11](https://github.com/mdegouw/Cherry/issues/11) loses three asks**: trace columns and `modified_since` (constraint 5), the de-listing signal (constraint 6 — the last one [#12](https://github.com/mdegouw/Cherry/issues/12) left open), and any basket-validation endpoint.
- **[#11](https://github.com/mdegouw/Cherry/issues/11) gains three**: the refusal must carry structured per-line detail; the price view must be efficiently filterable by `debtor_number` and **paged via `@odata.nextLink`**, since 1000 articles sits exactly on the response cap and Cherry must never depend on an exemption being configured; and the stock row should carry the **branch-level sellability flag**, since [#12](https://github.com/mdegouw/Cherry/issues/12)'s stock history retired the last-in-stock bookkeeping [#4](https://github.com/mdegouw/Cherry/issues/4) had obliged Cherry to keep.
- **[#12](https://github.com/mdegouw/Cherry/issues/12)'s residual 1 is now load-bearing.** Whether the ERP refuses on price divergence must be answered before [#11](https://github.com/mdegouw/Cherry/issues/11) is finished; the binding-price promise has no enforcement point without it.
- **[#18](https://github.com/mdegouw/Cherry/issues/18) draws its numbers from the refusal payload** rather than from a validation call, and gains a fourth cause with no gesture available: the outage-queued order refused in absentia.
- **The availability band is stored on the stock row**, recomputed on every stock pass and on a staff threshold edit, because [ADR-0005](./0005-catalogue-search.md) orders by it.
- **Three scheduled passes and one queued fetch** — stock every minute, articles and debtors hourly, none overlapping; plus the per-organisation price fetch, locked so concurrent logins fetch once.
- **`article_price` is pruned by `fetched_at`.** It is the one mirror table whose size tracks usage rather than the assortment.
- **Nothing in Cherry holds a price change stamp, a watermark or a delta cursor**, so [#12](https://github.com/mdegouw/Cherry/issues/12)'s local-time DST fold has no reader.
- **The map's degraded-mode fog is largely consumed**: the architecture is decided here. What remains is customer-facing copy — the staleness banner and the disabled-submit wording — which belongs with the legal and disclaimer copy still open on the map.
