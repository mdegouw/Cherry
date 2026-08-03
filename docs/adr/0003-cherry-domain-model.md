# The mirror is for speed, the ERP is for truth

---

Status: accepted

---

Cherry masters very little: organisations, users, one cart, and orders. This ADR fixes that model — what each entity holds, what a cart and an order line snapshot, how an order reaches the ERP and what happens when it cannot, and the vocabulary the ERP API contract inherits.

Decided in [#10](https://github.com/mdegouw/Cherry/issues/10), building on the identity model from [#6](https://github.com/mdegouw/Cherry/issues/6), the content seam from [ADR-0001](./0001-cherry-owned-product-content.md), the pickup slot from [ADR-0002](./0002-pickup-slot-and-cut-off-rule.md), and the Indicium constraints from [#2](https://github.com/mdegouw/Cherry/issues/2).

## The governing principle

Every mirrored fact in Cherry — price, availability band, stock, blocked status, on-account permission — is **advisory**. It exists so a page can render in milliseconds without touching the ERP.

Exactly one moment in Cherry makes a commitment: the **atomic order call**. That call is the ERP's to accept or reject, and its answer outranks anything Cherry believed.

This resolves the apparent conflict between the two constraints from [#2](https://github.com/mdegouw/Cherry/issues/2) — _"make order submission one call"_ (11) and _"Cherry mirrors; it never blocks on the ERP"_ (8). They only contradict each other if the mirror is treated as authoritative and the ERP as a formality. Inverted, each mechanism below has one obvious answer.

## The model

```
organisation                          user
  debtor_number      unique             organisation_id
  name                                  name, email
  home_branch_code   ← mirror           invited_at, activated
  blocked            ← mirror         staff  ← separate principal,
  on_account_allowed ← mirror                  no organisation (#6)
  activated_at       ← Cherry (#6)
  deactivated_at     ← debtor gone, never deleted
  mirrored_at

cart                 1 per organisation, durable, never expires
  organisation_id    unique
  └─ cart_line       (cart_id, article_code) UNIQUE
       quantity              ← SET, never incremented
       touched_by_user_id
       updated_at

order
  reference          C-26-4F27  ← the only human-facing id; idempotency key
  organisation_id, placed_by_user_id
  branch_code, pickup_date, pickup_slot_start   ← ADR-0002
  payment_method     on_account | at_pickup
  handover_state     submitted → accepted | rejected
  erp_reference      null until accepted        ← staff-only
  erp_message_id     on rejection
  erp_status_code    null | passthrough
  estimated_total, submitted_at
  └─ order_line
       article_code, description_snapshot
       quantity, order_unit, nominal_weight_kg
       unit_price, price_unit, estimated_total
       requested_quantity, shortfall_accepted_at   ← null unless shortfall
```

## The organisation holds no address and no VAT number

Cherry is pickup-only and the ERP invoices. Neither field has a reader, and a mirrored field with no reader is a field that goes silently wrong.

A debtor that vanishes from the ERP is **deactivated, never deleted** — users hang order history off it, and ADR-0001's principle that content may degrade but ordering never does applies equally to history.

## The cart is shared, and looks it

One durable cart per **organisation**, created on activation, emptied on submit, never expiring. A restaurant's chef adds through the afternoon and the manager submits in the evening — the real produce-trade workflow, which a per-user cart cannot express.

Three rules make sharing safe without locking:

**One line per article code.** `(cart_id, article_code)` is unique.

**Quantity is set, never incremented.** The manager who adds tomatoes to a cart already holding ten sees the ten and changes it to twenty deliberately, or leaves it. Incrementing makes "did we order 10 or 20?" unanswerable.

**Every line shows who touched it last, and when.** _"gewijzigd door Piet, 2 min geleden."_ Concurrent edits resolve last-write-wins, and the cost of that — a change you did not make — is paid by making it visible rather than by preventing it.

Optimistic locking was rejected: it converts a mostly-harmless overwrite into an error state every user must understand, and it would fire far more often on innocuous edits than on genuine conflicts. Real-time sync was rejected as a websocket stack for an object edited on the scale of minutes; a refresh on navigation plus Inertia polling on the cart page is sufficient.

### The cart stores no price

A cart line holds `article_code` and `quantity`. Price, band, description and facets are **always rendered live from the mirror**, so a cart is structurally incapable of displaying a stale price.

A stored price-at-add was rejected for two reasons. It is a second price concept with no commercial standing, and anything displayed as a price will be read as a quote. And in a shared cart it is *one user's* price shown to *another* — the chef's afternoon €12.40 presented to the manager as though it were theirs.

### The submit-time delta baseline travels with the request

Binding price is the live price at confirmation, so the customer must accept any delta — but a delta needs a baseline, and the baseline is _the price they were looking at when they decided_.

The submit request carries the prices the browser was showing. Cherry re-fetches, diffs against those, and hands any delta to [#18](https://github.com/mdegouw/Cherry/issues/18)'s gesture.

The delta therefore always means **"since you looked at this page"** — a definition that is identical whether one user or five are in the cart, and which needs no stored state. Writing a `price_last_shown` onto the line at render time was rejected precisely because it fails under sharing: the baseline would belong to whoever loaded the cart last, so the manager could be shown a delta measured from the chef's view.

## Submission: Cherry-first, then the ERP

```
POST /orders
  → INSERT order (handover_state: submitted, reference: C-26-4F27)
  → clear cart
  → attempt atomic ERP call, idempotency_key = reference
       accepted → handover_state: accepted, erp_reference stored
       rejected → handover_state: rejected, erp_message_id stored,
                  cart restored, customer told to phone the branch
       timeout  → handover_state: submitted, queued retry on the same key
  → confirmation page in every case
```

Cherry writes the order before calling the ERP. The order therefore cannot be lost to an ERP outage, a restart, or the multi-second cold start Thinkwise documents — and because Cherry mints the reference first, that reference is available as the idempotency key.

**The ambiguous timeout is the case that drives this.** When the connection dies mid-call, Cherry cannot know whether the ERP committed. Retrying risks a double order; not retrying risks losing it. Neither is acceptable, so the contract must make the call **idempotent on Cherry's reference** — and only a Cherry-first sequence has a reference to be idempotent on. This is a hard requirement on [#11](https://github.com/mdegouw/Cherry/issues/11), not a preference.

ERP-first submission was rejected because it makes the shop unable to take orders whenever the ERP is slow, and leaves the timeout with no safe resolution. Always-asynchronous submission was rejected because it can never show an ERP reference even when the ERP is healthy, and pushes every rejection past the point where the customer has been told yes.

### Handover state is Cherry's; fulfilment state is a passthrough

Two things get called "order status" and only one is Cherry's.

**`handover_state`** — `submitted → accepted | rejected`. About getting the order *into* the ERP. Cherry owns it, branches on it, and it reaches a terminal value within minutes.

> **Amended by [ADR-0006](./0006-order-reconciliation.md).** A fourth terminal value, `abandoned`, exists — staff stopping an unbounded retry — and `submitted` has **no upper time bound**, so "terminal within minutes" is the common case rather than a guarantee.

**`erp_status_code`** — a nullable mirrored code, polled for non-terminal orders, rendered through a Cherry-owned lookup to a Dutch label and **simply not shown when the code is unrecognised**. No Cherry logic ever reads this field.

Modelling a full `accepted → picked → ready → collected` lifecycle was rejected: Cherry would own a workflow it cannot observe or validate, every state it names becomes a state the Thinkwise team must commit to producing, and every ERP process change becomes a Cherry mapping bug. Under the passthrough, an unknown or newly-added ERP state degrades to silence.

Mirroring nothing at all was also rejected — it leaves no hook for "is it ready?" without a later contract change, and the poll is cheap at the order volumes in question.

## Order identity: one number, said out loud

`order.reference` — short, quotable, e.g. `C-26-4F27` — is the **only** identity a customer ever learns. It is the idempotency key, the email subject, the history label, and the thing said on the phone. `erp_reference` is stored but shown only to staff.

This follows from the map routing every exception through the branch: blocked debtor, order change, cancellation, no-show. The number on the customer's confirmation must be findable by a branch employee working in the ERP — so **the contract requires the ERP to persist Cherry's reference on the order and index it for search.**

Making the ERP number customer-facing was rejected on the timeout case: it does not exist yet, so a real order would exist that the customer has no way to name. Showing both was rejected because customers read out whichever number is nearer the top.

## The order line renders itself

An order line carries everything needed to display and re-price it **without the article existing**:

| Field | Why |
| --- | --- |
| `article_code` | ADR-0001: immutable, never reissued — no surrogate id needed anywhere |
| `description_snapshot` | ADR-0001 obligation: the code may vanish; history must still render |
| `quantity` + `order_unit` | what was bought, and in what — `4 kist` |
| `nominal_weight_kg` | the order-unit → price-unit conversion |
| `unit_price` + `price_unit` | the binding price as accepted — `€1.85 / kg` |
| `estimated_total` | stored, not recomputed: `4 × 5.000 × 1.85 = 37.00` |
| `requested_quantity`, `shortfall_accepted_at` | null unless a shortfall was accepted |

**Totals are estimates.** Actual weight resolves at picking, so `estimated_total` is what the customer accepted, never what they are invoiced.

Reading the unit metadata live from the mirror is *nearly* safe — ADR-0001 means a change of packaging mints a new article code, so `nominal_weight_kg` is immutable per code. What defeats it is the other half of ADR-0001: the code can disappear from the ERP entirely. So the line snapshots regardless.

**The accepted shortfall is a stored commercial fact.** When a customer asks for 40 and is told _"slechts 12 beschikbaar"_ and accepts, they knowingly took less. That is precisely what gets disputed at the counter, and it is also the only source of data on how often shortfalls bite.

**The availability band is not stored.** Per [#5](https://github.com/mdegouw/Cherry/issues/5) it is redaction for display, not a term of the sale, and promoting it to a stored field would make it look like one.

## Blocked debtors: the mirror stops early, the ERP decides

```
submit
  mirror says blocked  → stop, "neem contact op met je vestiging"
  otherwise            → atomic ERP call
                           reject(BLOCKED) → handover_state: rejected
                                             cart restored, same message
```

Cherry stops the submit on its mirrored flag when it already knows — which covers most blocked customers with no ERP round trip. When the flag is stale, the ERP rejects and Cherry records `handover_state: rejected` with the ERP's `MessageID`.

[#6](https://github.com/mdegouw/Cherry/issues/6) called blocked status the one mirrored fact whose poll lag has a commercial cost. Cherry-first submission dissolves that: the ERP was always going to be the gate, so Cherry's flag is a fast path rather than the enforcement, and **Cherry can never be more permissive than the ERP.**

A live debtor re-check before submit was rejected as a second synchronous ERP dependency on the highest-stakes path, needing a fail-open-or-closed policy of its own — for an outcome the atomic call already produces. Per [#6](https://github.com/mdegouw/Cherry/issues/6), no reason is ever shown; the customer phones the branch.

## Repeat ordering against a moved catalogue

Because a change of origin mints a new article code, a seasonal switch is one code vanishing and a different one appearing. **A partially-unreorderable history line is the normal case, not an edge case** — in January, a July order is largely dead codes.

`Bestel opnieuw` is available per order and per line. It adds every still-orderable line at its original quantity and reports the rest **split by whether tomorrow will differ**:

| Reason | Reported as | Because |
| --- | --- | --- |
| Band is `sold out` today | _vandaag uitverkocht_ | worth retrying tomorrow |
| Code gone from the ERP | _niet meer leverbaar_ | never returning under this code |
| Flagged unsellable | _niet meer leverbaar_ | as above |
| Hidden at this branch ([#4](https://github.com/mdegouw/Cherry/issues/4)) | _niet meer leverbaar_ | as above |

Collapsing these into one list was rejected: sold-out-today and gone-forever demand opposite responses, so an undifferentiated list makes the customer wait for something that will never come or re-hunt something that would be back in the morning.

**Where the cart already holds the article, the existing quantity is kept and the line flagged** — _"stond al in de mand, aantal ongewijzigd."_ Repeat ordering must not silently overwrite a number a colleague set; that is the one thing the shared-cart rules exist to prevent.

Substitution — suggesting the equivalent code from another origin — needs an equivalence relation between article codes that nothing in Cherry has. It remains open on the map.

## Vocabulary

Full definitions in [`CONTEXT.md`](../../CONTEXT.md). The decisions behind the contested ones:

**`Organisation`, not `customer`.** Cherry has both a buying company and a person at a keyboard; "the customer accepted the shortfall" cannot distinguish them. `Debtor` stays reserved for the ERP record being mirrored. The glossary governs code and spec vocabulary — **the Dutch UI says _klant_ freely.**

**`Order unit` and `price unit`, paired.** The pairing is the point: `kist` and `kg` are both units and confusing them is the central arithmetic error available in this domain.

**`Colli` is a count, not a unit type.** In Dutch produce trade, _"4 colli"_ is a quantity of packages. It is defined in the glossary as trade shorthand and explicitly **not** a Cherry field name — the model says `quantity`, because an article sold per stuk or loose by weight is not colli in any natural reading, and a field named for the majority case is misnamed for the rest.

## Consequences

- **[#11](https://github.com/mdegouw/Cherry/issues/11) gains four contract requirements**: the order call idempotent on Cherry's reference; the ERP persisting and indexing that reference; rejections carrying a keyed `MessageID` rather than translated text; and a status poll for non-terminal orders.
- **[#18](https://github.com/mdegouw/Cherry/issues/18) gains its baseline**: the submit request carries the prices the browser was showing.
- **Cherry can hold orders that do not exist in the ERP** — `submitted` ones stuck in retry and `rejected` ones. No surface exists for them; specified in [#19](https://github.com/mdegouw/Cherry/issues/19).
- **The confirmation email cannot assume ERP acceptance**, since it may be sent while `handover_state` is still `submitted`. Specified in [#20](https://github.com/mdegouw/Cherry/issues/20).
- `payment_method` is **resolved from** `on_account_allowed`, not chosen by the customer. There is no PSP, so the only alternative is paying at pickup.
- The `erp_status_code` → Dutch label lookup is staff-maintained config, joining the settings surface that [ADR-0002](./0002-pickup-slot-and-cut-off-rule.md) and [#5](https://github.com/mdegouw/Cherry/issues/5) also need.
- Nothing here requires a live ERP read on any render path, preserving constraint 8 from [#2](https://github.com/mdegouw/Cherry/issues/2) intact.
