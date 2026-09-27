# Audit Trail Guarantees

The `IndexedEvent` table is the service's audit trail: an append-only log of
the decoded contract events the indexer has seen. This document states
precisely what it records, what it does not guarantee, and how to query it.
For the full field-by-field reference, see [Database Schema:
IndexedEvent](database-schema.md#2-indexedevent).

## What it records

A row is written for every `created`, `withdrawn`, or `cancelled` contract
event that the indexer successfully **decoded** — regardless of what happens
next. In `Poller.applyPage` (`src/indexer/poller.ts`), the audit write happens
before the event is applied to the `Stream` table, and independently of it:

```ts
await recordIndexedEvent(indexedEventFromDecoded(event));
// ... only afterwards does the poller attempt to apply the event to Stream
outcome = await prisma.$transaction(async (tx) => {
  const res = await applyEvent(...);
  ...
});
```

This means an event's presence in `IndexedEvent` is **not** conditional on the
corresponding `Stream` mutation succeeding. An event that fails to apply — and
is therefore also recorded in `FailedEvent` — can still have an audit row in
`IndexedEvent`. The two tables answer different questions: `IndexedEvent` is
"what events did the indexer observe," `FailedEvent` is "what events did the
indexer fail to apply to current state."

Each row stores the event's `eventId` (the RPC's `TOID-index`, unique and
chronologically ordered), `kind`, `streamId`, `ledger`, `txHash`, the decoded
payload fields, and `closedAt` (the chain's own timestamp for the event, not
the database write time). Insertion uses `createMany({ skipDuplicates: true })`
keyed on `eventId`, so re-processing a page — which happens whenever the
indexer restarts before saving its cursor, or a failed event is replayed — is
idempotent: it neither creates a duplicate row nor changes an existing one.

## What it does not guarantee

**It is not guaranteed to record every event the indexer decoded.** The write
to `IndexedEvent` is best-effort: if the database insert itself fails, the
poller catches the error, logs a warning (`could not persist indexed-event
record`), and continues — it does not retry, and the failure is not recorded
anywhere else. Because this happens independently of the `Stream` update that
follows, a `Stream` row can advance to reflect an event that has no
corresponding row in `IndexedEvent`. This is the audit trail's main gap: do
not treat it as a complete, transactionally-consistent record of every state
change applied to `Stream`.

**It does not include events the indexer never fetched.** Events emitted
before the configured `INDEXER_START_LEDGER` (see [Indexer Start
Ledger](indexer-start-ledger.md)) are never requested from the RPC in the
first place, so they are absent from `IndexedEvent` entirely — not just the
`created` event, but any event for that stream before the start ledger. When a
later `withdrawn` or `cancelled` event for such a stream arrives, the indexer
reconciles `Stream` state with a `get_stream` RPC read (`orReconcile` in
`src/indexer/apply.ts`), but that reconciliation only writes to `Stream`. It
does **not** backfill the missing history into `IndexedEvent`. A stream
created before the start ledger will therefore always have an incomplete
event history — `GET /streams/:id/events` for it begins mid-lifecycle.

**It does not include events that failed to decode.** A malformed or
unparseable event never becomes a `StreamEvent`, so it never reaches
`recordIndexedEvent`. It is recorded only in `FailedEvent` (with `kind:
"malformed"`). See [Indexer Failure Policy](indexer-failure-policy.md).

**Rows are never updated after insertion.** No code path in this service
updates an existing `IndexedEvent` row; the only write is the initial
`skipDuplicates` insert. Treat rows as immutable once written, though this is
a property of the current code rather than a database constraint — nothing at
the schema level prevents an `UPDATE`.

**It carries no actor identity beyond on-chain addresses.** Rows record
`sender`, `recipient`, and `token` as they appear in the contract event; there
is no separate notion of "who queried" or "who administered" anything. It is
an on-chain event log, not a request-level access audit.

## How to query it

**Via the API** — `GET /streams/:id/events` returns a stream's full recorded
history in ascending chronological order (ordered by `eventId`, which sorts
correctly as a string because both halves of the `TOID-index` are zero-padded):

```
GET /streams/42/events
```

**Via SQL** — direct queries are useful for cross-stream or ledger-range
investigation that the API doesn't expose:

```sql
-- Full history for one stream, oldest first
SELECT "eventId", "kind", "ledger", "txHash", "closedAt"
FROM "IndexedEvent"
WHERE "streamId" = '42'
ORDER BY "eventId" ASC;

-- Every event observed in a ledger range (uses the [ledger] index)
SELECT "eventId", "kind", "streamId", "ledger"
FROM "IndexedEvent"
WHERE "ledger" BETWEEN 56000000 AND 56010000
ORDER BY "ledger" ASC, "eventId" ASC;

-- Cross-check against events that failed to apply to Stream state
SELECT ie."eventId", ie."kind", ie."streamId", ie."ledger"
FROM "IndexedEvent" ie
JOIN "FailedEvent" fe ON fe."eventId" = ie."eventId";
```

## Related documentation

- [Database Schema: IndexedEvent](database-schema.md#2-indexedevent) — complete field reference and write path.
- [Indexer Start Ledger](indexer-start-ledger.md) — why a late start ledger leaves gaps in a stream's recorded history.
- [Indexer Failure Policy](indexer-failure-policy.md) — how malformed and unapplied events are handled.
- [Event Replay](event-replay.md) — reprocessing events recorded in `FailedEvent`.
