# What can Thinkwise Indicium actually expose?

**Question:** What can Thinkwise's Software Factory / Indicium application tier expose as an API, natively or cheaply — and what does that constrain about the ERP API contract Aartsen will hand the Thinkwise dev team?

**Date:** 2026-08-02

**Where this lives:** This repo had no research-notes convention before this file. `docs/research/` is the new home for investigation write-ups; `docs/adr/` remains for decisions, `docs/agents/` for agent process docs.

---

## Confidence / source-quality note

Read this before trusting anything below.

**Strong (quoted directly from current official docs):** the OData surface and its query options, the resource-staging write protocol, the calculated-field taxonomy, the 1000-record response cap, the OAuth2 client-credentials setup, the container/IIS deployment story, the runtime lifecycle table.

**Weaker, flagged inline:**

- **No performance numbers exist.** Thinkwise publishes no latency SLA, no throughput benchmark, no documented rate limit, no request timeout, no max payload size, and no connection-pool sizing guidance. I searched specifically for each. Statements about latency below are qualitative guidance from Thinkwise or inference, and are marked.
- **The OData API page is stale on authentication.** It still says all authentication follows HTTP Basic. The IAM client-application docs and Thinkwise staff answers document full OAuth2. Do not design from the API page alone.
- **Community posts are used sparingly** and only where the answer is from an identified Thinkwise employee/moderator (Vincent Doppenberg, Mark Jongeling). These are first-party-adjacent, not formal documentation, and are labelled as such.
- **Undocumented areas, searched for and not found:** connector retry/back-off/dead-lettering semantics, minimum system-flow schedule interval, max URL length, max payload size, and any guidance on bulk delta extraction. Statements in those areas are marked as inference.
- **Everything here describes the platform, not Aartsen's installation.** Aartsen's licence, platform version, deployment topology and — critically — whether their ERP model carries trace/timestamp columns on the relevant tables, and whether those are trigger-filled, are all unknown. Several capabilities cited here are recent (response-size cap 2026.1; message brokers 2026.2), so they may not exist in Aartsen's version at all. See *Open questions*.

---

## 1. What Indicium exposes natively

Indicium is Thinkwise's generic application tier: it interprets the application model and serves business logic, workflow, security and reporting, and it is the API. It [supports multiple instances and hot-reloads model changes without downtime](https://docs.thinkwisesoftware.com/docs/indicium/indicium_general).

### Protocol

**OData v4.** Quoted: *"Indicium implements version 4.0 of the OData standard."* ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))

Base URL is `<web_app_root_url>/iam/<appl>/`, where `<appl>` is the application ID or alias in IAM ([same page](https://docs.thinkwisesoftware.com/docs/indicium/api)).

Two discovery documents come free:

- `/$metadata` — full CSDL description of every entity and operation *available to that user*. Useful as a permissions check as well as a contract.
- `/openapi` — an OpenAPI JSON spec, consumable by NSwag/Swagger/Postman. It can be filtered by `?entities=`, `?include_lookups=`, `?include_details=`, `?include_table_tasks=`, `?include_table_reports=` to keep the document small.
- `/application.svc` — OData service document (this is what Power BI's OData feed connector needs).

All three: [OData API](https://docs.thinkwisesoftware.com/docs/indicium/api).

### Supported query options

Explicitly listed as supported ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api)):

| Option | Supported | Notes |
|---|---|---|
| `$filter` | Yes | Also filters on **lookup display values** via `reference_id/display_column_id`, e.g. `$filter=contains(reference_id/display_column_id,'value')` |
| `$select` | Yes | |
| `$expand` | Yes | Works for lookups *and* 1:N details (`$expand=detail_<ref_id>`). `$select`/`$filter`/`$orderby`/`$prefilter` all work on the expanded target; only `$select` works on the source side |
| `$orderby` | Yes | |
| `$top` / `$skip` | Yes | Composable |
| `$count` | Yes | Idiom is `?$top=0&$count=true`, which returns `@odata.count` with an empty `value` array |
| `$search` | Yes | Free-text, but only over columns marked in the model's Search field |
| `$apply` | Yes | `groupby`, `aggregate`, `compute`, `filter` — and these **chain indefinitely** |
| `$batch` | **Not listed** | Absent from the documented operation list and from the docs entirely. Treat as unsupported |
| Delta links / `$deltatoken` | **Not listed** | Absent. See §6 |

**Aggregates are genuinely supported.** `$apply` supports `groupby` with `aggregate`, plus scalar computations (`Year`, `Month`, `Isoweek`, `Ceiling`, `Substring`, `Concat`, …), and transformations chain. The docs' own example selects all customers with more than five projects in 2021 in a single request:

```
/project?$apply=compute(year(project_date) as project_year)
                /filter(project_year eq 2021)
                /groupby((customer_id), aggregate(project_id with count as count))
                /filter(count gt 5)&$select=customer_id
```

([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))

### Thinkwise-specific extensions

- **`$prefilter`** — activates named prefilters defined in the model: `?$prefilter=prefilter_1,prefilter_2`. Critically, *"Authorization prefilters are applied automatically, regardless of whether you explicitly specify them."*
- **`$deselect`** — inverse of `$select`; keep everything except named columns. Useful to drop binary/HTML columns.
- **`$seek`** — returns the *row index* of records matching an expression, for jump-to-row paging.
- **`$export`** — server-side export to `.xls`/`.xlsx`/`.csv`.
- **`$query`** — moves query options into a `text/plain` POST body to dodge URL-length limits. The actual max URL length is not documented.

All: [OData API](https://docs.thinkwisesoftware.com/docs/indicium/api).

### Paging is now mandatory above 1000 rows

**Breaking change in platform 2026.1.** Quoted:

> *"From this version onwards, Indicium now limits the response size for tables and view to 1000 records for Thinkwise platform version 2026.1 and higher. We have implemented this change to protect both Indicium and the database from accidental or malicious large data requests."*

> *"External applications and third-party integrations and other clients may be affected by this change. Previously, these clients received all records from table endpoints in a single response. […] To retrieve all records, these integrations must now use the `@odata.nextLink` to fetch subsequent pages of data."*

([2026.1 release notes](https://docs.thinkwisesoftware.com/blog/2026_1))

So there **is** server-driven paging, via `@odata.nextLink`. Expanded details and lookups are each capped at 1000 too, inheriting the parent's exemption status. Per-endpoint exemptions can be configured in the Software Factory (*Maintenance > Runtime Configurations > tab Response size limit exemption*) or IAM (*Authorization > Applications > General settings > tab Response size limit exemption*). Platform 2025.3 and earlier can opt in via extended property `enableresponsesizelimiting = 1`. ([2026.1 release notes](https://docs.thinkwisesoftware.com/blog/2026_1))

### What the model controls — opt-in vs automatic

This is the question that most affects scoping. The answer is split:

- **Tables and views: automatic, gated by authorization.** Quoted: *"Through Indicium, a user will only have access to those tables, views and procedures for which the user has been authorized in IAM."* ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api)) There is no per-table "expose as API" switch documented — every table/view in the model is an endpoint for a user with rights to it. Exposure is therefore an **authorization** exercise, not a modelling one.
- **Tasks and reports: automatic**, same gating — they appear as POST endpoints.
- **Subroutines (functions/stored procedures): opt-in, per subroutine.** Quoted: *"For security reasons, subroutines are not exposed by the Indicium OData API by default. To expose a subroutine using Indicium, enable the API option of the subroutine in the Software Factory."* ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api)) Only SQL `Function` and `Procedure` types can be APIs — CLR functions/procedures, DLL assemblies and DB2 external routines **cannot**. Service names and parameter names can be renamed via **Subroutine alias**, so the public API need not follow Thinkwise's `lowercase_underscore` convention. ([Subroutines](https://docs.thinkwisesoftware.com/docs/sf/subroutines))
- **Table variants** are addressable as `entity.variant`, e.g. `/project.overview`. A [variant](https://docs.thinkwisesoftware.com/docs/sf/variants) is a re-presentation of a table with different prefilters/columns *without creating a view* — a cheap way to give Cherry a purpose-shaped projection over an existing table. Variant rights are *"always equal to or more limited than in the standard."*

### Reads are cheap; writes are a protocol

Reads are plain GETs. **Writes are not plain POSTs.** Every insert, update, task and report goes through **resource staging**: *"Every insert, update, task and report action is represented by a staged resource."* ([Resource Staging](https://docs.thinkwisesoftware.com/docs/indicium/resource_staging))

The multi-request form is `POST /table/stage_add` → `PATCH staged_table(#)` → commit. A **single-request** form exists and is what Thinkwise recommends for machine callers: *"When Indicium is called automatically by some third party application or service […] single request resource staging may be preferred."* ([Resource Staging](https://docs.thinkwisesoftware.com/docs/indicium/resource_staging))

Even in single-request form, writes are expensive, because Indicium runs model logic per field:

> *"When a request body contains parameters, the layout procedure is executed to ensure that the parameters can be edited by the user executing the API. […] For each parameter in the request body, Indicium will execute a layout procedure to detect if the column is editable. If a column is not editable, the action aborts […] After patching the value, a default is executed that can change the column value."*

([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))

Other write constraints:

- **One record per request** for inserts. No bulk insert on table endpoints. ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))
- Updates/deletes require the **entire primary key** in parentheses, and **all query-string parameters are ignored** — no `$filter`-based bulk update or delete.
- Lookups can be resolved by natural key rather than surrogate ID via `choose`, `choose_by_display`, and `choose_by_element` (with a `language_hint`) — genuinely useful, since Cherry will hold ERP business keys, not ERP surrogate IDs. ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))
- **Multi-row task execution** exists as a batching escape hatch for *tasks*: stage once, then `POST .../add_context` per row, then commit once. ([Multi-row task execution](https://docs.thinkwisesoftware.com/docs/indicium/multi-row_tasks))
- An **Import API** accepts `.xls`/`.xlsx`/`.csv` uploads into a subject, but each row is a separate insert running default and layout logic, and column mapping is by *translated header text in the calling user's language*. ([Import API](https://docs.thinkwisesoftware.com/docs/indicium/importapi))

Errors come back in a base64-encoded **`TSFMessages`** header carrying a JSON payload with `MessageID` and `TranslatedMessage` — Cherry must decode this to surface ERP validation failures. ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))

---

## 2. Push vs pull

**Short answer: there is no subscribable event stream, but the ERP *can* be made to call Cherry over HTTP. Push is something the Thinkwise team builds in the model, not something Cherry registers for. Realistic push latency is tens of seconds.**

### First, a disambiguation that will otherwise mislead the contract

Thinkwise's marketing says: *"Third-party applications and services, in turn, can connect to Thinkwise applications with minimal effort using the provided webhooks and REST API."* ([Platform overview](https://docs.thinkwisesoftware.com/docs/overview/platform_overview), repeated on [Indicium general](https://docs.thinkwisesoftware.com/docs/indicium/indicium_general))

**"Webhooks" here means inbound** — Indicium *receiving* webhook calls. The same framing appears on the `/public/` API: *"For access without authentication, you can make public API calls. This allows you to easily integrate with messaging services like: Azure PubSub, Service Bus and Event Hubs; Amazon SQS and SNS"* ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api)), and the roles doc explains why: *"methods like webhooks or subscriptions do not always allow authentication"* ([Roles](https://docs.thinkwisesoftware.com/docs/sf/roles)). All of that is other systems calling *in*.

There is **no webhook/subscription registry**. The [endpoints reference](https://docs.thinkwisesoftware.com/docs/indicium/endpoints) lists every endpoint category — OData API, Process flows API, File API, Import API, `/license`, `/health` — and contains no subscription, notification, change-feed or delta endpoint.

### Outbound HTTP does exist: the Web connector and HTTP connector

This is the mechanism by which the ERP can call Cherry. Process flows have connector action types including an **HTTP connector** and a **Web connector** ([Process flow connectors](https://docs.thinkwisesoftware.com/docs/sf/process_flows_connectors)).

- **HTTP connector** (low-level): takes URL, method (GET/POST/PUT/PATCH/DELETE/…), headers as a JSON Key/Value array, cookie, Content-Type, body, authentication type (None/Basic/Bearer/Digest/Windows-Negotiate), bearer token or username/password, and a **Timeout** — *"An integer that indicates the timeout of the request in milliseconds. Default is 100,000."* It returns a connector status code (0 = success; −1…−6 for unknown error, invalid URL, invalid method, invalid headers, invalid cookie, timeout) plus the HTTP status code, response headers and body.
- **Web connector** (preferred): references a reusable **Web connection** model object — base URL plus auth, with configured endpoints — and extracts response data into typed output parameters via JSONPath/XPath/regex. The docs say web connections *"are easier to maintain, configure and re-use."* ([Web connections](https://docs.thinkwisesoftware.com/docs/sf/web_connections), [OAuth connectors](https://docs.thinkwisesoftware.com/docs/sf/process_flows_oauth_connectors), [OAuth servers](https://docs.thinkwisesoftware.com/docs/sf/oauth_servers))

Auth toward Cherry can be bearer token, API key or basic — so securing the callback is straightforward.

**Undocumented, and therefore Cherry's problem:** retry, back-off, dead-lettering and transactional coupling are **not documented** for either connector. Error handling is *"branch on the status code with a Decision action"* ([Process flow actions](https://docs.thinkwisesoftware.com/docs/sf/process_flows_actions)) — there is no exception or compensation model. Connectors are **synchronous** by default; the only documented async escape is the *Execute system subflow* action, whose **Asynchronous** parameter when set to 'Yes' is *"fire and forget"* (default 'No').

### The hard part is what *starts* the outbound call

Process flows start from: a user action, *"an API action such as adding a row, deleting a row, or executing a task"*, a deep link, a **Custom protocol** endpoint, or a **schedule** (system flows only) ([Process flows](https://docs.thinkwisesoftware.com/docs/sf/process_flows)).

The critical caveat, from Vincent Doppenberg (Thinkwise moderator, marked best answer):

> *"If you perform an insert/update/delete statement from another application, directly on the database, a process flow won't fire, but a trigger will."*

([Triggers vs process flows](https://community.thinkwisesoftware.com/questions-conversations-78/triggers-vs-process-flows-1676))

**This is the single most important fact in this section.** Process-flow-driven push only fires for writes that go through Indicium or the GUI. Any batch job, DBA script, SSIS package or legacy interface writing straight to SQL Server bypasses it silently. In a wholesaler's ERP, stock and price movements are *exactly* the kind of thing likely to arrive via batch interfaces.

Triggers *are* real database events (before/after/instead-of insert/update/delete, [Logic concepts](https://docs.thinkwisesoftware.com/docs/sf/business_logic)) — but triggers are T-SQL and cannot make HTTP calls.

**So the only robust shape is: trigger → write to an outbox/queue table → scheduled system flow drains it via the Web/HTTP connector → POST to Cherry.** This is not just inference — Thinkwise ships it. There is a Thinkstore solution **"Standard http queue with system flow (API)"**: a queue table holding the input and output parameters of an HTTP connector action, plus a system flow that executes the calls and writes responses back ([Thinkstore model updates 2025.1](https://community.thinkwisesoftware.com/product-updates/thinkstore-model-updates-2025-1-5790)). It is a transactional outbox, and it is Thinkwise's own answer.

### Push latency floor

System flows are scheduled in IAM (*Authorization > Applications > General settings > Scheduled system flows*) ([System flows](https://docs.thinkwisesoftware.com/docs/iam/system_flows)). The *Run now* task documentation warns *"it can take up to 30 seconds before the system flow is started"*, and concurrency is configurable (multiple instances allowed or not). A schedule log records UTC start/end times.

**Inference:** realistic outbox-drain push latency is **tens of seconds, not sub-second**. No minimum schedule interval is documented.

### Message brokers: MQTT publish-only, and new

Added in platform **2026.2** ([2026.2 release notes](https://docs.thinkwisesoftware.com/blog/2026_2), [Thinkwise Platform 2026.2](https://community.thinkwisesoftware.com/product-updates/thinkwise-platform-2026-2-6842)). Quoted from the docs:

> *"Currently, you can only publish messages to a message broker. Subscribing to messages from a message broker will be supported in a future release."*

> *"Currently, only MQTT is supported."*

Brokers: Chariot, EMQX, HiveMQ, Mosquitto, Generic MQTT. Auth via username/password, OAuth client credentials or access token. Driven from a **Publish message** process action, available in system flows. ([Message brokers](https://docs.thinkwisesoftware.com/docs/sf/message_brokers), [Process flow actions](https://docs.thinkwisesoftware.com/docs/sf/process_flows_actions))

**No Kafka, AMQP, RabbitMQ, Azure Service Bus, Event Hubs or SQS/SNS connector exists.** The [functional integrations catalogue](https://docs.thinkwisesoftware.com/docs/overview/integrations_functional) lists maps, document generation, OCR, signing, printing, email, Teams/Slack/Power Automate, M365, identity providers, ERP/CRM and payments — and no message-queue platform. Those can be reached over generic HTTP, but not as first-class connectors.

Note this is **2026.2 functionality** — whether Aartsen's platform version has it at all is an open question.

### Live-update channels are internal only

Indicium uses SignalR over WebSockets, but only for its own concerns: Universal GUI import progress, the process flow monitor page, and the error log page (*"maintains an open connection with Indicium where new log entries are added automatically"*) ([Indicium troubleshooting](https://docs.thinkwisesoftware.com/docs/indicium/indicium_troubleshooting), [Indicium deployment](https://docs.thinkwisesoftware.com/docs/deployment/indicium) — IIS needs the WebSocket Protocol feature). **There is no public SignalR hub, WebSocket feed or SSE stream for entity data changes.** Web Push notifications (VAPID/PWA) exist but are browser push to *human users* ([User notification](https://docs.thinkwisesoftware.com/docs/iam/user_notification)).

### Inbound (Cherry → ERP) is the easy direction

**Custom protocol** process flows give an arbitrary HTTP endpoint at `/indicium/open/iam/<appl>/<process_flow>` — *"The first segment of the endpoint is /open because it is a special endpoint, entirely separate from Indicium's regular OData endpoints."* Any method, body and response are allowed; it supports *"SOAP, GraphQL, OData and even your own proprietary protocol"*, works only for system flows, takes an optional API alias, and **cannot be scheduled** ([Process flows](https://docs.thinkwisesoftware.com/docs/sf/process_flows), [Case: a message protocol-independent web service](https://community.thinkwisesoftware.com/news-blogs-21/case-a-message-protocol-independent-web-service-part-1-of-2-1947)).

The regular [Process flows API](https://docs.thinkwisesoftware.com/docs/indicium/process_flows) is a start/continue/cancel state machine (POST to start → 204 if complete, 201 + Location if it pauses for input → POST `continue` with a mandatory `status_code` → DELETE to cancel).

**Change detection is a cheap "has anything changed?" probe** — see §6. It is the closest thing to a push signal and it is still pull.

---

## 3. Latency & throughput — is a live lookup at page render viable?

**Verdict: not forbidden, but unproven, and you must not design as if it is free.** No published SLA, benchmark, rate limit, or timeout exists.

### Documented guidance

Thinkwise's own [performance knowledge-base page](https://docs.thinkwisesoftware.com/docs/kb/performance) is entirely qualitative. The parts that bear on a price lookup:

- *"The calculation is executed at the time of selection. Depending on the situation, this can require a lot of computing power and time. A view with complex calculations that is frequently consulted is less suitable because of this."*
- *"Views that make use of other views are advised against."*
- *"Views that are accessed with a filter on columns that are a result of a function are strongly advised against because the function must be calculated for all rows to be able to filter the view."*
- *"When functions or case statements are used in the where clause of an SQL query, the SQL engine must process all records one by one […] it is recommended to only use functions in the where clause on small sets and always disable sorting and filtering on calculated fields."*

Every one of those is a direct warning about the shape of "customer-specific computed price, filtered per request".

### Bulk reads are explicitly out of scope for Indicium

Vincent Doppenberg (Thinkwise moderator), answering a question about transferring a large table:

> *"Indicium operates on the assumption that data is retrieved one page at a time or at least with a top statement of a few hundred or a few thousand records."*

> *"Indicium needs to build the full response object in memory before serializing it to the client as JSON."*

> *"Indicium is not the recommended medium for transferring 22 million records"* — recommending Azure Data Factory or SSIS instead.

([Indicium API performance — large table transfer](https://community.thinkwisesoftware.com/questions-conversations-78/indicium-api-performance-large-table-transfer-6012))

Combined with the hard 1000-row cap (§1), a nightly full-catalogue sync is many paged requests, not one.

### Cold start is real

> *"the model needs to be loaded, which may take multiple seconds on top of the request performed"*

([Indicium troubleshooting](https://docs.thinkwisesoftware.com/docs/indicium/indicium_troubleshooting))

Mitigations are all documented: `Applications:Preload` reduces *"response times for users who are the first to access these applications after a cold start or restart"*; `Applications:RemoveUnusedModelAfterHours` defaults to **72 hours** ([Indicium configuration](https://docs.thinkwisesoftware.com/docs/deployment/indicium_configuration)). On IIS: set app-pool **Idle Time-out to 0** (default 20 minutes), **StartMode = AlwaysRunning**, and enable Preload ([Indicium deployment](https://docs.thinkwisesoftware.com/docs/deployment/indicium)).

Note the operational trap: **adding or changing a client application requires an Indicium restart** (*"Always restart Indicium after you added or changed a client application"*, [Client applications](https://docs.thinkwisesoftware.com/docs/iam/client_apps)) — which reintroduces the cold start.

### Deployment and scaling

- Indicium is an **ASP.NET Core** app. Current version requires **.NET 10** and the ASP.NET Core Hosting Bundle. Kestrel is built in but *"for security and management reasons, we do not recommend exposing the built-in web server in production environments"* — put IIS in front. Separate app pool per Indicium; .NET CLR version **"No Managed Code"**. ([Indicium deployment](https://docs.thinkwisesoftware.com/docs/deployment/indicium))
- **Containers are first class**: image `registry.thinkwisesoftware.com/public/indicium`, port 8080 since 2025.1.10, with Docker Compose, Docker Swarm and **AKS/Kubernetes** manifests documented. ([Container registry — Indicium](https://docs.thinkwisesoftware.com/docs/deployment/container_registry_indicium), [container deployment overview](https://docs.thinkwisesoftware.com/docs/deployment/container_deployment_overview), [AKS](https://docs.thinkwisesoftware.com/docs/deployment/container_deployment_aks))
- **Statelessness has an asterisk.** The general page claims Indicium *"is stateless and supports multiple instances"* ([Indicium general](https://docs.thinkwisesoftware.com/docs/indicium/indicium_general)), but the scaling docs require you to choose **either** sticky sessions **or** Redis to share user session state — and *"never use both at the same time"* ([Scalability](https://docs.thinkwisesoftware.com/docs/deployment/scalability), [Scaling Indicium](https://docs.thinkwisesoftware.com/docs/deployment/scaling-indicium)). The only capacity figure found anywhere in the docs is that horizontal + sticky sessions suits *"hundreds of concurrent users"* — a strategy qualifier, not a benchmark.
- **All database traffic runs as one account**: *"will perform all database traffic with a single user, the Database Pool User"*, and domain accounts are forbidden in `appsettings.json` because they *"can cause severe performance degradation"* ([Indicium deployment](https://docs.thinkwisesoftware.com/docs/deployment/indicium)). Pool sizing and command timeout are **not documented** — inference: standard ADO.NET defaults apply.
- **Measure with `Server-Timing`.** Indicium returns Server-Timing response headers broken down by CRUD, application logic, file storage and misc, and supports Azure Application Insights ([Indicium troubleshooting](https://docs.thinkwisesoftware.com/docs/indicium/indicium_troubleshooting)). This is the tool to answer the latency question empirically.
- A `/health` endpoint exists and *"can be accessed anonymously"* ([Endpoints](https://docs.thinkwisesoftware.com/docs/indicium/endpoints)).

### Indicium version — settled

There is now only one Indicium. The [runtime lifecycle policy](https://docs.thinkwisesoftware.com/docs/kb/lifecycle_policy) lists **Indicium** as GA since 2019.1 with no EoL date, and lists **Indicium Basic** under *End of Service Life* as of **2025** (alongside Mobile GUI (Indicium Basic), 2024). Historically these were "Indicium Universal" vs "Indicium"; Thinkwise's own comparison post already recommended Universal for third-party read access: *"If your third-party client is only reading data, deleting data or calling subroutines via Indicium, then it is recommended to use Indicium Universal."* ([Indicium vs Indicium Universal](https://community.thinkwisesoftware.com/news-blogs-21/indicium-vs-indicium-universal-445)). One residual trace of the split survives in the model: subroutines have both an **API** checkbox and an **API (legacy)** checkbox, the latter *"to publish it by Indicium Legacy"* ([Subroutines](https://docs.thinkwisesoftware.com/docs/sf/subroutines)). Cherry should use the modern one.

---

## 4. Computed values — can a customer-specific price be exposed as cleanly as a stored column?

**Partly. There is a clean read path and a clean call path, but they have opposite trade-offs, and the "queryable computed column" option is the one with the sharpest performance warnings.**

The Software Factory's [calculated field taxonomy](https://docs.thinkwisesoftware.com/docs/sf/data_model) offers four calculation types on a column:

| Type | Where it runs | Exposed via OData? | Filterable? |
|---|---|---|---|
| **Expression** | *"the UI executes the calculations. This is not stored in the database, neither is the column."* Can use the `t1` alias. | Added to the SELECT clause, so it appears as a column | Yes in principle, but see warnings |
| **Calculated column** | *"a virtual column that calculates its values"* — a SQL Server computed column. Row-local only; **subqueries not allowed**; *"cannot be applied with a view"* | Yes, behaves like a stored column | Yes; can be indexed if `PERSISTED` |
| **Calculated column (function)** | *"the database executes the calculation with help of a function"* — SF wraps your query into a function. **Can** contain subqueries, e.g. `select sum(order_total) from sales_order where sales_order_id = @sales_order_id` | Yes | Yes, but strongly discouraged — see below |
| **Query** | value calculated by UI or database, not stored | Yes | — |

Adding `PERSISTED` to a calculated column *"means that the value is calculated when a record is inserted and stored in the database, instead of being calculated at runtime"* ([Data model](https://docs.thinkwisesoftware.com/docs/sf/data_model)) — which converts it into an indexable stored column, but only for row-local expressions.

**Two hard limits that matter for pricing:**

1. **Expressions don't exist outside the UI/API select path.** *"The expression is not usable in the back-end logic and in other applications (for example, reports)"* ([Performance](https://docs.thinkwisesoftware.com/docs/kb/performance)). So an expression-based price can't be reused by ERP order logic — you'd get two implementations of the price and they will drift.
2. **Filtering on function-backed computed columns is the documented worst case.** *"Views that are accessed with a filter on columns that are a result of a function are strongly advised against because the function must be calculated for all rows to be able to filter the view."* ([Performance](https://docs.thinkwisesoftware.com/docs/kb/performance)) A per-customer price is exactly a function of `(customer, branch, product, moment)`. Filtering a product list *by* computed price will table-scan.

**Views and snapshots.** Views are full model citizens (table type *View*, with Meta auto / Meta custom / Template creation methods) and are exposed like tables ([Data model](https://docs.thinkwisesoftware.com/docs/sf/data_model)). A view is the natural home for a per-customer price *if the customer can be a filterable column* rather than a parameter. Snapshots (materialised) are **DB2/Oracle only** — not available on SQL Server.

**Table-valued functions as tables: not available.** Quoted from the data model docs, under *"Table-valued function as a table"*: **"This functionality is not yet available."** ([Data model](https://docs.thinkwisesoftware.com/docs/sf/data_model)) This kills the most elegant option — a parameterised, still-queryable price feed.

**The action-style path: subroutines as OData functions.** A subroutine of type `Function` can be marked as an API and can return **Scalar** or **Table** (*"The result is a table containing multiple rows […] the columns can be named in the Subroutine return columns tab"*) ([Subroutines](https://docs.thinkwisesoftware.com/docs/sf/subroutines)). It is then called with `GET /my_function(customer_id=123,branch_id=4)`.

This is the cleanest way to expose ERP pricing logic — one function, one implementation, reused by ERP logic. **But**: for function calls the docs state *"All query string parameters will be ignored."* ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api)) So **you cannot `$filter`, `$top`, `$orderby` or page a function result.** Whatever the function returns, you get all of it, shaped only by its parameters.

**Net for Cherry:** a computed price is *not* as cheap as a stored column. The realistic options are (a) a purpose-built function endpoint returning a price list for a customer+branch — clean, reusable, but unfilterable and unpaged; or (b) an ERP-side materialised price table that pricing logic maintains, exposed as an ordinary filterable, pageable, indexable table. Option (b) is the one that behaves like a normal column, and it is custom work.

---

## 5. Authentication for a server-to-server consumer

**OAuth 2.0 client credentials. This exists natively and is the right answer.**

> *"Machine-to-machine API access grants another application direct access to your own application, using the Client credentials grant flow of the OAuth2 protocol."*

([Client applications](https://docs.thinkwisesoftware.com/docs/iam/client_apps))

Setup, per the docs: create a client application with a **Client ID**, tick **Enabled**, **clear "Support OpenID Connect"** (which makes API access mandatory), set grant type to **Client credentials**, generate a **client secret**, and — the important field — bind it to a **User**: *"select a user. This is usually a service account. The client application will use it for authentication."*

Also available: Authorization Code grant for delegated (per-user) access; OpenID Connect with Indicium as provider ([OpenID](https://docs.thinkwisesoftware.com/docs/iam/openid)); external IdPs such as Entra ID for inbound user login ([OpenID providers](https://docs.thinkwisesoftware.com/docs/iam/openid_providers)); HTTP Basic against IAM/RDBMS/Windows/Kerberos users ([Users](https://docs.thinkwisesoftware.com/docs/iam/users)); and anonymous `/public/` calls. **No API-key mechanism is documented** — client ID + secret is the nearest equivalent.

Tokens: JWT bearer, **valid one hour**, with single-use rotating refresh tokens (Vincent Doppenberg, Thinkwise — [Authentication method Indicium Universal for external applications](https://community.thinkwisesoftware.com/questions-conversations-78/authentication-method-indicium-universal-for-external-applications-2396)). Interactive sessions expire after **30 minutes** of inactivity by default, tunable in Global Settings ([Users](https://docs.thinkwisesoftware.com/docs/iam/users)).

### Identity and authorization — the sharp edge

**The call runs as a real IAM user**, the one bound to the client application. All normal authorization applies: user group membership, role rights, application rights, and prefilters. Row-level security is done with prefilters, and authorization prefilters cannot be switched off:

> *"It is possible to indicate per role that a prefilter is intended to authorize data. In this case, the checkbox Data authorization prefilter can be checked in the Model rights screen. As a result, the prefilter cannot be disabled by users assigned to this role."*

([Roles](https://docs.thinkwisesoftware.com/docs/sf/roles))

**This creates a real design problem for customer-specific pricing.** A single service account has a single, fixed authorization context. If ERP prices are scoped per customer by prefilter, one service account either sees everything (and Cherry must scope correctly itself) or sees one customer's slice. The documented alternatives are both poor fits:

- **Authorization Code grant** gives per-user tokens but needs an interactive login — only workable if Cherry's customers authenticate against IAM directly, which collides with ticket #6.
- **User simulation** has an API (`GET`/`PATCH`/`DELETE account/api/usersimulation`) and *"Simulations respect all user-specific access controls and preferences"* — but the docs frame it as a **testing and debugging tool**, it requires **Simulator** (near-admin) rights, and it is **stateful** (PATCH to start, DELETE to stop). Under concurrent per-request web rendering this is hazardous, and Thinkwise does not document its concurrency semantics. ([User simulation](https://docs.thinkwisesoftware.com/docs/iam/user_simulation))

**Conclusion:** Cherry should authenticate as **one service account with breadth**, pass the customer identity as an explicit **parameter**, and enforce customer scoping in Cherry. The ERP API should not rely on IAM prefilters to scope per-customer data for Cherry.

Also note, for public/anonymous calls: *"the function `tsf_user()` will return the username of the database pool user"* and `tsf_is_public_request` is set — so ERP logic can distinguish them, but identity is lost. ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api))

**One concrete must-do:** service accounts are *"subject to the concurrent session limit by default"*, and there is an **Exclude from max. # sessions** setting to lift it ([Users](https://docs.thinkwisesoftware.com/docs/iam/users)). Without it, a busy webshop will hit the ceiling. Whether a service account consumes a named-user licence is **not documented** — ask Thinkwise.

---

## 6. Change tracking / deltas

**There is no OData delta-link mechanism. Cherry must poll with a timestamp filter, and the ERP model has to be built to make that possible.**

`$deltatoken` and delta links appear nowhere in the [OData API docs](https://docs.thinkwisesoftware.com/docs/indicium/api), and are not in the supported-operations list. No CDC or SQL Server Change Tracking integration is documented as an Indicium-exposed feature.

What *does* exist:

**1. Change detection — a cheap "has this changed since T?" probe, and it is API-callable.**

> *"The logic concept Change detection allows you to inform the user interfaces during certain events whether or not a subject has been changed and needs to be refreshed. […] Once enabled, the Change detection logic can also be called directly via the API."*

It takes an optional **`last_refresh_utc`** input parameter — *"The date and time used for the Change detection"* — plus an optional `variant_id`. It is enabled per subject at *User interface > Subjects > tab Settings > tab Performance > checkbox Change detection*, implemented as a control procedure with code group `CHANGE_DETECTION`. ([Logic concepts](https://docs.thinkwisesoftware.com/docs/sf/business_logic))

Caveats: it was designed for GUI auto-refresh, so it returns a *boolean-ish signal, not a changed-row set*; and *"Enabling this logic without assigning any code will effectively disable auto-refresh."* It is a good cheap guard ("should I bother re-syncing?") but not a delta feed.

**2. Trace fields — the "last modified" convention, but it is an add-on, not a guarantee.** This is the correction that matters most. Trace columns (`insert_user`, `insert_date_time`, `update_user`, `update_date_time`) are **not a built-in framework feature of generated applications**. They come from a Dynamic Model solution distributed via the Thinkstore:

> *"Trace field changes: automatically add trace fields to all tables so a user can see who added the record or who made the last change."*

([Dynamic model](https://docs.thinkwisesoftware.com/docs/sf/dynamic_model))

Once installed it is opt-out per table (tag a table `no_trace` to skip it). Crucially, **it ships in two variants**: filling via a `default_fill_trace_columns` control procedure, or via `trigger_fill_trace_columns` triggers ([Trace your changes by adding trace fields](https://community.thinkwisesoftware.com/questions-conversations-78/trace-your-changes-by-adding-trace-fields-1107), Frank Wijnhout, Thinkwise; refreshed in [Thinkstore 2024.1](https://community.thinkwisesoftware.com/product-updates/release-notes-thinkstore-2024-1-4730)).

**If the *defaults* variant is used, direct-to-database writes will not update `update_date_time`** — the same blind spot that stops process flows firing on direct DB writes (§2), and it would silently break Cherry's delta sync. Cherry must require the **trigger** variant.

(The behaviour description *"When a new row is added, the Added and Modified fields are filled… When an existing row is changed, only the Modified fields are filled"* in [Generic features](https://docs.thinkwisesoftware.com/docs/sf/generic_features) describes the Software Factory's own model, which has these columns — it is not a statement about generated customer applications.)

**3. System versioning (temporal tables) — history exists, but is not API-visible.** SQL Server and Oracle: tick **System versioning** on a table, set a **Retention period**, and SF generates a `[table_id]_history` table plus two hidden datetime columns. But:

> *"The two date fields (`tsf_valid_from` and `tsf_valid_to`) are generated in the script automatically, as hidden fields. They are not created in the Column list in the Software Factory."*

([Data model](https://docs.thinkwisesoftware.com/docs/sf/data_model))

The history table likewise appears in the CREATE/UPGRADE scripts *but not in the Software Factory's table list*. Because neither the history table nor the validity columns are model objects, **they are not exposed through OData and not addressable in `$filter`**. Surfacing history requires the Thinkstore dynamic-model solution *"Logging data available in the GUI"*, which generates a view per system-versioned table ([Dynamic model](https://docs.thinkwisesoftware.com/docs/sf/dynamic_model), [Make your logging data available in the GUI](https://community.thinkwisesoftware.com/questions-conversations-78/make-your-logging-data-available-in-the-gui-1154)). Temporal tables therefore give the ERP team raw material for a delta view — they do not themselves give Cherry a queryable delta.

**4. `ROWVERSION` is a modelled column type.** The Software Factory has a *"Use for optimistic locking"* column setting, and *"Only if the datatype is ROWVERSION, Use for optimistic locking is selected by default"* ([Data model](https://docs.thinkwisesoftware.com/docs/sf/data_model), [Thinkstore 2024.1](https://community.thinkwisesoftware.com/product-updates/release-notes-thinkstore-2024-1-4730)). **Inference:** an exposed `ROWVERSION` column would be a strictly monotonic, gap-free high-water mark — materially more reliable than a datetime for delta extraction, since it has no clock-skew or same-millisecond-tie problems. Thinkwise does not document this use, but it is worth putting on the table with their dev team.

**5. Datetime filtering works, but there is no `now()`.** ISO 8601 literals are supported, e.g. `?$filter=claim_start_date_time gt 2020-07-08T13:58:16Z`. Note the gotcha: **`now()` is not supported by Indicium**, so Cherry must always supply literal timestamps ([OData API](https://docs.thinkwisesoftware.com/docs/indicium/api)).

**6. Deletes are the classic gap.** A timestamp filter finds inserts and updates. It cannot find deletes or de-listings. Cherry needs either a soft-delete/status flag it can filter on, or a periodic full key-set reconciliation.

**7. The platform's own client polls.** The most telling evidence that no push/delta channel exists: auto-refresh in the Universal GUI is *"possible to refresh the subjects and variants automatically every x seconds"*, with the warning *"Performance can deteriorate if the screen is refreshed too often"* ([Settings for subjects](https://docs.thinkwisesoftware.com/docs/sf/subjects_settings)). If a delta feed existed, Thinkwise's own GUI would use it.

**Net:** the delta mechanism has to be designed into the ERP API. The cheapest robust shape is a `modified_since` parameter on each read endpoint backed by an indexed UTC timestamp (or `ROWVERSION`) maintained by **triggers**, plus soft-delete flags, plus a low-frequency full reconciliation — optionally gated by a cheap change-detection call.

---

## What's natively cheap

- **OData v4 read access to any table or view** in the model, with `$filter`, `$select`, `$expand` (lookups *and* 1:N details), `$orderby`, `$top`/`$skip`, `$count`, `$search`. No per-table modelling work — exposure is an IAM authorization exercise.
- **A machine-readable contract for free**: `/$metadata` (CSDL), `/openapi` (filterable OpenAPI JSON), `/application.svc`.
- **Server-side aggregation** via `$apply` with `groupby`/`aggregate`/`compute`, chainable — Cherry can ask for totals without pulling rows.
- **Server-driven paging** via `@odata.nextLink`.
- **Named prefilters** (`$prefilter`) and **table variants** (`/entity.variant`) as ready-made, purpose-shaped projections that need no new view.
- **Lookup resolution by natural key** (`choose_by_display` / `choose_by_element`) — Cherry can post business keys, not surrogate IDs.
- **OAuth2 client-credentials machine-to-machine auth**, JWT bearer, bound to a service account. Configuration only.
- **Exposing an existing stored procedure or SQL function as an endpoint**: one checkbox, plus an alias to give it a sane public name.
- **`/health`** endpoint and **`Server-Timing`** response headers for monitoring and latency measurement.
- **Change detection** as a cheap "has this subject changed since T?" probe.
- **Server-side export** (`$export` to CSV/XLSX) and an **Import API** for file-shaped bulk.
- **Outbound HTTP calls** from a process/system flow via the Web/HTTP connector (bearer/API-key/basic auth, configurable timeout) — the plumbing exists; only the triggering logic is custom.
- **An arbitrary inbound endpoint** for Cherry → ERP calls via a Custom protocol process flow at `/indicium/open/...`, with any method, body and response shape.

## What's custom work for the Thinkwise team

- **Any per-customer, per-branch computed price exposed as a filterable, pageable column.** Table-valued functions as tables are *"not yet available"*; expressions don't exist outside the select path; function-backed computed columns are explicitly warned against for filtering. Either a purpose-built function endpoint (unfilterable) or an ERP-maintained materialised price table (the good option) must be built.
- **Any delta/`modified_since` capability**: trace/timestamp columns are a Thinkstore add-on, not a framework guarantee, and must be filled by **triggers** (not defaults) to survive direct-to-database writes. Plus indexing, soft-delete flags, and a reconciliation endpoint.
- **Any outbound push to Cherry.** There is no subscription registry. The connector plumbing is native, but the event detection is custom: DB trigger → outbox table → scheduled system flow → HTTP POST. Thinkwise ships a Thinkstore "Standard http queue with system flow (API)" template for exactly this, but it still has to be wired to Cherry's events. Expect tens-of-seconds latency, and build retry/idempotency yourself — connector retry and dead-lettering are undocumented.
- **Surfacing history**, if temporal tables are the chosen delta source: history tables and their `tsf_valid_from`/`tsf_valid_to` columns are not model objects and are not OData-visible without generated views.
- **A bulk/batch read endpoint** shaped for "prices and stock for N products for customer C". `$batch` is unsupported and function results can't be paged, so the shape has to be designed deliberately.
- **Order submission as a single atomic operation.** Table endpoints insert one record per request and run layout/default logic per field. A header-plus-lines order needs a purpose-built task or procedure with atomic-transaction semantics.
- **An availability/stock projection** that doesn't require Cherry to understand ERP stock internals.
- **Response-size-limit exemptions** configured per endpoint if any endpoint must return more than 1000 rows.
- **Service-account configuration**: client application, secret, role/prefilter set, and **Exclude from max. # sessions**.

## Constraints this places on the Cherry ERP API contract

Written so a non-Thinkwise reader can act on them.

1. **Ask for purpose-built endpoints, not table access.** Raw OData over ERP tables is free but couples Cherry to the ERP's internal schema and makes Cherry re-implement pricing. Specify a small number of endpoints shaped around Cherry's questions ("prices for customer C at branch B", "availability for these SKUs"), not around ERP tables.
2. **Every list endpoint must be pageable and must state a page size.** Indicium caps table responses at **1000 records** from platform 2026.1 and requires clients to follow `@odata.nextLink`. Cherry's client must implement nextLink following from day one. Do not write a contract that assumes a single-response full catalogue.
3. **Do not put the computed price in a `$filter`.** Thinkwise's own docs strongly advise against filtering on function-derived columns because the function is evaluated for every row. Filter and sort on stored, indexed columns; return the price as a payload field.
4. **Prefer an ERP-maintained price table over an on-the-fly computed price.** Ask for pricing logic to write a materialised customer+branch+product price table with an indexed `modified_at`. This is the only option that behaves like a normal column — filterable, pageable, indexable, delta-able — and it keeps one implementation of pricing inside the ERP.
5. **Specify a `modified_since` (UTC) parameter on every read endpoint**, backed by an indexed timestamp — and **require that it is maintained by database triggers, not by default logic.** This is not pedantry: Thinkwise's trace-field solution ships in both variants, and the defaults variant does not fire for writes that reach SQL Server directly (batch jobs, interfaces, scripts). Getting this wrong produces a sync that silently misses exactly the updates a wholesaler cares about. Ask whether a `ROWVERSION` high-water mark can be exposed instead — it is monotonic and immune to clock skew and same-millisecond ties. Also note Indicium has no `now()`, so Cherry always sends literal timestamps; pin down inclusive/exclusive boundary semantics.
6. **Specify how deletions and de-listings are communicated.** A timestamp filter cannot express a delete. Require soft-delete/status flags, and agree a periodic full key-set reconciliation to catch drift.
7. **Assume pull. Treat any push as a bonus, and never as a correctness guarantee.** Push is buildable — connectors exist and Thinkwise ships an HTTP-queue template — but process flows do **not** fire on direct-to-database writes, so a push design that isn't trigger-backed will miss events. If near-real-time is wanted, specify: DB trigger → outbox table → scheduled system flow → POST to Cherry, with **idempotency keys** and at-least-once semantics (connector retry/dead-lettering is undocumented, so duplicates and drops are both on the table). Expect tens-of-seconds latency, not sub-second. Cherry's poll must remain the source of truth even if push is delivered.
8. **Do not design synchronous live ERP lookups into page render.** No latency guarantees exist, cold start is documented as multi-second, and every client-application change restarts Indicium. Cherry should render from its own mirror and, at most, do a live re-check at a single high-stakes moment (add-to-cart or checkout) with a short timeout and a defined stale-data fallback. Cherry mirrors; it never blocks on the ERP.
9. **Authenticate as one broad service account and pass customer identity as a parameter.** Client credentials binds to exactly one IAM user with one fixed prefilter context; per-user delegation needs an interactive login and user simulation is a debug tool. Customer scoping is therefore Cherry's responsibility — which means the contract must include customer identity in the request, and Cherry must be trusted to enforce it.
10. **Require `Exclude from max. # sessions` on the service account**, or a busy shop will hit the concurrent-session ceiling.
11. **Make order submission one call.** Specify a single task/procedure endpoint taking header plus lines, running as an atomic transaction, returning the ERP order reference. Do not accept a contract where Cherry inserts an order header then loops inserting lines — that is one request per line, each running per-field layout and default logic.
12. **Require the API spec as an artifact.** Indicium emits OpenAPI at `/openapi` and CSDL at `/$metadata`. Make the filtered OpenAPI document the contract deliverable and pin it in Cherry's repo, so drift is visible.
13. **Require ERP errors to be machine-readable.** Indicium returns failures in a base64-encoded `TSFMessages` header with a stable `MessageID`. Specify that Cherry keys on `MessageID`, not on translated text.
14. **Get `Server-Timing` enabled and agree a latency budget before committing to any synchronous path.** It is the only way to answer the performance question, because Thinkwise publishes no numbers.

## Open questions for the Thinkwise team

Public docs cannot answer these; Aartsen must ask directly.

1. **Which platform version is the ERP on**, and is it at/above 2026.1 (i.e. is the 1000-record cap already live)? What is the upgrade cadence?
2. **What is the deployment topology** — IIS on-prem, containers, Thinkwise Cloud? Single instance or horizontally scaled? Sticky sessions or Redis? Is there a non-production environment Cherry can integrate against?
3. **Do the relevant ERP tables (products, prices, stock) carry trace/timestamp columns — and are they filled by triggers or by default logic?** Are they UTC? Is system versioning switched on for any of them? This single answer determines whether a `modified_since` contract is a configuration change or a project.
4. **Do any writes to products, prices or stock bypass Indicium** — batch jobs, interfaces, SSIS, direct SQL? If yes, process-flow-based push is off the table and trigger-based trace columns become mandatory.
5. **How is a customer+branch price actually computed today** — a stored table, a view, a stored procedure, or logic embedded in order entry? This decides whether constraint #4 is a small ask or a large one.
6. **Does a service account consume a named-user licence?** What is the licensed concurrency, and is there a per-call or per-seat cost to a machine consumer?
7. **What is an acceptable request rate** against the production ERP database, and are there existing batch windows where an additional read load would be harmful?
8. **Is there an existing outbox, integration or messaging layer** in the ERP already, or an MQTT broker in use? Is the platform on 2026.2+ (message brokers) at all? If so, an event-driven path may be cheaper than it looks from the docs.
9. **Can Thinkwise commit to a latency target** for the proposed endpoints, measured via `Server-Timing`? What happens under concurrency?
10. **Is stock a single number**, or per-branch, per-batch, with reservations/allocations? What does "available" mean in ERP terms, and is there an existing definition Cherry should mirror rather than invent? (Feeds ticket #5.)
11. **Who owns the API contract's versioning and change process** once live — what notice does Aartsen get before a breaking ERP model change?

## Sources

Thinkwise official documentation (docs.thinkwisesoftware.com):

- https://docs.thinkwisesoftware.com/docs/indicium/api
- https://docs.thinkwisesoftware.com/docs/indicium/indicium_general
- https://docs.thinkwisesoftware.com/docs/indicium/endpoints
- https://docs.thinkwisesoftware.com/docs/indicium/resource_staging
- https://docs.thinkwisesoftware.com/docs/indicium/process_flows
- https://docs.thinkwisesoftware.com/docs/indicium/importapi
- https://docs.thinkwisesoftware.com/docs/indicium/multi-row_tasks
- https://docs.thinkwisesoftware.com/docs/indicium/indicium_troubleshooting
- https://docs.thinkwisesoftware.com/docs/sf/data_model
- https://docs.thinkwisesoftware.com/docs/sf/business_logic
- https://docs.thinkwisesoftware.com/docs/sf/subroutines
- https://docs.thinkwisesoftware.com/docs/sf/subjects
- https://docs.thinkwisesoftware.com/docs/sf/subjects_columns
- https://docs.thinkwisesoftware.com/docs/sf/subjects_settings
- https://docs.thinkwisesoftware.com/docs/sf/variants
- https://docs.thinkwisesoftware.com/docs/sf/roles
- https://docs.thinkwisesoftware.com/docs/sf/generic_features
- https://docs.thinkwisesoftware.com/docs/sf/message_brokers
- https://docs.thinkwisesoftware.com/docs/sf/process_flows
- https://docs.thinkwisesoftware.com/docs/sf/process_flows_actions
- https://docs.thinkwisesoftware.com/docs/sf/process_flows_connectors
- https://docs.thinkwisesoftware.com/docs/sf/process_flows_oauth_connectors
- https://docs.thinkwisesoftware.com/docs/sf/web_connections
- https://docs.thinkwisesoftware.com/docs/sf/oauth_servers
- https://docs.thinkwisesoftware.com/docs/sf/dynamic_model
- https://docs.thinkwisesoftware.com/docs/overview/platform_overview
- https://docs.thinkwisesoftware.com/docs/overview/integrations_functional
- https://docs.thinkwisesoftware.com/docs/iam/client_apps
- https://docs.thinkwisesoftware.com/docs/iam/client_apps_certificates
- https://docs.thinkwisesoftware.com/docs/iam/users
- https://docs.thinkwisesoftware.com/docs/iam/openid
- https://docs.thinkwisesoftware.com/docs/iam/openid_providers
- https://docs.thinkwisesoftware.com/docs/iam/user_simulation
- https://docs.thinkwisesoftware.com/docs/iam/system_flows
- https://docs.thinkwisesoftware.com/docs/kb/performance
- https://docs.thinkwisesoftware.com/docs/kb/lifecycle_policy
- https://docs.thinkwisesoftware.com/docs/deployment/indicium
- https://docs.thinkwisesoftware.com/docs/deployment/indicium_configuration
- https://docs.thinkwisesoftware.com/docs/deployment/scalability
- https://docs.thinkwisesoftware.com/docs/deployment/scaling-indicium
- https://docs.thinkwisesoftware.com/docs/deployment/container_registry_indicium
- https://docs.thinkwisesoftware.com/docs/deployment/container_deployment_overview
- https://docs.thinkwisesoftware.com/docs/deployment/container_deployment_aks
- https://docs.thinkwisesoftware.com/blog/2026_1 (2026.1 release notes)
- https://docs.thinkwisesoftware.com/blog/2026_2 (2026.2 release notes)
- https://docs.thinkwisesoftware.com/blog/2023_1 (2023.1 release notes — temporal versioning)

Thinkwise Community — posts by identified Thinkwise staff/moderators, or official product updates:

- https://community.thinkwisesoftware.com/questions-conversations-78/indicium-api-performance-large-table-transfer-6012 (Vincent Doppenberg)
- https://community.thinkwisesoftware.com/questions-conversations-78/authentication-method-indicium-universal-for-external-applications-2396 (Vincent Doppenberg)
- https://community.thinkwisesoftware.com/questions-conversations-78/triggers-vs-process-flows-1676 (Vincent Doppenberg)
- https://community.thinkwisesoftware.com/news-blogs-21/indicium-vs-indicium-universal-445
- https://community.thinkwisesoftware.com/questions-conversations-78/odata-pagination-3556 (Mark Jongeling)
- https://community.thinkwisesoftware.com/questions-conversations-78/trace-your-changes-by-adding-trace-fields-1107 (Frank Wijnhout)
- https://community.thinkwisesoftware.com/questions-conversations-78/make-your-logging-data-available-in-the-gui-1154
- https://community.thinkwisesoftware.com/product-updates/thinkstore-model-updates-2025-1-5790
- https://community.thinkwisesoftware.com/product-updates/release-notes-thinkstore-2024-1-4730
- https://community.thinkwisesoftware.com/product-updates/thinkwise-platform-2026-2-6842
- https://community.thinkwisesoftware.com/news-blogs-21/case-a-message-protocol-independent-web-service-part-1-of-2-1947
