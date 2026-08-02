# Cherry

Cherry is Aartsen's B2B webshop: existing trade customers place pickup orders against branch-scoped stock and intraday-volatile prices held in the Thinkwise ERP. Cherry mirrors that truth and never masters it.

This glossary grows as decisions land. The catalogue and content terms are settled ([ADR-0001](./docs/adr/0001-cherry-owned-product-content.md)), as is fulfilment ([ADR-0002](./docs/adr/0002-pickup-slot-and-cut-off-rule.md)); ordering, organisation and cart vocabulary is still open in [#10](https://github.com/mdegouw/Cherry/issues/10).

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

**Staff**:
An Aartsen employee principal, with no organisation and no debtor number. Owns pickup hours, company closures, picking lead times, band thresholds and — as the marketing department — product content.
_Avoid_: Admin, user (which means a customer's login), employee
