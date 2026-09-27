# Cacheable Endpoints

This documents which `GET` endpoints a CDN or shared proxy in front of this
API may cache, for how long, and which ones must always be forwarded to the
origin. The guidance below matches the `Cache-Control` headers the service
actually sets — see the source references under each endpoint.

## Safe to cache publicly

| Endpoint | Header | Notes |
| --- | --- | --- |
| `GET /streams` | `Cache-Control: public, max-age=30` | Paginated stream list. Set in `src/routes/streams.ts`. |
| `GET /streams/summary` | `Cache-Control: public, max-age=30` | Aggregate counts/totals by status. Set in `src/routes/streams.ts`. |
| `GET /streams/:id` | `Cache-Control: public, max-age=30`, plus `ETag` | Set in `src/routes/streams.ts`. The `ETag` is `"<updatedLedger>"`; a request sending a matching `If-None-Match` gets `304 Not Modified` instead of a body, so a cache or client that revalidates avoids re-fetching unchanged streams even after the 30s freshness window expires. |
| `GET /streams/:id/events` | `Cache-Control: public, max-age=30` | Event history is append-only per stream, so a cached response is never wrong, only momentarily incomplete. Set in `src/routes/streams.ts`. |

All four are read-only, derive entirely from Postgres, and carry no
per-caller data (no auth, no cookies, no personalization) — the same request
returns the same response for every caller, which is what makes public
caching safe. 30 seconds bounds how stale a cached response can be relative
to the indexer's own lag; it does not need to be shorter than the indexer's
own polling interval to stay useful, since the indexer itself only advances
in multi-second increments.

## Must not be cached

| Endpoint | Header | Why |
| --- | --- | --- |
| `GET /status` | `Cache-Control: no-store` | Reports live indexer lag; a stale reading defeats its purpose. Set explicitly in `src/routes/status.ts`, on both the cache-hit and cache-miss paths of its own 2-second in-process cache. |
| `GET /health` | *(none set)* | Liveness probe; a cached `200` could mask a process that has since become unhealthy. |
| `GET /ready` | *(none set)* | Readiness probe; can legitimately flip between `200` and `503` from one request to the next as database connectivity changes. Caching either response defeats the check. |
| `GET /metrics` | *(none set)* | Prometheus scrape target; a cached snapshot would report stale counters to every scraper behind the same cache. |
| `GET /` | *(none set)* | Static-looking API index, but not marked cacheable by the service; treat as uncacheable rather than assuming it's safe by omission (see below). |
| `GET /docs`, `/docs/json`, `/docs/yaml` | *(none set)* | Swagger UI and the generated OpenAPI spec. Not marked cacheable; the spec can change between deployments of a new image. |

`/status` is the only one of these that sets an explicit directive; the rest
simply don't set `Cache-Control` at all. That absence is not the same as
permission to cache: per [RFC 7234 §4.2.2](https://www.rfc-editor.org/rfc/rfc7234#section-4.2.2),
an HTTP cache is still permitted to store and reuse a response with no
`Cache-Control` header using a heuristic freshness lifetime. Because a stale
`/health`, `/ready`, or `/metrics` response is actively misleading to a load
balancer, orchestrator, or monitoring system, a CDN or proxy placed in front
of this API should be explicitly configured to bypass its cache for these
paths rather than relying on the origin's silence to mean "don't cache."

## Summary

- Cache `GET /streams`, `GET /streams/summary`, `GET /streams/:id`, and
  `GET /streams/:id/events` for up to 30 seconds, matching their
  `Cache-Control: public, max-age=30` header; revalidate `GET /streams/:id`
  with its `ETag` if the cache supports conditional requests.
- Never cache `GET /status` — it explicitly opts out with `no-store`.
- Never cache `GET /health`, `GET /ready`, `GET /metrics`, `GET /`, or the
  `/docs` endpoints — none of them declare a `Cache-Control` header, and all
  of them either report live state or can change on redeploy.

## Related documentation

- [Understanding the Indexer Lag Figure](indexer-lag.md) — why `/status` must always reflect the current poll.
- [README: API](../README.md#api) — the existing note on `/status`'s `Cache-Control: no-store` header and the `/streams/summary` cache duration.
