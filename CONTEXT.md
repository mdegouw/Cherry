# Cherry

Cherry is Aartsen's B2B webshop: existing trade customers place pickup orders against branch-scoped stock and intraday-volatile prices held in the Thinkwise ERP. Cherry mirrors that truth and never masters it.

This glossary grows as decisions land. The catalogue and content terms are settled ([ADR-0001](./docs/adr/0001-cherry-owned-product-content.md)), as is fulfilment ([ADR-0002](./docs/adr/0002-pickup-slot-and-cut-off-rule.md)), Cherry's own domain — organisation, cart and order ([ADR-0003](./docs/adr/0003-cherry-domain-model.md)) — the staff principal ([ADR-0004](./docs/adr/0004-staff-principal-and-audit.md)), how the catalogue is searched and filtered ([ADR-0005](./docs/adr/0005-catalogue-search.md)), what happens to an order the ERP never took ([ADR-0006](./docs/adr/0006-order-reconciliation.md)), how marketing produces content ([ADR-0007](./docs/adr/0007-product-content-back-office.md), which supersedes part of ADR-0001), and how price and availability reach Cherry at all ([ADR-0008](./docs/adr/0008-price-and-availability-freshness.md)).

Terms here govern code and specification, including the ERP API contract. They do not govern UI copy: the interface is Dutch, so it says _klant_ and _kist_ where the model says `Organisation` and `order unit`.

## Language

### Catalogue

**Article**:
A single sellable thing in the ERP, identified by its article code. Origin, class, calibre and packaging are part of _which_ article something is, not attributes of a looser grouping — so "Dutch Class I tomatoes in a 5kg crate" is one article and the Spanish equivalent is another.
_Avoid_: Product, SKU, item, line item

**Article code**:
The ERP's immutable business key for an article. Never reissued, so it is the only key Cherry stores for an article and the only join between Cherry content and ERP truth.
_Avoid_: Article id, product id, SKU

**Article group**:
The ERP's own volumetric grouping of articles — a flat list of a few dozen. Cherry hangs exactly three staff-editable settings on it, on **one screen**: the availability-band threshold, the default category, and the search aliases. It is not a shop category.
_Avoid_: Category, product group

**Assortment**:
The set of sellable articles. Company-wide and identical for every customer — there is no customer-specific assortment.
_Avoid_: Catalogue (which is the branch-suppressed, customer-facing view of the assortment)

**Facet**:
A mirrored ERP article attribute a customer can filter on — origin, class, calibre, packaging, brand. Facets are ERP truth, never authored in Cherry.
_Avoid_: Attribute, property, tag, variant axis

**Availability band**:
`available` | `limited` | `sold out` — the only availability a customer ever sees, bucketed Cherry-side from the ERP's quantity against one threshold per article group. **Redaction, not decision support**: it exists so a quantity never reaches the browser, which is why it is never a term of the sale and never stored on an order line. Stored on the mirrored stock row, because the catalogue query orders by it.
_Avoid_: Stock level, quantity, availability, status

**Catalogue query**:
The single query that produces every customer-facing list of articles. Free text, category and facets are optional predicates that narrow it **together**, over a base that always applies the branch's suppressions. There is no separate search; typing does not clear a category or a facet.
_Avoid_: Search, search query, filter, product listing

**Search text**:
A stored, normalised column per mirrored article holding everything a free-text query may match: the ERP description, the article code, the canonical facet values, the category name and the article-group alias. Doubled vowels are collapsed in it and in the query, so Dutch singulars reach their plurals. Marketing copy and storage advice are deliberately absent — findability must not depend on how well an article is merchandised.
_Avoid_: Search index, index, keywords, tsvector

**Article-group alias**:
Staff-typed alternative search terms attached to an ERP article group, folded into the search text of every article in it. One row covers a whole origin/class/calibre fan-out. It is a remedy for a reported miss, never a launch requirement, and it carries no commercial meaning.
_Avoid_: Synonym, keyword, tag, search term

### Content

**Product content**:
The presentation Cherry owns for an article: image, marketing copy, category and preparation/storage advice. Nothing a customer buys on is content. Authored on the **content group** by default and inherited; an article may override any of it, but nothing routes staff there.
_Avoid_: Product data, product info, enrichment, PIM data

**Content group**:
The vegetable, as opposed to the article codes it is sold in — a **derived** grouping of every article sharing an article group and the same facet-stripped description. It holds the image, generic copy and storage advice that its articles inherit, so a returning seasonal code arrives already merchandised. Nobody assembles one: staff only **merge** two that should have clustered and didn't. It carries no commercial meaning and never affects whether an article can be ordered.
_Avoid_: Product, product family, variant group, assigning articles to one

**Content residue**:
What remains of an article's description once every facet value is removed — the vegetable's own name, and the content group's key. Derived by removing known structured values, never by parsing prose.
_Avoid_: Base description, product name, normalised description

**Content queue**:
The list of content groups with no image, ordered by how many article codes each covers, descending — so the row at the top is the photograph that retires the most placeholders. Marketing's front door and their whole daily job; it is a production queue, never an operational health signal, and it never routes to a single article. The interface calls it _fotowerklijst_.
_Avoid_: Worklist (which in Cherry means the operational catalogue-health lists), backlog, todo

**Staging tray**:
Uploaded images not yet assigned to a content group, shown as thumbnails so marketing can match hundreds of files to rows **by eye**. Deliberately the only bulk path: a photograph is self-describing where an article code is not, so matching on filenames would silently put a wrong photo on a wrong vegetable.
_Avoid_: Media library, unassigned images, inbox, import

**Category**:
A node in Cherry's own browsing tree. Each article has exactly one primary category, at most two levels deep.
_Avoid_: Article group, taxonomy node, collection, department

**Orphaned content**:
Product content whose article code has vanished from the ERP for at least seven days, confirmed by a fully completed sync pass. Orphaned content is hidden and flagged for staff, never deleted, and reattaches automatically if the code returns.
_Avoid_: Deleted content, stale content, dangling content

### Fulfilment

**Pickup hours**:
Cherry-owned config saying when pickups may be collected at a branch — a weekly per-weekday pattern, overridden by dated exceptions. Deliberately not the branch's trading hours: a branch that trades on Saturday but takes no Cherry pickups is configured closed on Saturday.
_Avoid_: Opening hours, trading hours, business hours

**Open day**:
A date whose resolved pickup hours yield an open window at that branch. Resolution precedence is branch exception, then company closure, then weekly pattern. There is no calendar object behind this and no company-wide notion of it — whether Saturday is open is a per-branch configuration answer.
_Avoid_: Business day, working day, trading day

**Company closure**:
A date on which every branch is closed — national holidays and company-wide shutdowns, entered once. A branch exception overrides it in either direction.
_Avoid_: Holiday, bank holiday, blackout date

**Branch exception**:
A dated override of one branch's pickup hours — closed, or different hours. Outranks both the company closure and the weekly pattern.
_Avoid_: Override, special day, holiday

**Pickup slot**:
A 30-minute window a customer chooses to collect in, stepping from that date's opening time. It is Aartsen's promise that the order is picked and waiting **from** the slot's start; it obliges the customer to nothing, who may collect any time before close. The end only guarantees that 30 minutes of opening remain.
_Avoid_: Time slot, delivery window, appointment, booking

**Picking lead time**:
Per-branch minutes a branch needs to pick an order, accruing **only while that branch is open**. The single knob behind pickup availability: a slot is offerable once this much open time separates now from its start. The familiar "one hour before closing" is a consequence of it, never a configured value.
_Avoid_: Cut-off, cut-off time, lead time, preparation time

### People

**Organisation**:
A trade customer of Aartsen as Cherry knows it — the entity that buys, owns the cart, and owns every order. Mirrors an ERP debtor and is activated for Cherry by staff. Holds no address and no VAT number: Cherry is pickup-only and the ERP invoices, so nothing reads them.
_Avoid_: Customer, account, company, client, debtor (which is the ERP's record, not Cherry's)

**Debtor**:
The ERP's record of a trade customer, and the thing an organisation mirrors. Used only when talking about the ERP side of the seam — inside Cherry the entity is an organisation.
_Avoid_: Using it as a synonym for organisation

**User**:
One person's login, belonging to exactly one organisation. Invited by email by staff. All users of an organisation are equal — there is no organisation-admin, and any user may edit the cart and submit orders on the organisation's behalf.
_Avoid_: Contact person, account, member, customer

**Staff**:
An Aartsen employee principal — its **own table and own auth guard**, never a row in `users`, with no organisation and no debtor number. One role in MVP, all rights across all branches: pickup hours, company closures, picking lead times, band thresholds, ERP status labels, organisation activation, user invitation and — as the marketing department — product content. An Aartsen employee is never a customer, so `staff` and `users` email addresses are disjoint, and that exclusion is enforced on invite. Signs in at `/beheer`; cannot act as a customer, ever.
_Avoid_: Admin, user (which means a customer's login), employee, role (there is only one)

**Audit entry**:
An append-only record of one staff mutation, holding the causer, the moment, and both the old and new value. Every staff-editable thing produces them — commercial config and product content alike. Customer actions produce none: orders are immutable snapshots and a cart line already names who touched it. The interface calls the list _logboek_.
_Avoid_: Change log, revision, history (which in Cherry means a customer's order history)

### Units

**Order unit**:
The ERP-defined unit an article is bought in — kist, doos, zak, stuk. Immutable for a given article code, since changing the packaging mints a new code.
_Avoid_: UOM, packaging unit, colli, crate

**Price unit**:
The unit an article's price is quoted per — kg, stuk. Frequently not the order unit, which is the whole reason both terms exist.
_Avoid_: Unit, billing unit, UOM

**Nominal weight**:
The stated weight of one order unit, converting between order unit and price unit. Nominal because the actual weight is resolved at picking.
_Avoid_: Weight, net weight, actual weight

**Colli**:
The trade's own count of order units — "4 colli". A quantity, not a unit type. Deliberately **not** a Cherry field name: the model says `quantity`, because an article sold per stuk or loose by weight is not colli in any natural reading. Fine in conversation and in Dutch UI copy where it fits.
_Avoid_: Using it to name a unit type, or as a field name

### Ordering

**Cart**:
Exactly one durable, shared basket per organisation — created on activation, emptied on submit, never expiring. Any user may edit it, so the chef can add through the afternoon and the manager submit in the evening. Holds no prices: everything but article code and quantity is rendered live from the mirror.
_Avoid_: Basket, mand (in code), session cart, quote

**Cart line**:
One line per article code in a cart, carrying the quantity, who last touched it and when. Quantity is **set**, never incremented, so a colleague's number is always visible before it is changed. Concurrent edits resolve last-write-wins.
_Avoid_: Cart item, basket line, line item

**Order**:
An organisation's submitted commitment to collect articles in a pickup slot, placed by one user. Created in Cherry **before** the ERP is called, so it survives an ERP outage.
_Avoid_: Purchase, sale, transaction, quote

**Order reference**:
Cherry's own short, quotable identifier for an order — the only one a customer ever learns, and the idempotency key for the ERP call. The ERP is required to store and index it so branch staff can find an order by the number the customer reads out. The ERP's own reference is stored separately and shown only to staff.
_Avoid_: Order number, order id, ERP reference (which is the other one)

**Handover state**:
Cherry's own state for getting an order into the ERP: `submitted` → `accepted` | `refused` | `rejected` | `abandoned`. Usually terminal within minutes, but `submitted` has **no upper bound** — the retry never gives up, so an old `submitted` is a stuck order rather than an impossibility. Distinct from any ERP fulfilment status, which Cherry mirrors as an opaque code and never reasons about.
_Avoid_: Order status, state, fulfilment status

**Refused order**:
An order the ERP declined while the customer was still on the page — an over-order or a price divergence. Terminal and never retried: the customer is shown the real numbers, amends the basket, and submits again under a **new reference**, which is safe because a refusal is unambiguous about not having committed. Absent from order history and from every back-office surface, because there is nothing to action; kept only as evidence of how often shortfalls bite. The same ERP refusal arriving on a queued retry, with nobody present to answer it, is a **rejection** instead.
_Avoid_: Rejected order (which is the terminal, customer-visible one), failed order, declined order

**Stuck order**:
An order still at `submitted` more than fifteen minutes after submission — real in Cherry, absent from the ERP, and unknown as a problem to the customer, who is planning around its slot. Not a state but a **query**, because the retry is still running and may yet succeed. Since the ERP is fast, stuck orders arrive as a **cohort** during an integration outage rather than one at a time. The interface calls one _vastgelopen_.
_Avoid_: Failed order, pending order, error, queued order

**Abandoned order**:
A stuck order that staff have deliberately stopped retrying — the only thing that halts an unbounded retry, and the reason the state exists. Carries a required note, restores no cart, and has no effect in the ERP: if the order did commit there, staff must cancel it ERP-side themselves.
_Avoid_: Cancelled order (Cherry models no cancellation), deleted order, failed order

**Rejection code**:
The keyed `MessageID` the ERP returns when it refuses an order. Rendered to staff **raw and always**, with a Dutch label as a gloss when Cherry's lookup knows the code — never degraded to silence, unlike the customer-facing status code, because to staff it is the only actionable thing on the screen. Never shown to a customer, who is told the order did not go through and to phone the branch.
_Avoid_: Error message, error code, reason

**Order line**:
A snapshot complete enough to render and re-price itself without the article existing — description as shown, quantity and order unit, nominal weight, the accepted price and its price unit, and the estimated total. Records the accepted shortfall when there was one. Does not record the availability band, which is display redaction rather than a term of the sale.
_Avoid_: Line item, order item, cart line (which is the pre-submit thing)

**Shortfall**:
The gap between what a customer asked for and what is actually available, revealed as a real number on cart-add and at submit when the article is at `limited` or below. The two reveals have different authority: on cart-add it comes from the mirror and may be up to a minute optimistic, while at submit it comes from the ERP's own refusal and is the truth. Accepting one is a commercial fact and is stored on the order line.
_Avoid_: Backorder, partial, stock-out, tekort (in code)

**Estimated total**:
A line or order total computed from nominal weight, and therefore never the invoiced amount — actual weight resolves at picking. Stored as accepted rather than recomputed on display.
_Avoid_: Total, price, order value

### Integration

**Mirror**:
Cherry's local copy of ERP truth. Everything in it is **advisory**: it exists so pages render without touching the ERP, and the single atomic order call is where a commitment is actually made — the ERP's answer there outranks anything the mirror said. Filled two different ways, because Cherry mirrors what it queries and fetches what it merely displays: articles, stock and debtors are **polled** on a full walk, while price is **fetched per organisation on demand**.
_Avoid_: Sync, replica, cache (for the polled part; the price half genuinely is read-through)

**Sync pass**:
One complete walk of one class of ERP data — stock every minute, articles and debtors hourly. Always a **full** read, never a delta: the volumes are small enough that this is cheaper than a change stamp, and it makes deletion detection and the search-text rebuild fall out for free. A pass either completes or asserts nothing at all, which is what lets an article's absence be trusted; a walk that dies mid-paging never reaches the point where it could mistake half a catalogue for a mass deletion.
_Avoid_: Sync, import, delta, refresh, job

**Price fetch**:
The on-demand read of one organisation's entire price list from the ERP, keyed on its debtor number. Warmed at login, refreshed in the background while browsing, and **forced synchronously on the confirm page** so the price a customer commits against was ERP-verified seconds earlier. Never a background poll for everybody: price is keyed debtor × article, so the rows Cherry holds track who is actually shopping rather than how many debtors exist.
_Avoid_: Price sync, price mirror, price cache, price feed
