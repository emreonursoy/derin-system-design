# ADR 0001: Postgres only, with a narrow write table feeding a read table

**Status:** Accepted, frozen for submission. Nothing is built or benchmarked. **Class:** Postgres and write/read separation are Necessary. Rollups, Redis and targets are Assumed. **Inputs:** Q3, Q4, Q7, Q15, Q19, Q21, Q26, Q27, Q32, Q37, Q44, Q45, Q50.

Nothing is benchmarked. Every performance figure here is a target or an estimate, with arithmetic in docs/assumptions.md.

## Context
- A collection holds up to 500k responses x 80 questions (brief).
- 300 dashboard users at peak (brief). 10,000 participants is my target.
- Filters combine freely, so pre-computing every combination is impossible.
- A team this small should run as few data systems as it can. I start with Postgres only, and name what would end that.

## The comparison
**Option A: one row per response.** Each answer updates a compact row holding all 80 coded answers.

**Option B: narrow write table feeding a read table.** Each answer inserts one row into an append-only table. A projector then updates a read table with one row per response.

| Criterion | A: one row per response | B: write plus read table |
| --- | --- | --- |
| Save an answer | Rewrites the row | Small idempotent insert |
| Retry safety | Needs versions inside the row | Unique key on the client's answer ID |
| Dashboard load can slow saves | Yes, same table and same vacuum | Less, but they still share a server |
| Data written | Close. About 48 GB of WAL against 50 GB per large collection in the high case (DERIVED from ASSUMED) | Close. B writes slightly more with no server autosave |
| Freshness | Immediate on the primary | Adds the projector's delay |
| Parts to run | One table | Two tables plus a projector that can fail or lag |
| Repair a bug or change the encoding | Risky migration of the only copy | Rebuild the read table from the write table |

**Data volume doesn't decide this.** I choose B because a participant's save stays a small insert even if the projector stops, and because the read table can be rebuilt. The cost is a component that can fail, so its correctness rules are explicit below.

## Decision
Use option B, in Postgres on Azure Database for PostgreSQL flexible server.

| Part | Decision |
| --- | --- |
| Write table | Append-only in normal use, deleted only for withdrawal and retention. One row per answer, source of truth. Unique key on collection ID, response ID and client answer ID. The response ID comes from the token, which scopes each answer to its participant. |
| Access | Row-level security on collection ID. Separate database roles for interview, projector, query, redaction and export. |
| Autosave | Drafts stay on the phone. Submitted answers, including later changes, reach the server. |
| Read table | One row per response, and so one per invitation. Coded answers in one compact column. Only screener fields such as age, gender and location are indexed. |
| Partitions | Each large collection gets its own partition in both tables. |
| Encoding | Coded values per question. The exact type is settled by the benchmark. |
| Free text | Transcripts live in their own tables, never in the read row. |
| Rollups | Counters by question, option or bin, and state (all or completed). Numeric questions also keep count and sum. |
| Search | Postgres full-text, with no pgvector (Q15). Weak in some languages, not tested. |

### Main tables
Column names are ASSUMED where earlier sections don't fix them. Every table carries the collection ID, with row-level security. User IDs come from the token, with no foreign key across stacks.

| Table | Key | Columns that matter |
| --- | --- | --- |
| `collection` | `id` | organization ID, status, retention, data version |
| `survey_version` | collection ID, version | definition, immutable once published |
| `respondent` | `response_id`: pseudonymous, time-ordered | collection ID, survey version, state, hashed device credential, hashed withdrawal code. No names or contacts |
| `identity.invite_map` | `invite_id`: random | response ID (unique, cleared on withdrawal), expiry, revoked, withdrawn. Restricted schema with its own database role (ADR 0003) |
| `withdrawal_tombstone` | `response_id` | time, projected marker, cleanup state. Kept until backups age out |
| `answer_write` | collection ID, response ID, client answer ID | question ID, value, device sequence, transaction ID |
| `response_read` | collection ID, response ID | indexed screeners, compact coded answers, completed |
| `conversation_turn` | collection ID, response ID, turn ID | state, lease expiry, fencing number, text and reply. Redacted text is in a separate table |
| `chat_turn` | collection ID, turn ID | user, org, role, claim expiry, state, lease expiry, fencing number, question, answer, sources, source response IDs (internal) (ADR 0005) |
| `rollup` | collection ID, question ID, option or bin, state | count, sum |
| `saved_filter` | `id` | collection ID, name, filter, version, owner |
| `collection_feature` | collection ID, feature | enabled (ADR 0004) |
| `transcript_grant` | collection ID, user ID | granted by, time (ADR 0004) |
| `export_request` | `id` | collection ID, requester, type, filter, approval state, file (ADR 0004) |
| `audit_log` | `id` | collection ID, user ID, action, validated filter, row count, time |

## Projector: what must always hold
The projector runs in the background worker deployment (ADR 0002). Saves don't depend on it.

| Requirement | Rule |
| --- | --- |
| Idempotent | Retrying a save, or replaying a batch, changes nothing the second time. |
| Safe recovery | The position, read rows, rollups and data version commit in one transaction. A crash replays the whole batch or none of it. |
| Nothing skipped | A transaction that commits late is still projected. |
| One owner | One projector runs per stack. Another takes over if it dies. |
| Changed answers | A later answer to the same question replaces the earlier one in the read table, ordered by device sequence. Rollups subtract the old value and add the new one. History stays in the write table. |
| Withdrawals | See below. A withdrawn response is designed not to return, and T9 tests this. |
| Redis catches up | Each projector run compares the committed data version with the one in Redis, and republishes if they differ. After a crash between commit and publish, a quiet collection recovers on the next run. |
| One contribution per invitation | The invite map's unique response ID gives one response per invitation, and the read table has one row per response. |
| Completion | The phone submits completion as an idempotent event row in the write table, so the projector sees it in order, and moves that response into the "completed" rollups. |
| If it stops | Saves continue. The dashboard goes stale and says so, using the "as of" time. |
| Reconciliation | A background job compares counts between write table, read table and rollups. A mismatch alerts, and can trigger a rebuild from the write table. |

**Mechanism (a starting point, to be proven by T8):** each write row stores `pg_current_xact_id()`, and the projector reads below `pg_snapshot_xmin(pg_current_snapshot())`. An advisory lock on a direct connection gives one owner. Batches run about every 2s (ASSUMED). If this proves unreliable, option A is the fallback.

## One invitation, one response
**Each invitation maps to one logical response (Q50).** The minimum group therefore counts invitations, and every path that reads the read table or rollups counts the same way.

This prevents duplicate counting within one invitation. It doesn't establish unique real people: one person with two invitations counts twice, and a forwarded invitation counts once. No identity resolution is attempted.

## Withdrawal
A participant starts this with their withdrawal code, or a client passes on a request by invite ID (ADR 0003). The projector is the only writer of the read table and rollups, so it also applies withdrawals in order. Every path below is untested (T9).

1. **Log, then mark.** Append the pseudonymous response ID to a withdrawal log in a dedicated Blob Storage container, separate from the export lifecycle. A database restore doesn't affect it. Then one transaction locks the respondent row, sets it to withdrawn, and adds the tombstone. New saves are rejected, and the phone clears its outbox when told. The participant sees confirmation only after both succeed.
2. **Project.** For each tombstone with no projected marker, in one transaction: subtract the read row from rollups, delete it, delete the write rows, set the marker, and move the data version. A marker rather than a time cursor means a late commit isn't missed.
3. **Late saves.** Saves check the tombstone without a lock, to stay cheap. The projector skips and deletes any write row with a tombstone, in every batch.
4. **Stale workers.** Turn, reply, chat and redacted-text writers share-lock the respondent row, and recheck its state in their transaction. Work for a withdrawn response is discarded.
5. **Clean up.** A worker job deletes transcripts, redacted transcripts and turn records, and applies the chat rule in ADR 0005. It clears the response ID on the invite map row and marks it withdrawn, so reopening the link can't create a new response. It also asks the AI service to delete that response's data, if the contract provides this (ADR 0003). It tracks this in the tombstone's cleanup state, and retries until done. The content-free `respondent` row stays as the marker.
6. **Exports.** An export excludes all of the collection's tombstones, read from the primary when it starts. Before publishing, it rechecks the primary, and restarts if there's a new one. Earlier files in that collection with no issued download link are deleted. Files with an issued link age out within 24 hours (ASSUMED).
7. **Aggregates.** See "Replica lag and stale aggregates" below.
8. **After a database restore.** Replay the log into the restored database before it takes traffic, which re-creates the tombstones. Steps 2 to 7 then run again, since no projected marker exists. Log entries stay until the last backup holding that response ages out.

**Limits:** downloaded exports can't be recalled (Q18). Backups keep the data until they age out. Answers saved after the backup are lost in a restore, unless phones still hold them.

## Read path
| Part | Decision |
| --- | --- |
| Live unfiltered views | After each batch, the projector writes suppressed rollups to Redis. Dashboards poll every 5s and read only Redis. |
| Filtered views and cross-tabs | Run on a read replica in one snapshot: read the data version, then run the query, so a lagging replica can't store old data under a new version. |
| Query cache | Regional Redis. Keyed by collection, keyed hash of the filter, chart set, policy version and a 30s bucket of the data version. Results only after suppression (Q27). Expires after 60s (ASSUMED). |
| Open filtered views | Refresh every 30s (ASSUMED), with a visible "as of" time and a rate-limited refresh button (Q37). |
| Limits | 10s statement timeout. Concurrency limits per user and per organization (ASSUMED, tune with T1). Beyond them, requests queue, then get "busy". |
| Privacy | One shared query layer applies suppression to every aggregate, including chat and summaries (ADR 0004). |

### Replica lag and stale aggregates
Every replica query reads the data version in its snapshot, so results carry their true age. Lag has no fixed upper bound: Microsoft says it can reach hours under heavy writes.

| Output | Rule |
| --- | --- |
| Unfiltered live views | Written by the projector from the primary, so replica lag doesn't affect them. |
| Filtered views and cross-tabs | Show "as of". They can include a withdrawn response until the replica replays the withdrawal, plus up to 60s of cache (ASSUMED). An alert fires when lag passes 4s (ASSUMED). |
| Summary and held small-segment values after a withdrawal | Held values clear at once. Rebuilds wait until the replica's data version reaches the withdrawal's version. |
| Opening a transcript | Reads the primary, so a deleted transcript isn't shown. |
| Transcript search and chat quotes | Results are checked against tombstones on the primary before return. |
| Row-level and transcript exports | As in step 6 of Withdrawal. |

### Targets (none measured)
| View | Target | How it's estimated |
| --- | --- | --- |
| Live unfiltered reads | Under 1s p95, about 8s old | 2s batch + 1s run + 5s poll (DERIVED from ASSUMED) |
| New filter or cross-tab, on a cache miss | Under 3s p95, about 10s old | Holds only if replica lag stays under 4s. Microsoft says lag can reach minutes or more. |
| Open filtered view, just before refresh | About 67s old | 3s + 4s lag + 30s bucket + 30s on screen (DERIVED, Q37) |
| Without Redis | Over the 10s target | Polls read the replica, and data is older |
| Small segments | Exempt | Update only after 10 new responses, if that proposed control is adopted (ADR 0004) |

## Scaling: measure first
**Start with one read replica,** sized from the benchmark (Q45). The earlier estimate of about 70 cores with no cache (DERIVED from an ASSUMED 1s scan) stays in docs/assumptions.md. It is an estimate of what may be needed, not a deployment.

**With one replica:**
- Losing it stops filtered views and chat tools until it is replaced. They return "busy". Queries don't fall back to the primary, so answer saves stay protected.
- Unfiltered live views keep working from Redis.
- If Redis is also down, unfiltered views go stale.

**Benchmark (T1):** synthetic data at 500k x 80, on replicas with the cache off, with worst-case filters and cross-tabs, reported separately for 8 and 80 charts per query.

**Escalate only on a measured miss,** stopping at the first step that passes:
1. Tune queries and indexes.
2. Add replicas, up to the limit of five.
3. Move the read table to its own Postgres server. This is also the answer if projector age or saves miss, where replicas can't help.
4. Add a columnar store fed from the write table. This needs my approval, and its own residency check.

**Columnar trigger:** ad-hoc p95 stays at or above 3s after steps 1 to 3, or clients need filtered views fresher than about 67s.

## Privacy and residency
| Component | Impact |
| --- | --- |
| Write and read tables | Hold pseudonymous health answers, in the collection's regional stack only. No direct identifiers. |
| Replica | Same region, so no new residency exposure. |
| Redis | Suppressed aggregates, counters and opaque IDs only (Q34), never answers or transcripts. In the regional stack, with no geo-replication. Filter hashes are keyed, because screener values are easy to guess. |
| Backups | Intended to stay within the geography, not verified (risk R21a). Withdrawn data remains in backups until they age out. |
| Tombstones and withdrawal log | Pseudonymous response IDs, kept in the region. |

## Consequences and risks
- The projector is a new failure point, covered by the rules above, reconciliation and alerts.
- Query capacity is unknown until T1. The first replica may not be enough.
- Multiple choice adds write rows beyond 40M. Not yet sized.
- Replica lag can grow under sustained writes. This affects filtered views only.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Several ranked attempts per invitation | Tried, then dropped (Q50). Rules for ranking, late sync, completion and withdrawal outweighed the benefit. |
| Identity resolution across invitations | Out of scope, and more linking of health data. |
| A: one row per response | Close on data volume, and simpler. Rejected because it mixes saves with dead rows and vacuum, and leaves no history to rebuild from. It is the fallback. |
| Sizing 3 replicas of 32 vCores up front | Rests on an unmeasured scan. Replaced by measured scaling (Q45). |
| Per-answer Inngest event as the projector | Would hand a vendor a per-participant timeline. |
| Projector cursor on "ID greater than last seen" | Silently skips transactions that commit late. |
| Logical decoding | A stalled slot holds WAL on the primary, and storage pressure can force it read-only. |
| Triggers on the write table | Puts read table work back into the participant's save. |
| Withdrawal rollup fix as a later job | A crash after the delete would lose the values needed to subtract. |
| Pushing live updates over a stream | Needs fan-out across instances. Polling is simpler, and reads only Redis. |
| Refreshing filtered views every 10s | About 80 cores more (DERIVED from ASSUMED). |
| Live pinned filter sets | Dropped by my decision. Membership, backfill, withdrawal and reconciliation each became a new way to show wrong health figures. |
| Query the narrow table directly | Filtering on several questions needs joins across up to 40M rows per collection. |
| One column per question | Schema changes per collection, and many sparse columns. |
| Materialized views | Full refreshes can't meet the 10s target at this size (ASSUMED). |
| Columnar store now | A second system before any number shows the need. Allowed later by the trigger above. |
