# QoS oracle nest

A Nuthatch nest for **Edge & Node's gateway QoS oracle**, read from its own postings on Gnosis rather
than from its subgraph. It serves the per indexer, deployment and day figures behind Lodestar's Query
Performance chart and QoS Quality score, with no Graph API key.

```sh
nuthatch dev --dir qos-reo-nest --ipfs https://ipfs.thegraph.com/ipfs/
nuthatch sql --dir qos-reo-nest "SELECT * FROM qos_indexer_daily WHERE day = DATE '2026-09-07' ORDER BY query_count DESC"
```

Needs a nuthatch build with typed rows from IPFS documents: `[ipfs.rows]` (RFC-0037 slice 8, stacked on
nuthatch-org/nuthatch#1375).

## What the oracle publishes

Every five minutes the publisher calls `submitQoSPayload(bytes)` on the DataEdge
`0x5b4293b4c0f36cb5d4448950830bc777759b6c4f` twice, once per topic. The DataEdge emits no events; the
payload is UTF-8 JSON, `{"topic": ..., "hash": <CID>, "timestamp": <bucket end>}`, and the CID names a
JSON array of five-minute aggregates:

| Topic | One element per | Used for |
|---|---|---|
| `gateway_indexer_attempt_qos_5_minutes_prod_v3` | indexer, deployment, chain, gateway | the charts and the quality score |
| `gateway_query_result_qos_5_minutes_prod_v3` | deployment, chain, gateway | each deployment's query total (served share) |

Edge & Node posted from `0x0b8cef00f90553b9535845be6abbe3797582d424` until 2026-07-01 03:50 UTC (block 46,972,821) and from `0x8cbbe43f97f80efa6ba0a95f3d544e03f84db0ce` since 2026-07-03 14:55 UTC (block 47,014,403). Both carry the same gateway ID, `0xff4b7a5efd00ff2ec3518d4f250a27e4c29a2211`, and neither posted in the other's span. Indexer-attempt
documents run 1.5 to 1.9 MB and 2,500 to 2,900 elements; query-result documents about 0.6 MB.

`start_block` is 46,700,000 (2026-06-14 23:15 UTC), a 90-day window from 2026-09-13. The reference
subgraph starts at 24,747,400 if more history is wanted.

## Query surface

- **`qos_indexer_daily`** - per indexer per UTC day: queries, 200s, success rate, latency, blocks behind,
  fees, buckets seen, bad buckets and the worst bucket. The Query Performance chart.
- **`qos_allocation_daily`** - the same per indexer, deployment, chain and gateway. The grain the QoS
  Quality score reads.
- **`qos_deployment_daily`** - each deployment's query total from the gateway's side.
- **`qos_indexer_seconds_behind`** / **`qos_seconds_behind`** - seconds behind the freshest credible
  peer on the same deployment, per indexer and per deployment. A deployment-day with fewer than three
  credible peers has no figure, and `share_with_block_time` is the share of queries that have one.
- **`qos_indexer_attempt`** / **`qos_query_result`** - the typed five-minute rows everything above is
  built from, from a listed publisher and one document per bucket.
- **`qos_freshness`** - per topic, the newest bucket held and, separately, when the publisher last posted.
  Read from postings and stored documents, so it costs the same at 90 days as at one.
- **`qos_day_resolution`** - per day and topic, buckets posted and stored, postings whose document is not
  stored (a given-up document counts), and the day's closing block. What says a day is final.
- **`qos_posting`** / **`qos_stored_document`** - each document a listed publisher named, from its calldata,
  and each document stored, keyed by block and CID.
- **`qos_publisher`**, **`qos_rejected_documents`** - who counts as the publisher, and the documents no
  listed publisher named, counted rather than dropped.
- **`qos_indexer_attempt_rows`** / **`qos_query_result_rows`** - the stored typed rows, one per element of
  each proven document. Numbers are decimal text; the views cast them.

What each view computes is in `semantic.toml`. The arithmetic, which is where the old Lodestar card went
wrong:

- Every rate is a ratio of sums, never a mean of rates.
- Latency and blocks behind are weighted by queries. The oracle's average behaves as if it includes
  failed responses; weighting by 200s moved 99 of 374 indexer-days by more than 20%.
- A day is the UTC day of the bucket's **start**. The publisher's `timestamp` is the bucket's end, and
  dating by it files the 23:55 bucket under the next day.
- A bad bucket has at least 50 queries and under 90% success, counted per indexer across its
  allocations. A daily rate hides these: Ellipfra read 97.36% for 2026-09-06..12 while one bucket
  served 0 of 52,286.
- A bucket published twice counts once, from its first document.
- Seconds behind uses kittiwake's block-time table and its cohort rule (credible at 100 queries, a
  floor needs three credible peers). A chain with no known block time has no seconds figure.

## Gaps in the source

These are gaps in Edge & Node's data, not in this nest, and they read as gaps rather than zeros:

- From 2026-07-01 03:50 to 2026-07-03 14:55, about 59 hours with no postings, while the publisher changed address.
- From 2026-07-29, about 38 hours with no postings.
- From 2026-08-04, 37 hours or more with no postings.
- In 2026-09-06..12, two indexer-attempt buckets and five query-result buckets were never posted.

The publisher resumed from the tip each time without backfilling.

## How it is stored

- **Every document is proven** against its CID before anything is written (RFC-0037 slice 7); a document
  nothing proves writes no row.
- **Resolution completes.** Documents are fetched behind the cursor and retried until stored or given up
  on, so a gateway error delays a bucket rather than losing it (slice 6). A backfill needs no window cap.
- **Typed rows at resolution** (slice 8). Each document becomes one row per element in
  `qos_indexer_attempt_rows` or `qos_query_result_rows`, written in the same transaction as the document,
  and the raw JSON is not kept (`keep_content = false`). A document whose content does not fit the
  declared columns is refused whole and counted in `nuthatch_nest_ipfs_rows_refused_total`, never stored
  half typed. The views read columns and parse no JSON.
- **The publisher filter is live.** Call rows carry `tx_from`; `qos_published_call` keeps the listed
  publisher's posts, and `qos_rejected_documents` counts the rest.

## Still open

- **One day per statement, measured at 93 days.** kittiwake#143 asks for one day per statement inside
  nuthatch's 29 s SQL budget. Rows take their day through `qos_day_bounds`, whose text range DuckDB pushes into
  the Parquet scan, and the dedupe, the indexer-day and the seconds-behind floor read a day's rows once. On
  2026-09-15, with 66.6M indexer-attempt rows in 4,458 segments, one day of `qos_indexer_daily` took 13 to
  17 s (refused after 35 s before), `qos_indexer_seconds_behind` 11 to 12 s (60 s) and `qos_allocation_daily`
  7 to 10 s (40 s), each alone on the nest. That leaves little room when statements share it. Serve with the
  512 MB default and `NUTHATCH_SQL_MAX_CONCURRENCY` at least 2.
- **Sealing megabyte ranges** needs nuthatch with byte-bounded seal cuts (nuthatch#1391, merged 2026-09-14,
  and #1376). Without it a run sealing these ranges reached 4.1 to 4.5 GB RSS.

## Checks

`checks/parity-2026-09-07.sql` compares `qos_indexer_daily` for 2026-09-07 with figures computed straight
from that day's 288 raw documents by `scripts/parity-reference.py`, which shares no code with the views.
Counts, buckets and the worst bucket must match exactly; rates and fees to 1e-9 relative. Expect zero
rows. The check needs every document for the day resolved.

Run on 2026-09-13 from block 48,119,000 to the tip, killed with `kill -9` mid-resolution and restarted: 1,956
indexer-attempt and 1,954 query-result documents, every one verified, none given up, rejected or refused, and
one sender. For 2026-09-07, 288 documents per topic (715,970 and 315,117 typed rows), and the check returns
zero rows in 1.75 s. The store is 1.8 GB hot plus 312 MB sealed.

