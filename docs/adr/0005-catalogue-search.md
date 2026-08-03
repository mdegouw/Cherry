# Search is a predicate in the catalogue query, not a search engine

---

Status: accepted

---

Cherry has no search index, no search engine and no new dependency. Free text is one optional predicate inside the single query that already serves browse and facets. This ADR fixes that mechanism, the one Dutch rule it needs, what is searchable, and how the list is ordered.

Decided in [#14](https://github.com/mdegouw/Cherry/issues/14), building on the assortment size from [#4](https://github.com/mdegouw/Cherry/issues/4), the band thresholds from [#5](https://github.com/mdegouw/Cherry/issues/5), and the content seam from [ADR-0001](./0001-cherry-owned-product-content.md).

## The constraint that shaped everything

The datastore is **MySQL 8.4**, and no additional long-lived service may be deployed. Both are given to this decision rather than made by it — see _Unowned decision_ below.

That matters because it removed the one option that would have solved Dutch morphology outright: Postgres ships a `dutch` Snowball configuration, so `to_tsvector('dutch', …)` collapses `tomaat`/`tomaten` for free. MySQL has no equivalent.

The important discovery is that **nothing else on the menu stems Dutch either**. Meilisearch and Typesense have no stemmer; they get recall from prefix matching, typo tolerance and hand-authored synonyms. MySQL `FULLTEXT` has no stemmer. So "add a search engine" would not have bought the capability the ticket was worried about, and the no-extra-service constraint costs far less than it appears to.

## Why there is no index and no Scout

Text, category and facets narrow **the same list, together**. A buyer standing in _Tomaten_ with `Herkomst: NL` ticked who types `kl.i` gets NL Class I tomatoes — the text does not clear the category or the facets.

Composition is what rules out a separate index. Once text must combine with relational predicates in one ordering and one pagination, the match has to be a `WHERE` clause. A standalone index returning article codes to intersect afterwards makes ranking and paging across the intersection genuinely nasty.

Laravel Scout's entire job is to abstract an external index. With no external index there is nothing to abstract and no sync to keep honest, so Scout is not used. **No new dependency is introduced by this decision**, which also means no dependency approval is needed under `CLAUDE.md`.

## The mechanism: AND of LIKE

Split the input on whitespace. Every token must appear as a substring of the article's search text:

```sql
WHERE search_text LIKE '%tomat%' AND search_text LIKE '%kl.i%'
```

No index is usable — a leading wildcard forces a scan. At the size [#4](https://github.com/mdegouw/Cherry/issues/4) established (hundreds to ~1000 sellable articles, one row each, short Dutch text) that is a few hundred kilobytes permanently resident in the InnoDB buffer pool. The scan is the cheap part of the request.

MySQL's default `utf8mb4_0900_ai_ci` collation makes matching case- and accent-insensitive with no further work.

### Why not InnoDB FULLTEXT

An ERP description reads `Tomaten los NL Kl.I 5kg`. InnoDB tokenizes that as `Tomaten`, `los`, `NL`, `Kl`, `I`, `5kg` — and `innodb_ft_min_token_size` defaults to 3, so **`NL`, `Kl` and `I` never enter the index**. `Kl.I`, the class marker a produce buyer types constantly, becomes unsearchable.

Lowering that floor means a `my.cnf` change, an index rebuild and a server restart on infrastructure that is constrained enough to have fixed MySQL in the first place. Boolean mode also supports only a trailing `*`, so no mid-word matching, and the built-in stopword list is English.

Trade descriptions are punctuation-dense and abbreviation-dense. A tokenizer is a liability here, not an asset.

## The one Dutch rule

`AND`-of-`LIKE` already handles the plural class that merely suffixes, because the singular is literally a substring of the plural: `ui`→`uien`, `aardappel`→`aardappelen`, `wortel`→`wortels`.

It fails on open-syllable vowel shortening — `tomaat`→`tomaten`, `banaan`→`bananen`, `boon`→`bonen`, `peer`→`peren`, `kool`→`kolen`, `noot`→`noten`, `peen`→`penen`. A small set, but core produce, and the failure mode is an empty list after a correctly-spelled Dutch word.

Dutch has exactly one rule here: the singular carries a **doubled** vowel where the plural carries a single one. So collapse doubled vowels — `aa→a`, `ee→e`, `oo→o`, `uu→u` — in the stored search text **and** in the query:

| Typed | Normalised query | Stored text | Normalised stored | Match |
| --- | --- | --- | --- | --- |
| `tomaat` | `tomat` | `Tomaten los NL` | `tomaten los nl` | ✓ |
| `boon` | `bon` | `Bonen fijn` | `bonen fijn` | ✓ |
| `peer` | `per` | `Peren Conference` | `peren conference` | ✓ |
| `aardappel` | `ardapel` | `Aardappelen vast` | `ardapelen vast` | ✓ |

This is effectively a one-rule Dutch stemmer, and it is the only rule this vocabulary needs. It is deterministic, needs no maintained data, and is tested as a table of word pairs.

It can over-merge — `kool`→`kol` also reaches `kolen`. At ~1000 inspectable rows that is verifiable exhaustively rather than hoped about, and the cost of an occasional extra row in a live-filtering list is far below the cost of an empty one.

## Interaction: as-you-type

Search filters the catalogue list in place, debounced, from **2 characters**. There is no submit, no separate results page and no second ranking.

This is what makes whole-word recall a non-problem: a buyer reaches all tomatoes at `toma` without ever completing a word. The vowel rule above exists purely for the fast typist who enters a familiar name in one burst and would otherwise land on zero.

## Search text

One stored, normalised `search_text` column per mirrored article, built from:

- the **ERP article description**, verbatim
- the **article code** — these buyers order by code, and codes get typed
- the **canonical facet values**: origin, class, calibre, packaging, brand
- the article's **category name**
- the **article-group alias** (below)

Facet values are included alongside the description because the two use different vocabularies for the same fact: the description abbreviates to `NL` and `Kl.I` where the facet holds `Nederland` and `Klasse I`. A buyer typing either should succeed. They are safe to denormalise because [ADR-0001](./0001-cherry-owned-product-content.md) established facets are article-invariant — a change of origin mints a new article code, so a facet value never changes under a live code.

**Marketing copy and preparation/storage advice are excluded.** `AND`-of-`LIKE` produces no relevance score, so there is no way to weight prose lower — inclusion is all-or-nothing. A paragraph about tomatoes drags those articles into unrelated queries, and worse, it makes findability a function of merchandising completeness: the well-written article surfaces and the placeholder one does not. Searching storage advice so that `koel bewaren` returns a list is close to meaningless. ADR-0001's principle is that content may degrade while ordering never does; indexing copy would quietly convert content degradation into a discovery problem.

### Rebuilding

`search_text` has two upstreams, so it is application-maintained and rebuilt:

- by the **sync pass**, for the ERP-sourced parts — passed to [#9](https://github.com/mdegouw/Cherry/issues/9)
- by a **queued job** on any staff edit to a category name or an article-group alias

Normalisation therefore lives in testable PHP rather than in SQL. MySQL 8's `REGEXP_REPLACE` could do the vowel collapse at query time over a `CONCAT_WS` of the joined columns, which would remove invalidation entirely — but it pushes a linguistic rule somewhere it cannot be unit-tested against Dutch word pairs, and does concat-and-regex work per row per keystroke.

A full rebuild covers ~1000 rows in seconds, so **"rebuild everything" is always a safe recovery** and no incremental invalidation needs to be trusted.

## Suppression

Search applies exactly the suppressions the catalogue applies, and no others:

- `sellable` off — absent
- hidden after a month out of stock at that branch ([#4](https://github.com/mdegouw/Cherry/issues/4)) — absent
- **no category filter** — an uncategorised article is fully findable

The last one is load-bearing. ADR-0001 makes an uncategorised article orderable but absent from browse, with search as the safety net that stops a merchandising gap from becoming lost revenue. Filtering search by category presence would remove exactly the articles that most depend on it.

Suppression is per-branch, so the query is per-branch — consistent with [#4](https://github.com/mdegouw/Cherry/issues/4)'s finding that the catalogue caches once per branch rather than once per organisation.

## Ordering

There is no relevance score, so ordering is entirely a product decision.

**The availability band drives it**: `available` → `limited` → `sold out`, alphabetical by description within each. A buyer's job is to fill a crate today, and [#5](https://github.com/mdegouw/Cherry/issues/5) keeps sold-out articles visible, so without this they occupy prime space in a list built for buying.

The usual objection — that a band-ordered list reorders under the buyer as stock moves — does not apply. [ADR-0003](./0003-cherry-domain-model.md) ruled out websockets and the mirror is polled server-side, so there is no push channel. The list reorders only on a re-query: a keystroke, a filter change, a page load. Rows cannot move under someone sitting still.

**An exact article-code match sorts first**, above the band ordering. A buyer typing a full code wants that one article.

A suppressed code still returns **nothing** from search. Search is the shop, and the shop does not stock it. The buyer reaches such an article through order history, where ADR-0001 requires a "no longer available" page rather than a 404.

### The tripwire

Band ordering is a `CASE` over stock quantity joined per-branch against [#5](https://github.com/mdegouw/Cherry/issues/5)'s per-article-group thresholds, and `LIKE` cannot use an index. The catalogue query is therefore **permanently a scan-and-sort** — no index can serve it.

At ~1000 rows this is nothing, and it is the same scan `LIKE` already forces. It is recorded here because it is the first thing that breaks if the assortment ever grows by an order of magnitude, which [#4](https://github.com/mdegouw/Cherry/issues/4) says it will not.

## Aliases

Whether a buyer types Aartsen's article name, a botanical name or a regional term is a fact about Aartsen's customers that no amount of reasoning produces. Rather than guess, Cherry gives staff the remedy.

An **article-group alias** is staff-typed alternative search terms attached to an ERP article group. Its text folds into the `search_text` of every member article at build time, so the query is unchanged.

The article group is the right anchor for three reasons: one row covers the whole origin/class/calibre fan-out (a single category may hold dozens of near-identical tomato codes); the grouping already carries [#5](https://github.com/mdegouw/Cherry/issues/5)'s band thresholds and seeds ADR-0001's category defaults, so it is not new structure; and it reaches uncategorised articles.

Two alternatives were rejected:

- **Per-article aliases** — unmaintainable by construction against the fan-out.
- **Category-attached aliases** — they would fail for uncategorised articles, which are precisely the ones that depend on search. Rejected on principle, not preference.

Aliases **ship empty and are not a launch gate**. Description, facets, code and vowel normalisation carry v1; aliases are what staff reach for when a real miss is reported. An alias editor on the article group is passed to [#15](https://github.com/mdegouw/Cherry/issues/15).

## No typo tolerance

With no engine, typo tolerance would mean Levenshtein in PHP over the cached column as a zero-result fallback — cheap at this size, but built for demand nobody has measured, and a fuzzy fallback can mask a genuine catalogue gap.

The consumer-shop rationale does not apply here. It assumes a shopper who submits a query, gets nothing and leaves. A professional watching a list filter live sees the empty result at the fourth character and corrects it.

## Unowned decision: the datastore

MySQL 8.4 is a constraint on this ADR, not a product of it. Nothing in the map or any prior ADR has ever recorded why Cherry uses MySQL, and `.env` still carries the starter kit's `DB_CONNECTION=sqlite` while `compose.yaml` provisions `mysql:8.4`.

This is flagged rather than resolved. Had the engine been open, Postgres would have been the better fit for this specific problem — but the decision belongs to hosting and operations, not to search.

## The governing principle

**The corpus is small enough that the naive thing is the right thing.** Every option rejected here — an engine, an index, a tokenizer, a stemmer, typo tolerance — buys scale or linguistic sophistication that a thousand rows of Dutch produce descriptions do not need, and each would add a component that can be stale, unsynchronised or wrong. What the domain actually needs is one deterministic Dutch rule and a scan.
