# Cherry

Cherry is Aartsen's B2B webshop: existing trade customers place pickup orders against branch-scoped stock and intraday-volatile prices held in the Thinkwise ERP. Cherry mirrors that truth and never masters it.

This glossary grows as decisions land. The catalogue and content terms below are settled ([ADR-0001](./docs/adr/0001-cherry-owned-product-content.md)); ordering, organisation and cart vocabulary is still open in [#10](https://github.com/mdegouw/Cherry/issues/10).

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

### People

**Staff**:
An Aartsen employee principal, with no organisation and no debtor number. Owns branch hours, band thresholds and — as the marketing department — product content.
_Avoid_: Admin, user (which means a customer's login), employee
