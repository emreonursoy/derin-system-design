# ADR 0001: Postgres only, with a narrow write table feeding a read table

**Status:** Proposed, for my review. **Inputs:** Q3, Q4, Q7, Q11, Q12, Q15, Q17, Q18, Q19, Q21, Q26, Q27, Q32, Q34. **Open:** Q37.

All figures come from docs/assumptions.md. Sizing inputs are ASSUMED, and nothing is benchmarked.

## Context
- A collection holds up to 500k responses x 80 questions.
- Targets (ASSUMED): rollup reads under 1s p95, ad-hoc filters under 3s, data at most 10s old.
- Writes (DERIVED from ASSUMED pace): 500 answers/s average, 1,500/s burst.
- Dashboard (DERIVED): 60 live polls/s, at most 10 ad-hoc queries/s.
- Filters combine freely, so pre-computing every combination is impossible.
- A team this small should run as few data systems as it can. I start with Postgres only, and name what would end that.

## The comparison

**Option A: one row per response.** Each answer updates a compact row holding all 80 coded answers.

**Option B: narrow write table feeding a read table.** Each answer inserts one row into an append-only table. A projector then updates a read table with one row per response.

| Criterion | A: one row per response | B: narrow write plus read table |
| --- | --- | --- |
| Save an answer | UPDATE rewrites about 1.16 KB (ASSUMED) | INSERT of about 100 B (ASSUMED) |
| Retry and out-of-order safety | A version per question inside the row can reject stale saves, adding about 160 B (ASSUMED). No history. | Unique key on the client's answer ID rejects duplicates, and keeps history |
| Dead rows in the participant path | 80 versions per response, in the table participants write to | None; the churn moves to the read table |
| Dashboard load can slow saves | Yes, same table and same vacuum | Less, but they still share a server |
| Filter scan | About 830 MB per collection (DERIVED) | About 714 MB (DERIVED) |
| Total data written | See the next section | See the next section |
| Final storage per collection | About 830 MB at fill factor 70 (DERIVED) | 8.7 GB (DERIVED): read table about 714 MB, plus write table and two indexes |
| Freshness | Immediate on the primary | Adds up to 3s for the projector (ASSUMED) |
| Parts to run | One table | Two tables plus a projector that can fail or lag |
| Change the encoding, or repair a bug | Risky migration of the only copy | Rebuild the read table from the write table |
| Withdrawal | Delete one row, then fix rollups | Delete from both, then fix rollups |

## Total data written, including rewrites
This table uses WAL per collection, because every byte also reaches the replica. Sizes are ASSUMED and results DERIVED (arithmetic in assumptions.md).

"Saves per answer" counts every server write for one final answer, including autosave. "High" means an updated row moves to a new page. "Low" means it stays on the same page, where Postgres can log only the changed part. In B, drafts stay in the write table and only the final answer reaches the read table.

| Saves per answer | A, high | B, high | A, low | B, low |
| --- | --- | --- | --- | --- |
| 1 (no server autosave) | 48.4 GB | **50 GB** | 8 GB | **16 GB** |
| 1.5 | 72.6 GB | 54 GB | 12 GB | 20 GB |
| 3 | 145.2 GB | 66 GB | 24 GB | 32 GB |
| 5 | 242 GB | 82 GB | 40 GB | 48 GB |

- **With no server autosave, B writes slightly more:** 1.6 GB in the high case, and 8 GB in the low case. That is at most 0.1 MB/s on average and 0.3 MB/s at burst.
- **With server autosave, B can win.** In the high case it breaks even at 1.04 saves per answer. In the low case it never does, and stays 8 GB higher.
- **Dead rows are equal at one save per answer,** at 40M either way. The difference is where they sit: in A they share a table with participant saves. Above one save, A's grow, and B's read table stays at 40M.
- **Batching does not help.** A 2s batch holds about 0.3 answers per response at burst, so it reduces commits, not rewrites.
- **Not measured:** full-page images, vacuum, heap page writes, rollups, transcripts and the search index. The benchmark must report `pg_stat_wal`.

**Does the recommendation hold? Yes, and data volume doesn't decide it.** The two are close. I recommend B for three reasons:
- A participant's save stays a small idempotent insert, even if the projector stops.
- The read table can be rebuilt after a bug or an encoding change.
- B gives server autosave a cheap place to land if we ever need it.

The cost of B is a component that can fail, which the projector design below addresses.

## Decision
Use option B, in Postgres on Azure Database for PostgreSQL flexible server.

| Part | Decision |
| --- | --- |
| Write table | Append-only. One row per answer. Unique key on collection ID, response ID and client answer ID, with the response ID taken from the token. This scopes each answer to its participant. Time-ordered response IDs lead the key, because random IDs scatter index writes, which is an added cost otherwise (up to about 12 MB/s of WAL just after a checkpoint, DERIVED). A second index on transaction ID serves the projector. This is the source of truth. |
| Access | Row-level security on collection ID. Separate database roles for interview, projector, query, redaction and export. |
| Autosave | Stays on the phone, so one save per answer is the design case. Only final answers reach the server. |
| Read table | One row per response. Coded answers sit in one compact column. Only screener fields such as age, gender and location are indexed, so most updates can avoid index writes. Fill factor 70 (ASSUMED) leaves room for same-page updates. |
| Partitions | Each large collection gets its own partition in both tables. Small collections share a set of hash partitions. |
| Encoding | Coded values per question, with a bitmask for multiple choice and a coded bin plus raw value for numeric. The exact type (typed array or JSONB) is settled by the benchmark. |
| Free text | Transcripts live in their own table. Never in the read row. |
| Changed answers | A later answer to the same question replaces the earlier one in the read table. Order within a device follows its sequence number. Across devices, last received wins (ASSUMED rare). History stays in the write table. |
| Rollups | Counters by question, option or bin, and state (all or completed). Numeric questions also keep count and sum. See ADR 0004. Updated in the same transaction as the read table. |
| Pooling | PgBouncer in transaction mode. |

Search uses Postgres full-text search, since pgvector is declined (Q15). Language coverage is a known weakness, including Arabic (ASSUMED, not tested).

### Projector
A NestJS worker on the primary. Rejected: a per-answer Inngest event, which at 500/s would hand a vendor a per-participant timeline, and add load Inngest isn't sized for here.

| Point | Decision |
| --- | --- |
| Finding new rows | Each write row stores `pg_current_xact_id()`. The projector reads only rows below `pg_snapshot_xmin(pg_current_snapshot())`, so it never passes a transaction that hasn't committed yet. "ID greater than last seen" would silently skip late commits. |
| Exactly once | The cursor, read rows, rollups and data version commit in one transaction. A crash replays the whole batch or none of it. |
| One owner | A Postgres advisory lock, on a direct connection, because PgBouncer's transaction mode can't hold a session lock. A standby worker takes over if the holder dies. |
| Pace | A batch at most every 2s, running in up to 1s at burst (both ASSUMED). |
| Long transactions | One long transaction on the primary would hold the projector back. Saves are short. Rollup rebuilds run in small batches, for this reason. We alert on watermark age. |
| Rebuild | Bulk load, not through the projector, which would take about 3.7 hours per large collection (DERIVED). |
| Reconciliation | An Inngest job, carrying IDs only, compares counts per question between write table, read table and rollups. A mismatch alerts, and can trigger a rebuild. |
| If it stops | Saves continue. The dashboard goes stale and says so, using the "as of" time. |

### Read path
| Part | Decision |
| --- | --- |
| Live rollups | After each batch, the projector writes suppressed rollups to Redis. Dashboards read only Redis, so polls never touch Postgres. |
| Live polling | Dashboards poll every 5s (Q32). An unchanged version returns an empty response, which saves little during active collection, when nearly every poll returns a body. |
| Ad-hoc queries | On in-region replicas, in one repeatable-read transaction: read the data version, then run the query. Both come from the same snapshot, so a lagging replica can't store old data under a new version. |
| Query cache | Azure Managed Redis in the regional stack. Key: collection, keyed hash of the filter, chart set, and a 30s bucket of the data version. Holds results only after suppression (Q27). Expires after 60s (ASSUMED). |
| Refresh of open filtered views | Every 30s (ASSUMED), with a visible "as of" time showing the data's true age, and a rate-limited refresh button (Q37). |
| Limits | 10s statement timeout. 2 concurrent ad-hoc queries per user and 5 per organization (ASSUMED), counted in Redis. Beyond that, requests queue, then get a "busy" response. |
| Suppression | One shared query layer applies the minimum of 10 and the Q21 controls to every aggregate, including chat and summaries (ADR 0004). Transcripts and row-level exports are separate, permission-gated paths (Q20). |

### Replica settings
| Setting | Decision |
| --- | --- |
| `hot_standby_feedback` | On. Microsoft warns it can bloat the primary, but the 10s statement timeout limits how long cleanup is held back. We alert on bloat. |
| Replica count | 3, at 32 vCores each (ASSUMED). 70 cores with no cache, including chat, / 32 per replica, rounded up, so a Redis outage doesn't overload them. Losing a replica during that outage would. |
| Lag | The UI shows "as of". An alert fires when a replica is over 4s behind, which is the budget for a cold ad-hoc query. |

### Freshness budget
From server commit, at normal operation. Inputs ASSUMED, results DERIVED. These assume p95 queries and a running projector, so they aren't absolute limits.

| View | Arithmetic | Age |
| --- | --- | --- |
| Live rollups | 2s batch + 1s run + 5s poll | About 8s, within 10s |
| Cold ad-hoc query | 2s + 1s + 3s query = 6s, leaving 4s for replica lag | Within 10s if lag is under 4s. Microsoft says lag can reach minutes, so 4s is a target, not a promise. |
| Cached filtered result | 7s to reach the replica + up to 30s bucket | About 37s (Q37) |
| Open filtered view, just before refresh | 37s + 30s on screen | **About 67s**, over the target (Q37) |
| Without Redis | 2s + 1s + lag + 5s read cycle + 5s poll | 13s plus lag, over the target |
| Small segments | Update only after 10 new responses | Exempt (Q21) |

## Benchmark and columnar trigger
Run on synthetic data: 3 collections of 500k x 80, 1,500 answers/s, 60 polls/s, 10 ad-hoc queries/s and 7.5 chat tool queries/s, with worst-case filters and cross-tabs, on replicas with the cache off. Cache-off is the honest case, because the hit rate is unproven. Report separately for 8 and 80 charts per query: 80 charts over 500k rows is 40M counter updates, which may far exceed 1s. Also report the share of same-page updates, because hot rows may fill their pages' spare room within seconds, making the high WAL case the likely one.

Escalate if any of these holds after tuning, on 3 replicas of 32 vCores (ASSUMED budget):

| Measure | Trigger |
| --- | --- |
| Query cost | Mean above 5 core-seconds. Replicas saturate here (96 / 17.5 queries/s, DERIVED), well before the 3s trigger. |
| Ad-hoc filter or cross-tab | p95 at or above 3s |
| Live rollup age, excluding exempt small segments | Above 10s |
| Participant answer save, measured at the server | p95 above 500 ms (ASSUMED) |

Rollup reads come from Redis, so they aren't a Postgres trigger. Rollup age or save misses point to the primary or projector, where replicas can't help. Query misses escalate in this order, stopping at the first step that passes:
1. Add replicas, up to the limit of five.
2. Move the read table to its own Postgres server, so dashboard load and its WAL leave the participant server.
3. Add a columnar store fed from the write table. This needs my approval, and its own residency check.

## Privacy and residency
| Component | Impact |
| --- | --- |
| Write and read tables | Hold pseudonymous health answers, in the collection's regional stack only. No direct identifiers. |
| Replicas | Same region, so no new residency exposure. |
| Redis | Holds suppressed aggregates, counters and opaque IDs only (Q34), never answers or transcripts. In the regional stack, with no geo-replication. Safe to share across roles because the minimum of 10 applies to all roles. Scoped to the collection. Filter hashes are keyed, because screener values alone are easy to guess. |
| Backups | Stay within the geography (ADR 0002). Withdrawn data remains in backups until they age out. |
| Withdrawal | See below. |

### Withdrawal
A participant starts this with their withdrawal code (ADR 0003).

1. Add the pseudonymous response ID to a tombstone table. The write path rejects anything for that ID, so an offline phone can't re-create the response later.
2. Delete from the write table, read table, transcripts, redacted transcripts and turn records.
3. Remove that response's quotes from chat history, and delete undownloaded export files containing it (ADR 0004).
4. Adjust rollups for that collection through a job carrying a batch ID, so a vendor can't see which response withdrew.
5. Regenerate the summary (ADR 0005).
6. Exports already downloaded can't be recalled (Q18).

## Consequences and risks
- Microsoft warns that replica lag can grow under sustained heavy writes. This now affects ad-hoc queries only, since live rollups no longer use a replica.
- Core demand is 70 with no cache (DERIVED from ASSUMED), which fits 3 replicas of 32 vCores only if the 1s cold scan holds. That scan is the most important number to benchmark. Query rates are averages with no burst.
- Open filtered views can show data about 67s old, which is over the 10s target (Q37).
- The projector and watermark are new failure points, covered by reconciliation and alerts.
- Multiple choice adds write rows beyond 40M. Not yet sized.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| A: one row per response | Close on data volume, and simpler. Rejected because it mixes participant saves with dead rows and vacuum, and leaves no history to rebuild from. It is the fallback if the projector proves unreliable. |
| Projector cursor on "ID greater than last seen" | Silently skips transactions that commit late. |
| Projected marker on each write row | Turns an append-only table into a rewritten one. |
| Logical decoding | Handles ordering well, but a stalled slot holds WAL on the primary. Microsoft warns storage pressure can force the server read-only. |
| Triggers on the write table | Puts read table work back into the participant's save. |
| Pushing live updates over a stream | Needs fan-out across instances. Polling is simpler, and reads only Redis. |
| Refreshing filtered views every 10s | 30 queries/s x 4 core-seconds = 120 cores (DERIVED from ASSUMED), about four replicas. |
| Query the narrow table directly | Filtering on several questions needs joins across up to 40M rows per collection, and misses the 3s target (ASSUMED). |
| One column per question | Schema changes per collection, and many sparse columns. |
| Materialized views | Full refreshes cannot meet the 10s target at this size (ASSUMED). |
| Columnar store now | Adds a second system before any number shows the need. Allowed later by the trigger above. |
| Elastic Clusters (Citus) | Built for scale-out. The read table is about 2.1 GB across three collections (DERIVED), so one server should fit. |
