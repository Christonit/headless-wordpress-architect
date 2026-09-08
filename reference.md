# Headless WordPress — contracts and guards

Load this when implementing sync, catalogs, SEO, or third-party embeds.
Architecture options are suggestions; safety requirements are mandatory.

## Evidence protocol

Before claiming a bottleneck or architecture defect, capture at least one:

- Repository path and relevant code
- Request trace or response-time distribution
- Build duration and page count
- Cache headers / cache behavior
- Runtime log, queue metric, or failed event
- Rendered HTML or Core Web Vitals trace

Label everything else as an assumption. Never invent repository evidence.

For REST vs GraphQL, compare equivalent fields under the same cache state,
hosting, authentication, and concurrency. Report p50/p95 (or the available
distribution), payload bytes, query shape, and sample size.

## Safety requirements

- Authenticate inbound mutation/webhook endpoints.
- Validate event and API payloads at trust boundaries.
- Keep API credentials and privileged WordPress access server-side.
- Do not expose draft, private, pending, or scheduled content publicly.
- Sanitize or allowlist untrusted HTML and third-party scripts.
- Make writes, retries, and deletes idempotent.
- Prevent older revisions from overwriting newer content.
- Preserve an audit trail sufficient to repair partial publication.

## Per-slug JSON (suggested minimum)

Flattened strings, not `{ rendered }` objects:

- id, type, slug, status
- dates as UTC ISO (parse naive WordPress datetimes in the **editorial
  timezone**, not the server timezone)
- title, parsed `content`, excerpt if needed
- featured image URL (strip an existing image-proxy prefix before applying
  yours)
- author object from the authors dictionary
- taxonomy ids/slugs the template uses
- SEO snapshot (title, description, OG, canonical, schema string)
- derived fields computed at write (reading time, referenced entities)

Omit unused embedded objects, link maps, and plugin keys the frontend never
reads.

## Listing catalog

Prefer a small queryable row per published item: partition by source + CMS id,
sort by publication date, feed key for newest-first pagination.

If the sort key **is** the date, a republish with a new date can duplicate
rows — delete old rows for that id before upsert.

List APIs can return empty `content`. Existence-check storage before linking if
catalog and blobs can drift.

## Index shards (medium sites without a catalog database)

Paginated files: routes plus a tiny dictionary (id, type, tags). Full HTML
never lives here.

A webhook may only patch the first shard; a periodic full sync may be needed
to reconcile the rest.

Route keys often come from the public path on `link`, not the bare slug.

## Webhook payload

Accept several shapes (id at root, nested `post`, nested `post_data`, JSON
strings). Extract id, type, status, slug. Log a summary, not full post HTML.

Unsupported type → acknowledge and skip.
Non-publish after REST → no public write.
Delete → storage + catalog, idempotent.

Do not return success before durable acceptance when event loss is outside the
freshness contract. An in-memory queue loses accepted work on process failure.

Queue delivery may be duplicate or out of order. Store a source version
(`modified_gmt`, revision, or monotonic sequence) and condition every write so
an older operation cannot replace a newer snapshot.

For multi-step publication, use an outbox/state record or equivalent:

```text
accepted → document-written → indexes-updated → obsolete-route-removed → done
```

Retry from the last incomplete step. Reconciliation finds records stuck between
states.

## REST harvesting

- Walk total-page headers; cap page size (often 100).
- Minimum delay between calls; backoff on 429/502/504.
- Re-fetch after webhook with no-cache / cache-bust query.
- Custom types: expose in REST and keep the core post JSON shape when possible
  so one parser works.

## Self-heal

Miss → REST by slug → **filter exact slug** → store via the same normalizer →
read back → serve. A fuzzy first hit is worse than a 404.

## Hydration and data loading

- Request-aware fetch on the server when cookies or A/B variants matter.
- Pin volatile display values (current month/year) in shared SSR state.
- Raise fatal errors at page setup so HTTP status matches the error page.
- Isolate client widgets whose DOM depends on scroll or late-loaded scripts.

## SEO checks

- CMS domain should not appear in canonical, OG, or JSON-LD on the public site.
- Avoid emitting both `Z` and a numeric offset on the same datetime string.
- One H1 on the template; demote duplicate H1s in body if needed.
- Validate against rendered HTML.

## Parser idempotency

Re-storing the same HTML must not double-apply link wraps, placeholders, or
URL rewrites.

## Third-party libraries

Allowlist scripts (audio players, voice APIs, chart widgets). Strip unknown
scripts from stored HTML. One loader module; one in-flight promise per library
URL.

## Recovery

Prefer re-triggering the same save pipeline (touch/save in WordPress, or
backfill through the single-item processor) over a second JSON shape for
scripts.

## Related reading

- [WPGraphQL performance guidance](https://www.wpgraphql.com/docs/performance)
- [WordPress REST posts endpoint](https://developer.wordpress.org/rest-api/reference/posts/)
- [At-least-once queue delivery example](https://developers.cloudflare.com/queues/reference/delivery-guarantees/)
- [Earlier Core Web Vitals case study](https://dev.to/christonit/how-to-improve-page-load-times-and-web-core-vitals-in-nuxt-websites-j3l)

The case study is historical evidence for the performance problem and reported
outcome. Its statement that REST cannot query by slug and its
`await setTimeout(...)` example are superseded by this skill and current
WordPress documentation. Do not copy those two implementation details.
