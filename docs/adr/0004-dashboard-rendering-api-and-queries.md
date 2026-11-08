# ADR 0004: Dashboard as a static client app, a REST API, and one query layer

**Status:** Accepted, frozen for submission. Nothing is built or benchmarked. **Class:** Live charts, filters, saved sets, cross-tabs, transcripts, exports, roles and per-collection features are Required. The query layer and audit are Necessary. Disclosure values, retention and limits are Assumed. **Inputs:** Q4, Q7, Q15, Q16, Q18, Q19, Q20, Q21, Q26, Q28, Q29, Q32, Q37, Q38, Q43, Q44, Q46, Q50. **Risks:** docs/risks.md.

Figures are ASSUMED or DERIVED, with arithmetic in docs/assumptions.md. Nothing is benchmarked. Chat and the summary are in ADR 0005.

## Rendering
| Criterion | Static client app (chosen) | Server-rendered Next.js |
| --- | --- | --- |
| Health data in HTML | None | In every rendered page |
| Parts to run | None added | One more service per region |
| Auth | Browser sends the JWT to the regional API | Server must hold or forward the token |
| First paint | Slower, then data loads | Faster |
| Value of search indexing | None, behind login | None |

**Decision:** a Next.js static export served from Front Door, with shadcn/ui and client-side data fetching. Researchers use desktops and decent networks, so the slower first paint is an acceptable cost.

| Point | Decision |
| --- | --- |
| Finding the region | After login, the app asks the control plane for the org's collections and their regions. |
| Token | The session stays at the control plane. The JWT is held in memory, and renewed before its 5-minute expiry. |
| Charts | Recharts through shadcn/ui charts (ASSUMED suitable). Aggregates only. Every chart has a table view. |
| 80 questions | Virtualized list. A chart loads near the viewport, and requests are grouped into one query per view. |
| Analytics and errors | PostHog on UI events only, with no autocapture or replay. Sentry on an allowlist. Filter state never goes in URLs. |
| Security headers | A strict content security policy, since an injected script could read the token. |

## API style
| Criterion | REST with OpenAPI (chosen) | GraphQL | tRPC |
| --- | --- | --- | --- |
| Limiting query cost | Each endpoint has one known cost | Clients shape queries | Each endpoint has one known cost |
| Streaming and long-poll | Plain HTTP | Separate transport | Needs an adapter |
| Typed client | Generated | Generated | Built in |
| Use by other services | Open | Open | TypeScript only |

**Decision:** versioned REST on NestJS, with OpenAPI and a generated TypeScript client. Filters go in POST bodies, so their values stay out of URLs and logs. Every request goes through one guard: token, org against collection, role, feature, rate limit.

| Endpoint | Purpose | Minimum role |
| --- | --- | --- |
| `GET /v1/collections/{id}/live?since={version}` | 5s poll, from Redis | Viewer |
| `POST /v1/collections/{id}/views` | Charts for a filter | Viewer |
| `POST /v1/collections/{id}/crosstabs` | One cross-tab | Viewer |
| `GET, POST, PATCH, DELETE /v1/collections/{id}/filter-sets` | Saved filters | Read: viewer. Write: analyst |
| `POST /v1/collections/{id}/transcripts/search`, `GET .../transcripts/{ref}` | Redacted transcripts | Analyst with grant |
| `POST /v1/collections/{id}/exports`, `GET .../exports/{id}` | Start and check an export | Analyst. Row-level: admin. Transcript: admin with grant |
| `POST /v1/collections/{id}/chat/turns`, `GET .../{turn}` | Chat: the POST responds with SSE, and GET fetches the saved answer (ADR 0005) | Viewer |
| `GET /v1/collections/{id}/summary` | Latest summary | Viewer |
| `POST /v1/collections/{id}/invites/{invite}/withdraw`, `.../revoke` | Act on a participant's request. Returns no response data. Audited | Admin |

## Roles and features
| Action | Viewer | Analyst | Admin |
| --- | --- | --- | --- |
| Counts, charts, cross-tabs, saved filters, summary, chat over aggregates | Yes | Yes | Yes |
| Create or edit saved filters, aggregate export | No | Yes | Yes |
| Read redacted transcripts, chat with quotes | No | With grant | With grant |
| Row-level or transcript export | No | No | Yes, with the restrictions under Exports |
| Manage members, withdraw or revoke an invitation | No | No | Yes |

- **Per-collection features:** a `collection_feature` table holds switches such as chat, transcripts and early-access charts. The server enforces them, cached for 30s (ASSUMED). The client only hides controls.
- **Who activates early access is a product assumption, not a decision (Q46).** The table supports either an org admin or Surgo staff. No role row or endpoint is designed for this until the owner is known.
- **Org check:** the token's org must match the collection's org, on every request. Row-level security is a second check. One org owns each collection (Q38).
- **Transcript grant:** needs an analyst or admin role **and** a per-collection grant from an admin.
- **Account security:** MFA for admins and analysts, using Better Auth's two-factor plugin. Per-role enforcement is our own check (ASSUMED). Row-level and transcript exports need a recent session, 15 minutes (ASSUMED).
- **Audit:** append-only, in the regional Postgres: user, action, collection, validated filter, row count and time. Covers ad-hoc queries, chat tool calls, transcript reads, exports, withdrawals, revocations, and grant and feature changes. Holds no answers, but filters make it sensitive.

## The query layer
**This is the most important decision in the dashboard.** All dashboard and AI aggregate reads pass through one NestJS module. Only the projector, export, transcript and rebuild roles also have grants on the read table.

1. Validate the filter against the collection's question definitions.
2. Compile it to parameterized SQL. No request text enters SQL.
3. Read the data version and run the query in one snapshot on the replica (ADR 0001).
4. Apply the disclosure policy.
5. Cache in Redis, and audit.

Filter clauses are ANDed, and values within a clause are ORed:

```json
{ "all": [
  { "q": "gender", "in": ["woman"] },
  { "q": "age", "between": [30, 50] },
  { "q": "city", "in": ["nairobi"] }
], "state": "all" }
```

## Disclosure
**The requirement is architectural: one enforcement point, for every path.** Charts, cross-tabs, live rollups, the Redis-down path, chat tools and summaries all call one function. Transcripts and row-level exports are separate, gated paths (Q20).

**One invitation contributes at most one participant (Q50).** Every count here is per invitation (ADR 0001). That doesn't verify real people, so "10" means 10 invitations.

**The values are policy.** Surgo's disclosure policy isn't known, so everything below is **proposed, for review (Q44)**. They live in a versioned policy object, so changing a value needs no redesign, and the version is part of the cache key.

| Proposed control | Rule | Status |
| --- | --- | --- |
| Minimum group | A segment or cell under 10 invitations shows "too few responses" | Proposed baseline, for every role |
| Complement | If total minus segment is under 10, the segment is suppressed. Same for "all" minus "completed" | Proposed baseline |
| Secondary suppression | Hide more cells in a row or column until the hidden total reaches 10 | Proposed baseline |
| Clause minimum | Each ORed value, and each fixed age or numeric band, must reach 10 alone | Proposed |
| Rounding | Filtered counts round to the nearest 5. Percentages use rounded counts | Proposed |
| Live hold | A segment under 100 only updates after 10 new responses. Held values clear when a response withdraws, and rebuild as in ADR 0001 | Proposed |
| Numeric extremes | Minimum and maximum are never shown | Proposed |
| Near-identical filter alert | Audit flags pairs differing by one clause value, and alerts without blocking | Proposed |

**These reduce the risk of working out hidden values. They don't prevent it.** Rounding and suppression can't fully stop differencing: enough overlapping queries, averaging, or outside knowledge can still narrow down a value. If everyone in a segment gave one answer, that reveals it too. The chart warns, and doesn't suppress. T4 tests known attacks. Adding noise is the next step if tests recover values.

**Costs:** rounding makes filtered numbers slightly inexact, so cross-tabs may not sum and won't match row-level exports. The clause minimum means rare values can't be filtered on.

## Speed: four classes of work
Everything here is a target or estimate, never a measurement.

| Class | Examples | Path | Target |
| --- | --- | --- | --- |
| Unfiltered live views | Counts and charts | Redis | Under 1s p95, about 8s old |
| Filtered queries | A filter, saved or ad-hoc | Replica scan, with cache | Under 3s p95 on a cache miss. An open view can be about 67s old, with "as of" and a refresh button (Q37) |
| Cross-tabs | One table | Same scan path | Same as filtered. Multiple choice costs more, and is in T1 |
| Slow or background | Exports, summary, redaction, a view with many charts | Background worker, or charts loading in stages | No interactive target. They show progress |

Overload: a 10s statement timeout and concurrency limits, then a "busy" response.

## Charts by question type
Unfiltered charts come from rollups. Filtered charts use one scan of the read table, in the same shape.

| Type | Stored in rollups | Chart | Note |
| --- | --- | --- | --- |
| Single choice | Count per option | Bar | Percent of those who answered |
| Multiple choice | Count per option, plus respondents | Bar | Percentages can exceed 100, and the chart says so |
| Scale | Count per point, plus sum | Bar with mean | Mean hidden if the group is under the minimum |
| Numeric | Count per bin, plus count and sum | Histogram with mean and estimated median | Up to 20 bins (ASSUMED) |
| All types | Counters for "all" and "completed" | Toggle (Q19) | One response per invitation, so completed counts at most one (ADR 0001) |

Conversational answers aren't charted, since the brief lists only these four types. They appear as transcripts.

## Filters and saved sets
| Point | Decision |
| --- | --- |
| Fields | Any closed question, plus the completed toggle. No free-text filters. |
| Speed | Screener fields are indexed. Others scan the collection's partition. |
| Saved sets | Stored with the collection: name, filter JSON, owner, version. The definition only, never a result. Use the same path as ad-hoc filters. |
| Sharing | Visible to the org that owns the collection. Editable by analysts and admins. |
| Reuse | Versioned with the question definitions, which are immutable once published (Q19). |

## Cross-tabs
| Point | Decision |
| --- | --- |
| Shape | One row question by one column question, with an optional filter. Up to 30 x 30 cells (ASSUMED). |
| Execution | One aggregate query, cached like a view. Two questions per collection has 3,160 pairs (DERIVED), so pre-computing is out. |
| Multiple choice | Totals can exceed respondents, and the table says so. |
| Suppression | The same policy. A hidden cell can still be estimated from one-way charts on the same filter. Controls limit this, and don't remove it. |
| Export | Aggregate export of the table as shown. |

## Transcripts and redaction
**Redaction reduces identification risk. It doesn't guarantee anonymity.** It misses things, and a person can be identified by their story with no named entity. So transcripts stay a separate, audited permission.

| Point | Decision |
| --- | --- |
| Service | Azure Language Text PII, on a regional resource (ASSUMED). Accuracy, language coverage and processing location aren't verified. |
| Language gate | A language must pass our redaction test before its transcripts are browsable. A failing language stays hidden. |
| When | A background worker, in batches. Both sides of the conversation are redacted, since a reply can repeat a name. |
| If redaction is missing | The transcript stays hidden. Unredacted text is never shown on the dashboard. |
| Storage | Raw and redacted text in separate tables and database roles. No replica grant on raw tables. |
| Browsing | Needs the grant, and a filter matching the minimum, **including after search terms**. Exact match counts are hidden. Random order, and page and daily limits (ASSUMED). Each read is audited. |
| Identity | A reference issued per result list. This makes it harder to test which filters a transcript appears in, and identical quote text can still match. Opening a transcript reads the primary (ADR 0001). |
| Display | Plain text only. |
| Refusal flags | Self-harm categories can't be filtered on, and aren't shown with the transcript. |
| Search | Postgres full-text on the redacted table. Weak for some languages. |

## Exports
| Point | Aggregate | Row-level | Transcript |
| --- | --- | --- | --- |
| Who | Analyst, admin | **Admin can start it** | **Admin with the transcript grant can start it** |
| Contents | Tables as shown, after suppression | Coded answers, with screeners in bands. No raw numeric values or timestamps | Redacted text, in languages that passed the redaction test only |
| Filter (proposed) | Any | Matches the minimum, and each combination of screener bands has that many rows | Matches the minimum |
| Live control plane check | No | Yes, blocked if down (Q29) | Yes, blocked if down |
| Recent session and MFA | No | Yes | Yes |
| Audit and notice to all admins | Audit | Both | Both |
| IDs and order | None | Random ID per export, shuffled. No invite or response ID | Same |

### Approval is an optional organizational policy (Q43)
| Point | Position |
| --- | --- |
| Baseline | An admin can start a row-level or transcript export. No second person is needed. |
| Optional policy | An org setting on the control plane (ASSUMED) can require a second admin's approval. The live check returns it. |
| Single-admin org with the setting on | Exports wait until a second admin exists. |
| Pending | **Any further or mandatory approval requirement is pending Surgo's policy.** |
| Known limit | Without it, one stolen or misused admin account can export, relying on MFA, recent sign-in, the live check and audit (R2b). Notice to admins adds nothing in a one-admin org. With it, an admin could still create a second account. |

**Row-level export sits outside the minimum group.** An admin can compute suppressed cells from the file, and match overlapping exports on row content. We say so openly to clients.

**How it runs:**
1. The API records the export and starts an Inngest job, carrying IDs only.
2. The background worker reads from the replica by response ID ranges, so each response appears once, and writes CSV to Blob Storage in chunks. Chunks may reflect slightly different moments, and the file states its range. Withdrawal rules are in ADR 0001. Exports run one or two at a time (ASSUMED), so they can compete with queries while there's one replica.
3. The requester downloads directly from Blob Storage through a **user delegation SAS** lasting 15 minutes (ASSUMED), so large files stay off the gateway. The API re-checks role and control plane first, and audits it. **A leaked link works from anywhere until it expires.**
4. A lifecycle policy deletes the file after 24 hours (ASSUMED).

| Point | Decision |
| --- | --- |
| Spreadsheet safety | Cells starting with `=`, `+`, `-`, `@`, tab or carriage return are escaped. |
| Withdrawal | **A downloaded file can't be recalled** (Q18), and the screen says so. Undownloaded files are deleted, and running exports recheck before publishing (ADR 0001). |

## Retention and withdrawal
All periods are proposed and ASSUMED, and can be set per collection (Q18). The legal basis is out of scope.

| Store | Kept until |
| --- | --- |
| Write table, read table, redacted transcripts | Collection's retention period. A collection that never closes is treated as closed after 2 years |
| Raw transcripts, turn records | Collection close + 30 days |
| Export files | 24 hours |
| Chat history | 30 days, or a source's withdrawal, which replaces the affected turns (ADR 0005) |
| Audit | 1 year |
| Gateway, WAF and application logs | 30 days |
| Backups | Their retention window |
| Withdrawal tombstones and log | Until the last backup holding that response ages out. The log's container has its own lifecycle |
| Unsynced data on phones | Deleted on acknowledgement, otherwise cleaned up when the app next runs (ADR 0003) |

Withdrawal itself is in ADR 0001.

## Privacy and residency
| Component | Impact |
| --- | --- |
| Front Door | Serves app files only. |
| Browser | Holds aggregates in memory, and redacted text only for permitted users. |
| Query layer | The one path to aggregates, so one bug affects everything, which also means one place to test. |
| Redis | Suppressed aggregates. Filter hashes are keyed. |
| Redaction service | Processes raw transcript text. Location not verified (R10). |
| Blob Storage | Row-level health data, in the region, for 24 hours (ASSUMED), on a public endpoint. **A downloaded file then sits outside any geography.** |
| PostHog, Sentry | No answers, filter values or chart data. Hosting not verified. |
| Audit table | Pseudonymous user IDs and validated filters, with restricted access. |

## Consequences and risks
- The query layer is a single point of correctness. It needs the heaviest tests, including attempts to recover hidden values.
- Proposed controls may frustrate researchers (R21b).
- A Redis failover blanks held small segments until new state builds (R21g).
- The first paint is slower than a server-rendered page.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Server-rendered dashboard | Adds a service per region, and puts health data in rendered HTML, for a login-only tool. |
| GraphQL | Clients could compose expensive queries. Per-endpoint cost control is simpler. |
| tRPC | Ties the contract to one TypeScript client. |
| Pre-computing cross-tabs | 3,160 pairs per collection, and filters multiply them. |
| Cubes by screener fields | A store to keep correct, before any measurement shows a need. First thing to try if scans miss. |
| Sampling | Researchers expect real counts, and small samples worsen disclosure. |
| Suppression in database views | Harder to test with secondary suppression. |
| Adding noise now | Harder to explain to clients. Next step if tests recover values. |
| Hard-coding disclosure values | Surgo's policy isn't known. |
| Mandatory second-admin approval | Blocks one-admin orgs, and is a business policy we can't invent (Q43). |
| Exports through the API | Large files would hold unbuffered connections through the gateway. |
| Blob Storage on a private endpoint only | A researcher's browser couldn't reach it. |
| Excel files | CSV is simpler to generate and stream. |
