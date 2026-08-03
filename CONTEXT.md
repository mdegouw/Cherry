# Cherry

Cherry is Aartsen's B2B webshop: existing trade customers place pickup orders against branch-scoped stock and intraday-volatile prices held in the Thinkwise ERP. Cherry mirrors that truth and never masters it.

This glossary grows as decisions land. The catalogue and content terms are settled ([ADR-0001](./docs/adr/0001-cherry-owned-product-content.md)), as is fulfilment ([ADR-0002](./docs/adr/0002-pickup-slot-and-cut-off-rule.md)), Cherry's own domain — organisation, cart and order ([ADR-0003](./docs/adr/0003-cherry-domain-model.md)) — the staff principal ([ADR-0004](./docs/adr/0004-staff-principal-and-audit.md)), and how the catalogue is searched and filtered ([ADR-0005](./docs/adr/0005-catalogue-search.md)).

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
The ERP's own volumetric grouping of articles. Cherry uses it to set availability-band thresholds and to seed an article's default category; it is not a shop category.
_Avoid_: Category, product group

**Assortment**:
The set of sellable articles. Company-wide and identical for every customer — there is no customer-specific assortment.
_Avoid_: Catalogue (which is the branch-suppressed, customer-facing view of the assortment)

**Facet**:
A mirrored ERP article attribute a customer can filter on — origin, class, calibre, packaging, brand. Facets are ERP truth, never authored in Cherry.
_Avoid_: Attribute, property, tag, variant axis

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
The presentation Cherry owns for an article: image, marketing copy, category and preparation/storage advice. Nothing a customer buys on is content.
_Avoid_: Product data, product info, enrichment, PIM data

**Content group**:
An optional Cherry-owned grouping of articles that share an image and generic copy, used only so an article without its own content can inherit some. It carries no commercial meaning and never affects whether an article can be ordered.
_Avoid_: Product, product family, variant group

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
Cherry's own state for getting an order into the ERP: `submitted` → `accepted` | `rejected`. Terminal within minutes. Distinct from any ERP fulfilment status, which Cherry mirrors as an opaque code and never reasons about.
_Avoid_: Order status, state, fulfilment status

**Order line**:
A snapshot complete enough to render and re-price itself without the article existing — description as shown, quantity and order unit, nominal weight, the accepted price and its price unit, and the estimated total. Records the accepted shortfall when there was one. Does not record the availability band, which is display redaction rather than a term of the sale.
_Avoid_: Line item, order item, cart line (which is the pre-submit thing)

**Shortfall**:
The gap between what a customer asked for and what is actually available, revealed as a real number on cart-add and at submit when the article is at `limited` or below. Accepting one is a commercial fact and is stored on the order line.
_Avoid_: Backorder, partial, stock-out, tekort (in code)

**Estimated total**:
A line or order total computed from nominal weight, and therefore never the invoiced amount — actual weight resolves at picking. Stored as accepted rather than recomputed on display.
_Avoid_: Total, price, order value

### Integration

**Mirror**:
Cherry's local copy of ERP truth — articles, prices, stock, debtors — kept fresh by polling, since the ERP cannot push. Everything in the mirror is **advisory**: it exists so pages render without touching the ERP. The single atomic order call is where a commitment is actually made, and the ERP's answer there outranks anything the mirror said.
_Avoid_: Cache (which implies a read-through that Cherry never does), sync, replica
