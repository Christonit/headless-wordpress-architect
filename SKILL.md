---
name: headless-wordpress-architect
description: >-
  Designs and reviews scalable headless WordPress frontends using REST,
  GraphQL, static generation, SSR, content dictionaries, webhooks, persistent
  storage, and a shared content service. Use when planning content delivery,
  improving freshness or Core Web Vitals, processing WordPress HTML, deferring
  third-party embeds (audio players, charts, voice APIs), or choosing
  architecture for medium, large, and multi-site publishing platforms.
---

# Scalable Headless WordPress

Treat these guidelines as architectural **suggestions**, not universal rules.
Choose an approach after measuring:

- Content volume and publishing frequency
- Traffic and request concurrency
- WordPress response time
- Build duration and timeout limits
- Acceptable content freshness
- Runtime and storage costs
- Editorial workflow
- Deployment platform constraints

Prefer the simplest architecture that satisfies current scale, with a clear
migration path. WordPress is the CMS (write side). The frontend should not
treat WordPress as the origin for every page view unless that is still cheap
enough.

## Agent execution contract

First classify the request:

1. **Review** — inspect an existing repository or architecture.
2. **Design** — propose an architecture from supplied constraints.
3. **Implementation** — change code after understanding the existing system.

Then:

1. Inspect available evidence before recommending a pattern.
2. Record constraints, unknowns, and freshness requirements.
3. Separate **observed facts**, **assumptions**, and **recommendations**.
4. Apply only the guidance relevant to the site's scale and failure model.
5. State trade-offs and rejected alternatives.
6. Define how the result will be validated.

Never invent repository findings, traffic levels, benchmarks, plugin behavior,
or infrastructure. If evidence is unavailable, label the conclusion as an
assumption and request the missing input when it could change the design.

### Required deliverables

**Review mode**

- Scope and evidence inspected
- Current content flow
- Findings ordered by impact
- Recommended sequence of changes
- Validation plan
- Unknowns that could change the recommendation

Use this format for each finding:

```text
Finding:
Evidence: file path, supplied metric, or observed behavior
Impact:
Recommendation:
Trade-off:
Validation:
```

**Design mode**

- Constraints and assumptions
- Proposed data flow
- Content freshness target
- Alternatives considered and why they were not selected
- Failure handling and recovery
- Security boundaries
- Observability and validation plan
- Migration path from the current architecture

**Implementation mode**

- Confirm the current data contracts and runtime boundaries first.
- Make the smallest coherent change.
- Preserve backward compatibility or document the migration.
- Add tests for publication state, duplicate delivery, stale updates, slug
  changes, partial writes, and recovery where relevant.
- Report what changed, what was verified, and what remains unverified.

## Guidance strength

Distinguish two categories:

- **Adaptable architecture choices:** REST vs GraphQL, SSR vs static generation,
  storage technology, cache TTL, shared service, queue technology.
- **Safety requirements:** authenticate mutations, keep secrets server-side,
  exclude non-public content, validate untrusted HTML, make event processing
  idempotent, and handle deletion/unpublishing.

Do not present adaptable choices as dogma. Do not weaken safety requirements
into optional suggestions.

## Architecture selection

| Site | Typical signal | Suggestion |
|------|----------------|------------|
| Small / static | Infrequent publishing; full builds finish comfortably inside the deployment budget | Generate pages at **build time**. GraphQL or REST is fine if each page is requested once. Avoid hitting WordPress on every view when the generated page already has what it needs. |
| Medium | Builds and runtime CMS requests remain within measured latency, cost, and freshness targets | REST is often a practical default. Snapshot shared layout data. Incremental builds may be acceptable. |
| Large | Frequent publishing, long builds, high concurrency, multiple sites, or strict freshness targets | Incremental builds may add freshness delay and timeout risk. Consider webhooks → normalize → persist JSON → server route by slug. |

Classify with project measurements rather than universal page-count or
traffic thresholds. A slow or heavily customized WordPress install may need a
cached architecture at modest scale; a well-cached installation may safely
serve a larger workload.

### Core recommendations (adapt to scale)

- For static sites that only change once per day, consider a WordPress GraphQL
  plugin and building those pages at build time. That requests page data once
  and reduces load on WordPress.
- For medium and large sites, prefer REST. At project or build start, request
  information shared by the core layout or all pages (categories, tags, menus,
  authors).
- Create JSON dictionaries with only the information the site needs. Smaller
  payloads and less accidental exposure of unused CMS fields.
- Create a JSON dictionary (or per-document snapshot) of the basic SEO fields a
  page needs, so a plugin update is less likely to break frontend SEO
  consistency.
- Store static files in the project `public/` folder or persistent storage such
  as object storage.
- If measured traffic, latency, and origin capacity are modest, REST at request
  time can still be fast on page load and SPA navigation.
- Medium sites can often use incremental builds.
- For large sites, incremental builds can delay freshness and risk timing out.
- To keep content fresh without a build delay, consider webhooks on post
  create/update/delete. Use the event as a trigger, retrieve what the frontend
  needs, normalize, persist, and serve by slug so the frontend mostly paints.
- Prefer parsing and transforms (embeds, escaping, enrichment) in the script
  that stores the webhook payload, not in middleware or page data fetching.
- Non-critical or reactive requests and scripts: observers, mounted hooks, or
  watch effects. Prefer waiting until the window `load` event for heavy
  third-party libraries that are not required for UX.
- Move granular elements that need their own API call into their own component.
  Specify server vs client; missing that boundary causes hydration errors and
  extra CMS load.

## Central content service

For **very large sites or sister sites**, consider a separate web service for
shared operations so frontends share a **single source of truth**.

That service can own:

- Webhook ingestion
- Normalization and HTML transforms
- SEO payload generation
- Taxonomy and author sync
- Persistent content storage
- Feed / search indexes
- Redirects
- Cache invalidation
- Backfills and reconciliation
- Multi-site content contracts

Benefits: one implementation of shared transforms, consistent JSON across
frontends, independent scaling of processing, centralized retries and
observability.

Do not split out a service only for purity. It adds deployment, auth, and
operations cost. It is most useful when multiple projects repeat the same work.

## WordPress API selection

**REST** is often a good default for posts, pages, taxonomies, authors, media,
search, comments, and webhook re-fetch by ID. It is usually straightforward to
cache by URL.

**GraphQL** is useful for nested relationships, related content, and threaded
comments. Place slow queries behind a **cached server endpoint**. Do not use
GraphQL as the primary page loader on large sites unless measurements support
it.

Compare equivalent REST and GraphQL payloads under production-like cache state,
hosting, concurrency, and query complexity. Record response-time distributions
(not one request) and the fields returned. Deep GraphQL connections can add
server work, while precise field selection and smart caching can make GraphQL
competitive. See [WPGraphQL performance guidance](https://www.wpgraphql.com/docs/performance).

WordPress REST can query by slug (`?slug=`). A slug→id dictionary is still
useful for O(1) lookup, avoiding a live CMS call, and holding a normalized
route. Indexed lookup beats scanning a large array. See the official
[WordPress posts endpoint](https://developer.wordpress.org/rest-api/reference/posts/).

Avoid interpolating unsanitized slugs into GraphQL query strings; use variables.

## Shared dictionaries

At project or build start, paginate REST (`per_page` + total-page headers).
Write trimmed JSON for authors, categories, tags, menus, routes, redirects,
slug mappings, and basic SEO.

Keep fields the UI needs. Drop unused links, embedded objects, private
metadata, and plugin junk.

Host dictionaries where the runtime can read them (`public/` and bundled server
assets if the deploy target has no filesystem).

## Detail documents and listing indexes

For larger systems, consider separating:

1. **Per-slug document** — parsed HTML, SEO snapshot, resolved author, featured
   image, taxonomies the template needs. Detail pages paint this.
2. **Listing index** — title, slug, date, image, tags. **No HTML.** Feeds,
   sitemaps, related widgets.
3. **Shared dictionaries** — authors, menus, etc.

Use type-aware storage keys so posts, pages, and custom types cannot collide on
the same slug. Serve reads through a server route, not the browser talking to
WordPress on hot paths.

Suggested large-site flow:

```
WordPress save/delete
  → webhook (id, type, status)     // doorbell, not the document
  → acknowledge quickly
  → delay, then REST GET with embeds
  → normalize HTML + SEO           // once, at write
  → persist per-slug JSON
  → upsert listing row (no HTML)
Frontend
  → server route by slug → stored JSON → paint
  → lists → catalog
  → miss → optional self-heal, then 404
```

## Webhook processing

Treat the webhook as a **trigger**. Bodies are often missing SEO plugin fields,
embedded media, and custom meta, and may fire before REST is consistent.

Suggested workflow:

1. Authenticate. Acknowledge quickly **only after the event is durably
   accepted** when losing it would violate the freshness requirement.
2. Skip unsupported types with success so the CMS does not retry forever.
3. Do not store drafts or scheduled (`future`) content until publish.
4. Wait briefly after save if REST tends to be stale, then fetch by ID with
   embeds and cache-busting.
5. Normalize here; persist; update indexes.
6. On trash/unpublish/delete, remove the document **and** listing rows.
7. Dedupe bursts (`save` + `update`, bulk edit). Cap concurrency. Back off on
   429/502/504.

Use the **same normalizer** for incremental webhooks and bulk backfill.

An in-memory queue can work when occasional loss after acknowledgement is
acceptable and reconciliation repairs it. It is not durable: a crash can lose
accepted work. Use a durable queue or persist an inbox record before returning
success when the event must survive process failure.

Assume queue delivery can be duplicated and reordered unless the selected
system proves otherwise. At-least-once systems require idempotency; a queue
does not by itself guarantee that updates finish in publication order. See
[queue delivery guarantees](https://developers.cloudflare.com/queues/reference/delivery-guarantees/)
for one concrete example.

### Concurrency and partial-failure guards

**Older update finishes last**

- Attach a source revision, `modified_gmt`, or monotonic version to every job
  and stored document.
- Before writing, compare it with the stored version.
- Reject an older write; do not rely on FIFO execution alone.
- Use conditional writes or transactions where concurrent workers can race.

**Document write succeeds but listing write fails**

- Treat the operation as a resumable state machine or outbox workflow.
- Retry secondary writes with an idempotency key.
- Reconcile documents against listing/search/sitemap indexes periodically.
- Decide which layer is authoritative and make partial success observable.

**Slug changes**

- Write the new canonical key and listing row.
- Remove the old key and old listing/search entries.
- Record a redirect or canonical mapping from the previous route.
- Make retries safe if the process stops between those steps.

**Duplicate delivery**

- Derive an idempotency key from site, content type, content ID, and source
  revision/event ID.
- Ensure repeated store, index, delete, and redirect operations converge on the
  same state.

Webhook payload shapes vary. Accept a few variants; convert to one internal
event.

## Storage

Normalized JSON can live in `public/`, object storage, a key-value store, or
the content service. `public/` fits content that only changes on deploy.
Persistent storage fits webhook updates without a frontend rebuild.

## Content normalization

Prefer deterministic transforms **once at write**, not on every request:

- Flatten `{ rendered }` strings
- Decode entities; rewrite CMS URLs to public URLs (keep media CDN paths you
  intend to keep)
- Resolve authors from the authors dictionary
- Strip unused embedded REST objects after copying what you need
- Prepare embed **placeholders**; do not execute third-party scripts at write
- Image dimensions / lazy attributes; idempotent link wraps
- Schema + SEO snapshot
- Skip empty content

Runtime loaders should retrieve and paint, not re-parse large HTML.

## SEO stability

Snapshot title, description, canonical, Open Graph, and structured data into
the per-slug file at write time. Live plugin JSON can change on plugin update.

At render, rewrite CMS hostname → public hostname in canonical, OG, JSON-LD,
sitemaps, and in-body links.

Store schema as a string if nested `@graph` objects break consumers.

Indexability is an environment decision at request time so staging artifacts
are not permanently indexable.

Feeds that change all day: generate sitemaps from the listing index. Avoid a
static file in `public/` that **shadows** a dynamic sitemap route (static files
usually win).

Expand date shortcodes with values pinned once on the server so SSR and
hydration match.

Validate SEO against **rendered frontend HTML**, not only the CMS API.

## Server vs client

| Work | Suggestion |
|------|------------|
| Article JSON, SEO, core nav dictionaries | Server / SSR |
| Comments, related extras, live widgets | Isolated component; often client or cached server |
| Heavy third-party libraries | After `load` / idle / visibility; see below |

State the boundary explicitly. Wrong boundary → hydration mismatch or a
WordPress stampede.

Bridge SSR data with keyed per-route state if the framework hydrates an empty
page payload. Avoid unawaited layout-wide fetches during hydration that many
components deep-watch.

Use request-aware fetch on the server when cookies or variants matter.

Non-critical extras should not fail the page.

## External WordPress libraries and embeds

CMS HTML often pulls in libraries such as:

- Text-to-speech / article audio (e.g. BeyondWords)
- Voice APIs (e.g. ElevenLabs)
- Financial or chart widgets (e.g. Stockdio)
- Video, social, and other iframe embeds

If they are **not critical** for first paint, accessibility, or core UX,
inject them after the page has **fully loaded** (window `load`), then idle.

Suggested order:

1. Server-render a lightweight placeholder.
2. Hydrate.
3. Wait for `document.readyState === "complete"` / `load`.
4. `requestIdleCallback` with a timeout fallback (`setTimeout`).
5. Below the fold: `IntersectionObserver` (generous `rootMargin`).
6. Load each library **once**.
7. Swap placeholder when ready; clean up on unmount.

Strip or replace third-party `<script>` tags from stored HTML during
normalization. Load approved scripts through a controlled client loader.

```ts
function afterWindowLoad(callback: () => void) {
  if (document.readyState === "complete") {
    callback()
    return
  }
  window.addEventListener("load", callback, { once: true })
}

function whenIdle(callback: () => void) {
  if ("requestIdleCallback" in window) {
    window.requestIdleCallback(callback, { timeout: 3000 })
    return
  }
  window.setTimeout(callback, 0)
}

onMounted(() => {
  afterWindowLoad(() => {
    whenIdle(() => loadExternalLibraryOnce())
  })
})
```

Qualifications:

- User intent overrides deferral (click play, focus a widget).
- `preconnect` only origins you will hit soon; too many waste sockets.
- `async`/`defer` do not control library init after download.
- `async`/`await` does not make work concurrent.
- `await setTimeout(...)` does not wait; wrap the timer in a Promise.
- Keep API secrets on the server. Browser voice/chart integrations should call
  a protected server route when keys or quotas apply.
- Prefer `data-src` → `src` (or delayed iframe mount) for heavy iframes that
  would otherwise start during `DOMContentLoaded`.
- Lazy-load iframe **components** (`v-if` / dynamic import) after mount, not in
  the initial document, when they compete with LCP.

Fixed delays (1s / 3s) are last-resort heuristics. Load state, idle, visibility,
and user intent are better signals.

## Performance (Core Web Vitals)

When GraphQL, request count, or early iframes hurt FCP/TBT/LCP:

- Preconnect only to fonts and the CMS/content origin you will use immediately.
- Move leftover GraphQL (categories, tags, comments, related) to `onMounted`
  after the REST (or stored JSON) path has painted above-the-fold content.
- Reduce requests on the critical path; one prepared document per slug is
  usually cheaper than many CMS round-trips.
- Compare REST vs GraphQL with production-sized queries, not toy queries.

See [How to Improve Page Load Times and Web Core Vitals in Nuxt Websites](https://dev.to/christonit/how-to-improve-page-load-times-and-web-core-vitals-in-nuxt-websites-j3l)
for a worked example of preconnect, deferred embeds, and REST vs GraphQL.
The ideas apply to other SSR frontends, not only Nuxt. Treat it as a historical
case study: its claim that REST cannot query by slug and its
`await setTimeout(...)` example are superseded by this skill and current
WordPress documentation.

## Caching and freshness

If JSON updates in place via webhooks, long-lived cached **HTML** can stay
stale after storage has changed. Consider no shared HTML cache for frequently
edited articles, short TTL, or explicit invalidation.

Cache hashed assets aggressively. Match content cache to an explicit freshness
requirement.

## Recovery

Webhooks miss events. Consider periodic reconciliation, backfills through the
same normalizer, and optional request-time repair.

If repairing by slug, **exact-match** the slug. Fuzzy first-result lookups can
store the wrong document. Recovery should be fallback, not the happy path.

## Review checklist

- Output uses the required mode-specific deliverable.
- Findings cite evidence; assumptions are labeled.
- Architecture matches measured scale and freshness, not dogma.
- WordPress is not queried on every view unless that is still cheap.
- Shared data is trimmed; detail vs listing payloads are split.
- Webhooks re-fetch authoritative REST instead of trusting incomplete events.
- Durable acceptance matches the promised reliability.
- Event processing is idempotent and revision-aware.
- Partial writes and slug changes have recovery paths.
- Drafts, scheduled, unpublish, and delete are handled.
- Incremental and bulk use the same contract.
- Transforms are write-time and idempotent.
- SEO snapshots survive plugin changes; CMS host does not leak into canonicals.
- Non-critical WP libraries load after `load` / idle / visibility.
- Server vs client is explicit; secrets stay on the server.
- Sister sites share a content service when duplication is real.
- Changes are justified by profiling.

For contracts and operational guards, see [reference.md](reference.md).
For portfolio-grade demonstrations and an evaluation rubric, see
[examples.md](examples.md).
