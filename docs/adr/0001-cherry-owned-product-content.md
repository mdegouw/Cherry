# Cherry-owned product content is article-keyed presentation only

---

Status: accepted

---

The ERP owns article truth; Cherry owns presentation. This ADR fixes exactly where that seam falls, what Cherry stores, and how content survives an article's lifecycle.

Decided in [#7](https://github.com/mdegouw/Cherry/issues/7).

> **Partially superseded by [ADR-0007](./0007-product-content-back-office.md).** Two provisions below have changed: a **content group is derived**, not a grouping staff assemble (see "Content group"), and **copy and storage advice are group-level by default**, with the article-keyed version as the override (see "The seam rule" and "Content group"). Everything else here stands.

## The seam rule

**Can this fact change without the article code changing?** If yes, it is volatile ERP truth and Cherry must never store it as content. If no, Cherry may own it.

At Aartsen the answer is _no_ for every commercially meaningful attribute: **origin, class, calibre, packaging and brand each mint a new article code**. They are mirrored ERP facts — displayed and filterable, never typed by staff. The ERP encodes them in the article description _and_ exposes them as separate fields, so Cherry mirrors the fields and renders the description verbatim.

**Cherry owns exactly four things:** image, marketing copy, category, and preparation/storage advice.

The rule matters most for origin. Were it a Cherry-typed field it would go stale into a false claim about a physical good — and origin declaration carries EU marketing-standard weight for most fruit and vegetables.

## Join key

Content binds to the **ERP article code**, which is immutable and never reissued. No surrogate ERP id is stored; no drift detection is needed. Immutability is load-bearing — it is what lets a returning seasonal article reattach its content automatically.

## Shape: flat, no product entity

One catalogue tile per article code. Cherry has **no `product` entity**, so the word _product_ has no referent in the domain — `article` is the unit of everything (catalogue, cart, order line, content).

This is a deliberate deviation from the webshop-default variant model. Because origin/class/calibre fan out into separate codes, a `product` grouping N codes was the obvious alternative and was rejected: it makes a hand-maintained merchandising mapping **load-bearing for ordering** (an unmapped code becomes unbuyable), and it forces availability bands to aggregate across variants, reopening [#5](https://github.com/mdegouw/Cherry/issues/5). The flat list also matches what these buyers already navigate — the paper price list and the ERP's own order-entry screen.

Facets over the mirrored ERP fields do the collapsing a variant model would have done. A `product` entity remains addable later as purely additive presentation, because content stays article-keyed underneath.

## Content group

A Cherry-owned, optional grouping used **only** to inherit an image and generic copy. Resolution order for an article's image: its own → its content group's → a neutral placeholder.

A content group is never load-bearing for ordering. An article in no group is fully buyable; it just shows a placeholder. This collapses launch photography from ~1000 shoots to ~200 without requiring the grouping to be complete.

## Categories

A Cherry-owned tree: **one primary category per article, at most two levels deep.** No multi-homing in MVP — the facets already span categories, and multi-homing multiplies breadcrumbs and canonical-URL questions for merchandising polish.

Defaults come from an **ERP article group → category mapping** (a few dozen rows, not a thousand), overridable per article. New ERP articles therefore land somewhere sane with no staff action. Note that ERP article groups are _volumetric_ — they exist to set availability-band thresholds ([#5](https://github.com/mdegouw/Cherry/issues/5)) — so they seed the tree but are not the tree.

"Uncategorised" is a staff worklist state, never customer-visible: such an article is still fully orderable through search and facets, it simply does not appear in browse.

## Images

No product photography exists today. Aartsen's marketing department shoots, uploads and assigns images in Cherry — which implies a content-editing principal, passed to [#13](https://github.com/mdegouw/Cherry/issues/13).

`spatie/laravel-medialibrary` handles storage and conversions (approved as a new dependency). Cherry stores an original and derives its own variants in a queued job — WebP with a JPEG fallback. Cherry never resizes on request and never hotlinks a supplier URL.

**v1 ships with mostly placeholders.** Photography is continuous backfill, not a launch gate: these buyers order by code and description, not by browsing photographs.

## Content i18n is blocked upstream, not by this schema

Content columns hold **plain Dutch text**. No per-locale columns, no translations table — including category names, which stay staff-editable rows so marketing can rename without a deploy.

This is recorded explicitly so it is not later mistaken for an oversight: the majority of text on an article page is **ERP-owned and Dutch-only** (description, origin, class). A translated Cherry copy layer would produce English prose above a Dutch article description. Content i18n is gated on the ERP becoming multilingual, not on Cherry's schema. Adding a locale later is a mechanical migration that arrives bundled with admin-UI work anyway.

Cherry's own UI chrome remains fully key-driven and i18n-ready, unchanged.

## Lifecycle: content is never cascade-deleted

Three disappearances, only one of which is a lifecycle event:

1. **`sellable` flag off, or hidden after a month out of stock at a branch ([#4](https://github.com/mdegouw/Cherry/issues/4))** — the code still exists. Visibility change; content untouched.
2. **The code is deleted from the ERP** — content becomes orphaned.
3. **The code is absent from one sync pass** — not a lifecycle event.

Case 3 is the trap worth guarding. Indicium offers no deltas and enforces 1000-record paging ([#2](https://github.com/mdegouw/Cherry/issues/2)), so Cherry rebuilds the catalogue by walking pages. A sync that dies mid-walk looks exactly like a bulk deletion; naive orphaning would silently unpublish hundreds of articles.

Therefore:

- Orphaning is derived **only from a sync pass that completed in full**, and only after the code has been **absent for 7 consecutive days**. A partial pass marks nothing. This makes "full pass completed" a first-class signal the sync must expose — passed to [#9](https://github.com/mdegouw/Cherry/issues/9).
- Orphaning **marks a staff worklist state; it never deletes.** A human purges. Seasonal produce codes leave and return, and deleting the Dutch-tomato image in January means re-shooting it in July.
- A direct URL to a hidden or orphaned article serves a **"no longer available" page, not a 404** — order history links to it, and a 404 on something the customer demonstrably bought reads as a bug.

Order lines must therefore snapshot the article description so history renders without the article existing — passed to [#10](https://github.com/mdegouw/Cherry/issues/10).

## The governing principle

**Content and merchandising are allowed to degrade; ordering never does.** Missing image, missing copy, missing category, missing content group — each degrades presentation only. No content state can make a sellable article unbuyable.
