# Content groups are derived, and marketing works a queue of them

---

Status: accepted

---

[ADR-0001](./0001-cherry-owned-product-content.md) settled *what* product content is and left the surface marketing produces it on deliberately open. This ADR fixes that surface — and in doing so changes two things ADR-0001 assumed about the model itself.

Decided in [#15](https://github.com/mdegouw/Cherry/issues/15), discharging the article-group alias obligation [#14](https://github.com/mdegouw/Cherry/issues/14) left on it.

## Who this is for

**One or two Aartsen employees, part-time, non-technical.** No agency, no freelancer, no intern — which matters because that is the exact trigger ADR-0004 named for revisiting its single-role decision. The trigger does not fire; one `staff` role stands.

Two consequences shape everything below. The surface must be usable by someone who opens it once a fortnight and has forgotten where things are. And since a company that has shot **zero** product photography for ~1000 articles has no photography function today, Cherry is *creating* this job rather than automating an existing one — so the throughput bottleneck is the shooting, never the uploading.

## The surface is a launch gate

ADR-0001 says v1 ships on mostly placeholders and photography is continuous backfill, which reads as permission to defer the back-office. That reading is wrong, and ADR-0001 supplies the reason without connecting it: **origin, class, calibre and packaging each mint a new article code** ([#12](https://github.com/mdegouw/Cherry/issues/12) confirmed it outright). Aartsen sells seasonal produce, so the assortment does not sit still while being slowly photographed — it **churns**. The Dutch tomato code dies in autumn and a Spanish one is born with no image, no copy and no category override.

So this is not a backfill project that completes and leaves a UI with nothing to do. It is **permanent maintenance at the rate the seasons rotate origins**, beginning on day one of production. A developer in that loop is a standing tax on a developer, forever, for work a non-technical person should do in fifteen minutes a week.

The escape hatch [#15](https://github.com/mdegouw/Cherry/issues/15) offered — seed content by a developer-run import while the UI comes later — was also checked and has an **empty source**: `aartsen.com` is a corporate site with no assortment listing, no article pages, no photography and no descriptions. There is nothing to import.

## Content groups are derived, not managed

**This supersedes ADR-0001's treatment of the content group as a grouping staff assemble.**

ADR-0001's own arithmetic forces it. Its claim is ~1000 shoots collapsing to ~200 groups — but a two-level category tree with a few dozen leaves cannot produce 200 buckets, and neither can the article-group list, which [#12](https://github.com/mdegouw/Cherry/issues/12) confirms is a **flat list** of a few dozen rows. A content group is therefore finer than both, at ~5 articles each: exactly one real vegetable's origin/class/calibre/packaging fan-out. **A content group is the vegetable.** It is the `product` entity ADR-0001 refused, with the danger removed, since ADR-0001 already guarantees a content group is never load-bearing for ordering.

And a vegetable is **computable**. The facets are separate structured fields ([#12](https://github.com/mdegouw/Cherry/issues/12), 1.5) whose values also appear in the description. Strip every facet value from the description and the residue is the vegetable's name — mechanical removal against known values, not fuzzy parsing. **Two article codes share a content group iff they share an article group and the same residue.**

What this buys is the largest saving available in this ADR: the seasonal churn above **self-heals**. The Spanish tomato code is born, its residue matches the dead Dutch one, and it inherits the existing photograph, copy and storage advice with **zero staff touches**. A hand-picked model forfeits that entirely and taxes every seasonal rotation.

So there is **no content-group creation screen, no assignment screen and no rule builder.** The only surface a derived group needs is a **remedy**: staff merging two residues that should have clustered and didn't (`Tomaten rond` versus `Ronde tomaten`). Rare, reactive, and the same ergonomic shape [ADR-0005](./0005-catalogue-search.md) chose for aliases — reached from a reported problem, never a blank list to fill in.

The failure mode is already covered by ADR-0001's governing principle: a residue that fails to cluster becomes a group of one, which shows a placeholder, which is **degraded presentation and nothing worse**. No article becomes unbuyable.

**Validation gate.** Whether residues actually cluster is a fact nobody has, because it needs [#17](https://github.com/mdegouw/Cherry/issues/17)'s article export. This decision is recorded *with that export as its check*, and **hand-picked assignment is the named fallback** if descriptions prove too inconsistent to strip.

## The queue is content groups, ordered by fan-out

[#15](https://github.com/mdegouw/Cherry/issues/15) proposed a worklist of "articles with no image, newest first". Both halves are wrong.

**The article is the wrong unit.** Photographing one article photographs its whole group, so a 1000-row article list would present the same tomato eight times and invite marketing to tick off seven of them by accident. The queue is over **content groups without an image** — ~200 rows at launch, each row exactly one shoot.

**"Newest first" degrades to arbitrary at launch**, when everything is new. The ordering marketing wants is commercial impact, and the available proxy is free: **the number of articles in the group, descending**. A group holding 8 codes is a vegetable Aartsen sells in eight origin/class/packaging combinations; one holding 1 is a curiosity. It compounds correctly — the top of the list is always the shoot that retires the most placeholders per photograph.

Order-line volume is the better signal and is **deliberately not built for MVP**: it does not exist at launch, so it would ship untested against real data. It is the obvious post-MVP refinement.

**Boundary with [#16](https://github.com/mdegouw/Cherry/issues/16).** #16 asks whether "articles with no image / no copy" is one surface or two. It lives **here and only here** — it is marketing's production queue, ordered by shoot impact, not an operational health signal. #16 keeps the genuinely operational lists: uncategorised articles, orphaned content, unmapped article groups, sync health. Two surfaces, no overlap.

## The day-one batch is absorbed by a staging tray

Hundreds of photographs may arrive on day one, which is the one case that could justify filename→article-code matching. It does not.

Three candidates were weighed: a filename matcher over hand-renamed files, a one-off developer import from a folder plus a spreadsheet, and a **staging tray** — drop all the files at once, they land as thumbnails, marketing assigns each to a group row by eye.

The tray wins on the argument [#12](https://github.com/mdegouw/Cherry/issues/12) itself used to insist on immutable article codes: *"dan staat er stilzwijgend een verkeerde foto bij een verkeerde groente."* A matcher and an import are both **silent-wrong-photo generators** — a human renaming 300 files or filling 300 spreadsheet rows will transpose some, and the matcher then attaches leeks to a tomato with complete confidence and no signal, until a customer finds it.

The tray makes that failure structurally hard, because **a photograph is self-describing in a way an article code is not.** A thumbnail of round tomatoes beside a row reading `Tomaten rond` is a match a human makes correctly in a second and essentially cannot get wrong. Two further costs disappear with it: nobody renames camera files (the rename *is* the fortnight the matcher was meant to prevent), and marketing never hand-types an immutable business key. ~200 assignments at a couple of seconds each is one sitting.

A derived content group also has **no stable human-typeable identifier** — its key is a description residue — so any filename convention would have to name an *article* code and have Cherry redirect the image to that article's group, meaning something other than what it says.

The tray is the trickle path too: one file dropped next month lands in the same place. **Filename matching returns only if the tray proves too slow against a real batch**, at which point the real filenames will tell us what convention to match.

## Saving is publishing

**No draft state, no publish button, no separate preview mode.**

A draft state would be a **third visibility state** beside "hidden" and "orphaned", giving every content read a third condition — and a draft that silently withholds a finished photograph is exactly the accumulation ADR-0001 refuses. A publish step also assumes a reviewer, and there isn't one: with one or two part-time people the author *is* the approver, so the button is theatre, and eventually a button they forget to press — leaving finished work invisible with no signal, because the queue would consider the group done. The safety a draft flow provides is already bought by ADR-0004's audit log, which covers content mutations with old and new values.

Preview is not a mode either. What marketing is uncertain about is whether a photo reads at tile size and whether copy fits, and a separate preview answers that badly because it is a second rendering that drifts from the real one. **The editors render the real customer-facing tile and article page inline**, live. Preview is simply what the editor looks like.

Two provisions make this safe, and are cheap:

- **No autosave, explicit save only.** This is what disposes of the interrupted-mid-sentence objection: an abandoned edit publishes nothing because it was never saved.
- **Replacing an image does not delete the previous file.** ADR-0004's log records *that* the photo changed; retaining the old file is what lets a wrong replacement be undone. Consistent with ADR-0001 refusing to cascade-delete content anywhere else.

## Copy and storage advice are group-level by default

**This supersedes ADR-0001's placement of copy as article-keyed content**, which stands only as the override.

Article-level text is nearly always redundant with data Cherry already displays. The only things distinguishing two codes in one group are origin, class, calibre, packaging and brand — every one a mirrored facet already rendered on the page — so article-specific copy would mostly restate in prose what the facet row states in structured form. Storage advice is clearer still: how to keep tomatoes does not depend on whether they came from Spain.

The consequence is the one that matters: **marketing's entire daily job is ~200 rows for text as well as photography.** A thousand-row copy backlog is a project nobody finishes; two hundred is a quarter's work at a relaxed pace. And it composes with the derivation — a new code inherits image, copy and storage advice together, still with zero touches.

Two editors, therefore:

- **Content group** — the queue row expanded: image, generic copy, storage advice. Where nearly all work happens.
- **Article** — its own image, own copy, own storage advice, and its category override. Exists because ADR-0001 keys content on the article code, but it is the exception and the surface should feel like one: **the queue never routes there.**

The accepted cost: a genuinely origin-specific selling point (*"Spaanse pruimtomaten zijn in januari zoeter"*) must be typed on one article and will not be inherited. That is correct — it is not true of the Dutch code.

## One article-group screen

Three decisions independently hung a staff-editable setting on the ERP article group, none knowing about the others: the **default category** (ADR-0001), the **availability-band threshold** ([#5](https://github.com/mdegouw/Cherry/issues/5)), and **search aliases** ([ADR-0005](./0005-catalogue-search.md)). Since the article group is a flat list, this is one table of a few dozen rows with three editable columns, **inline-editable, on one screen.**

The argument is not tidiness, it is the unmapped-group case. A new ERP article group arrives with **three** empty settings. On one screen that is a single new row with three blank cells and the gap is impossible to miss; across three screens it is three absences in three places nobody visits, and then articles land uncategorised with a nonsense threshold and no findability — three symptoms that get diagnosed as three unrelated bugs.

ADR-0004 already cleared the only real objection, having considered walling marketing off from commercial config and **declined** it, on the grounds that the same handful of trusted employees do both jobs and audit is the real control.

Note against the map's back-office fog, which files aliases with the `erp_status_code` and rejection-code label lookups: **aliases belong here instead**. Those lookups are keyed on ERP codes; an alias is keyed on the article group, like the two settings beside it.

Two smaller placements follow without argument:

- **The per-article category override lives in the article editor.** It is a Cherry-owned content field; it needs no screen of its own.
- **An unmapped article group's articles land uncategorised, and Cherry invents no fallback category.** ADR-0001 already makes uncategorised a staff worklist state, never customer-visible and never blocking an order. A catch-all *Overig* would be strictly worse: customer-visible, apparently deliberate, and it hides the gap by making the article look categorised.

## What v1 contains

Everything inside ADR-0004's `/beheer` shell, under one `staff` role, with every mutation audited.

1. **Content-group queue** — groups with no image, ordered by fan-out descending. Marketing's front door.
2. **Staging tray** — hundreds of files dropped at once, assigned to rows by eye.
3. **Content-group editor** — image, generic copy, storage advice.
4. **Article editor** — own image, own copy, own storage advice, category override.
5. **Article-group table** — default category, band threshold, aliases.
6. **Merge remedy** for two derived groups that should have clustered.

Deliberately **not** in v1: order-volume queue ordering (no data at launch); filename matching (returns only if the tray proves slow against real filenames); a content-group creation or assignment UI (**deleted permanently, not deferred**); draft/publish and a separate preview mode.

## Consequences

- A content group is a **derived** entity with no staff-authored membership. Its key is `article group + facet-stripped description residue`, and stripping is done against the structured facet values, never by parsing prose.
- [#17](https://github.com/mdegouw/Cherry/issues/17)'s export is now this ADR's validation gate as well as its own deliverable: it decides whether residues cluster. The fallback is hand-picked assignment.
- ADR-0001's content-group and copy provisions are superseded here; a forward pointer is added there so the contradiction is not discovered by a reader.
- The content queue is **marketing's**, and [#16](https://github.com/mdegouw/Cherry/issues/16) must not duplicate it. #16 owns the operational lists only.
- The article-group table is the first concrete piece of the map's unticketed configuration surface, and it takes [#5](https://github.com/mdegouw/Cherry/issues/5)'s threshold and [ADR-0005](./0005-catalogue-search.md)'s aliases with it.
- Image storage stays as ADR-0001 set it (`spatie/laravel-medialibrary`, queued conversions), with one addition: **replaced originals are retained**, so a `medialibrary` collection is not truncated on replace.
- No autosave anywhere in the content back-office. This is a stated provision, not an omission.
- Saving publishes, so there is no content state between "absent" and "live" — the visibility states remain exactly hidden and orphaned.
