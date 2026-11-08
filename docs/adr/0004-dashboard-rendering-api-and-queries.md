# ADR 0004: Dashboard as a static client app, a REST API, and one query layer

**Status:** Proposed, for my review. **Inputs:** Q4, Q7, Q15, Q16, Q18, Q19, Q20, Q21, Q24, Q26, Q27, Q28, Q29, Q32, Q34. **Open:** Q37. **Risks:** docs/risks.md.

Figures are ASSUMED or DERIVED, with arithmetic in docs/assumptions.md. Nothing is benchmarked. Chat and the summary are in ADR 0005.

## Rendering

| Criterion | Static client app (chosen) | Server-rendered Next.js | Server components with streaming |
| --- | --- | --- | --- |
| Health data in HTML | None | In every rendered page | In every rendered page |
| Delivery | Front Door, same files for all | A Next.js server in every region, through the gateway | Same |
| Parts to run | None added | One more service per region | One more service per region |
| Auth | Browser sends the JWT to the regional API | Server must hold or forward the token | Same |
| First paint | Slower, then data loads | Faster | Fastest |
| Value of search indexing | None, behind login | None | None |

**Decision:** a Next.js static export served from Front Door, with shadcn/ui and client-side data fetching. A health dashboard behind a login gains little from server rendering, and each server adds a place where data is decrypted and rendered. Researchers use desktops and decent networks, so the slower first paint is an acceptable cost.

| Point | Decision |
| --- | --- |
| Finding the region | After login, the app asks the control plane for the org's collections and their regions, then calls that region's API host. |
| Token | The session stays at the control plane. The JWT is held in memory only, and renewed before its 5-minute expiry. |
| Charts | Recharts through shadcn/ui charts (ASSUMED suitable). A chart is a client render of aggregates only. |
| 80 questions | Virtualized list. A chart only loads when near the viewport, and requests are grouped into one query per view. |
| Accessibility | Every chart has a table view. |
| Analytics and errors | PostHog on UI events only, with no autocapture and no replay. Sentry on an allowlist that strips URLs and breadcrumbs, with no replay and no IP capture (Q17). Filter state never goes in dashboard URLs. |
| Security headers | A strict content security policy, since a script injected here could read the token. |

## API style

| Criterion | REST with OpenAPI (chosen) | GraphQL | tRPC |
| --- | --- | --- | --- |
| Limiting query cost | Each endpoint has one known cost | Clients shape queries, so cost needs extra analysis | Each endpoint has one known cost |
| Fits the WAF and gateway paths | Clear paths and timeouts per route | One path for everything | Paths exist, but are tied to the client |
| Streaming and long-poll | Plain HTTP | Needs a separate transport | Needs an adapter |
| Typed client | Generated from OpenAPI | Generated | Built in |
| Use by a Derin engine or others | Open | Open | TypeScript only |

**Decision:** versioned REST on NestJS, with an OpenAPI contract and a generated TypeScript client. Read queries that carry filters use POST, so filter values stay out of URLs and logs. Polls and streams use GET, carrying IDs only. Every request goes through the same guard: token, role, feature, collection, rate limit.

| Endpoint | Purpose | Minimum role |
| --- | --- | --- |
| `GET /v1/collections/{id}/live?since={version}` | 5s poll. Unfiltered counts and charts from Redis. Empty if the version is unchanged. | Viewer |
| `POST /v1/collections/{id}/views` | Charts for a filter. | Viewer |
| `POST /v1/collections/{id}/crosstabs` | One cross-tab, with optional filter. | Viewer |
| `GET, POST, PATCH, DELETE /v1/collections/{id}/filter-sets` | Saved filters. | Read: viewer. Write: analyst. |
| `POST /v1/collections/{id}/transcripts/search` | Find redacted transcripts in a filter. | Analyst |
| `GET /v1/collections/{id}/transcripts/{ref}` | One redacted transcript. | Analyst |
| `POST /v1/collections/{id}/exports`, `GET .../exports/{id}` | Start and check an export. | Analyst; row-level admin |
| `POST /v1/collections/{id}/chat/turns` and `GET .../{turn}`, `GET .../{turn}/stream` | Chat, with SSE and long-poll (ADR 0005). | Viewer |
| `GET /v1/collections/{id}/summary` | Latest executive summary. | Viewer |

## Roles and features
Enforced in NestJS, in every path. The role comes from the token (Q28).

| Action | Viewer | Analyst | Admin |
| --- | --- | --- | --- |
| Counts, charts, cross-tabs, saved filters, summary | Yes | Yes | Yes |
| Chat over aggregates | Yes | Yes | Yes |
| Create or edit saved filters | No | Yes | Yes |
| Aggregate export | No | Yes | Yes |
| Read redacted transcripts, chat with transcript quotes | No | Yes | Yes |
| Row-level or transcript export | No | No | Yes, with live control plane check |
| Manage members and features | No | No | Yes |

**Per-collection features:** a `collection_feature` table in the regional stack, holding switches such as chat, transcripts and early-access charts. The guard checks it, and the result is cached for 30s (ASSUMED). The client gets the list to hide controls, but the server is what enforces them.

**Org check:** the guard compares the token's org with the collection's org on every request, because a role only means something within its org. Row-level security on `collection_id` is a second check (ASSUMED). Reads of chat turns and exports also check the owner. Whether users outside the owning org need access is Q38.

**Transcript grant:** reading transcripts needs an analyst or admin role **and** a per-collection grant from an admin. It is not automatic with the role.

**Account security:** MFA is required for admins and analysts, using Better Auth's two-factor plugin. Enforcing it per role is our own check (ASSUMED). Row-level and transcript exports also need a session under 15 minutes old (ASSUMED), using its documented session freshness.

**Audit:** an append-only table in the regional Postgres, recording user, action, collection, validated filter, row count and time. It covers ad-hoc queries, chat tool calls, transcript reads, exports, and role and feature changes. Kept for 1 year (ASSUMED), with restricted access. The filter makes it sensitive, but it holds no answers.

## The query layer
All aggregate reads pass through one NestJS module. Nothing else can read the read table.

1. Validate the filter against the collection's question definitions.
2. Compile it to parameterized SQL. No text from the request ever enters SQL.
3. Read the data version and run the query in one snapshot on a replica (ADR 0001).
4. Apply suppression.
5. Store in Redis, and audit.

A filter is JSON. Clauses are ANDed, and values within a clause are ORed:

```json
{ "all": [
  { "q": "gender", "in": ["woman"] },
  { "q": "age", "between": [30, 50] },
  { "q": "city", "in": ["nairobi"] }
], "state": "all" }
```

### Suppression and comparison controls (Q7, Q21)
The commitment is to **reduce** the risk of working out hidden values, not to prevent it entirely. Every number here applies to all roles and every aggregate output. All paths call one suppression function, including the projector's live rollups and the Redis-down read.

The privacy review showed a simple attack: chart `city in [nairobi, village]`, chart `city in [nairobi]`, and subtract, which reveals a village of 3. These controls are built to stop that attack.

| Control | Rule |
| --- | --- |
| Minimum group | A segment under 10 responses shows "too few responses", with no count. |
| Clause minimum | Each value in an ORed clause must reach 10 on its own. Age and numeric filters use fixed bands, each of which must reach 10. Otherwise the filter is refused. |
| Cell minimum | Any cell or bin under 10 shows as suppressed. |
| Secondary suppression | If a cell in a row or column is suppressed, more are too, until the hidden total reaches 10. |
| Complement | If total minus segment is under 10, the segment is suppressed. The same applies to "all" minus "completed". |
| Rounding | **All filtered counts round to the nearest 5** (ASSUMED), at every segment size. Percentages use rounded counts. Means come from rounded bins, and round to a coarse step. |
| Live comparison | Results are held by normalized filter, so equivalent filters share one held value. A segment under 100 only updates after 10 new responses, and unfiltered live charts do the same when responses are slow. These are exempt from the 10s target. |
| Numeric extremes | Minimum and maximum are never shown. |
| Identical answers | If everyone in a segment gave one answer, the chart shows a warning, and doesn't suppress. A finding like "all said yes" is valid. This remains a known exposure. |
| Detection | Audit stores the validated filter. It flags pairs differing by one clause value. An alert goes to all admins and to Surgo, and doesn't block. |

These controls don't stop every overlap, or a determined person with outside knowledge. Rounding means cross-tabs may not sum exactly, and won't match row-level exports. The risk list says so.

## Charts by question type
Unfiltered charts come from rollups. Filtered charts come from one scan of the read table, using the same shape.

| Type | Stored in rollups | Chart | Note |
| --- | --- | --- | --- |
| Single choice | Count per option | Bar | Percent of those who answered |
| Multiple choice | Count per option, plus respondents | Bar | Percentages can exceed 100, and the chart says so |
| Scale | Count per point, plus sum | Bar with mean | Mean hidden if the group is under 10 |
| Numeric | Count per bin, plus count and sum | Histogram with mean and estimated median | Up to 20 bins (ASSUMED), set in the question definition. Median comes from bins, and is labelled as an estimate. |
| All types | Each counter exists for "all" and "completed" | Toggle (Q19) | A completion adds that response's answers to the "completed" counters |

A changed answer subtracts the old value and adds the new. Conversational answers aren't charted, since the brief lists only these four types. They appear as transcripts.

## Filters and saved sets
| Point | Decision |
| --- | --- |
| Fields | Any closed question, plus the completed toggle. No free-text filters. |
| Speed | Screener fields are indexed. Filters on other questions scan the collection's partition. |
| Saved sets | Stored in the regional Postgres, with the collection: name, filter JSON, owner, version. The definition only, never a result. |
| Sharing | Visible to everyone in the org with access to the collection. Editable by analysts and admins. |
| Reuse | Saved sets belong to one collection, and are versioned with its question definitions, which are immutable once published (Q19). Copying to another collection re-validates. |
| Privacy | A saved set passes through the same suppression, so sharing one reveals nothing new. Filter labels are written by staff, and should not name individuals. |

## Cross-tabs
| Point | Decision |
| --- | --- |
| Shape | One row question by one column question, with an optional filter. |
| Limit | Up to 30 x 30 cells (ASSUMED). Numeric questions use their bins. |
| Execution | Not pre-computed. One aggregate query in the same scan, cached like a view. Two questions per collection has 3,160 pairs (DERIVED), and most are never opened. |
| Multiple choice | Each selected option counts once, so totals can exceed respondents, and the table says so. This costs more per row than single choice, and is in the benchmark. |
| Suppression | All controls above. Row and column totals are suppressed if they would reveal a hidden cell. A hidden cell can still be estimated from one-way charts on the same filter, which rounding limits but doesn't remove. |
| Output | Counts, and row or column percentages from those counts. Statistical tests aren't in scope. |
| Export | Aggregate export of the table as shown, including suppression. |

## Transcripts and redaction
| Point | Decision |
| --- | --- |
| Service | Azure Language **Text PII**, on a regional resource. Microsoft lists Arabic, Swahili, Somali and Amharic for Text PII. Its Conversation PII lists only English, French, German and Spanish, so we don't use that. |
| When | A Postgres-backed NestJS worker, like the projector, in batches, about 100 to 200 texts/s (DERIVED). Not one Inngest job per answer, which would give a vendor a per-participant timeline. Both sides of the conversation are redacted, since a reply can repeat a name. Unredacted text is never shown on the dashboard. |
| Language gate | A language must pass a redaction test before its transcripts are browsable. A failing language stays hidden. |
| Storage | Raw and redacted text are in separate tables, with separate database roles. Only interview generation (own response only) and the redaction worker read raw. No replica grant on raw tables. See "Retention". |
| If redaction is missing | The transcript stays hidden. |
| Browsing | Needs the transcript grant, and a filter matching 10 or more. **The floor applies after search terms too.** Exact match counts are hidden. Random order within a recent window, 20 per page (ASSUMED). At most 200 transcripts per user per day (ASSUMED). Each read is audited. |
| Identity | A reference issued per result list, never reused across filters, so a person can't test which filters a transcript appears in. Opening a source quote uses a short-lived link. |
| Display | Plain text only, since participants wrote it. |
| Refusal flags | Self-harm refusal categories can't be filtered on, and aren't shown with the transcript. Access to them belongs to the client's safeguarding contact. |
| Search | Postgres full-text on the redacted table, by language where a configuration exists, otherwise simple matching. Weak for some languages. |

Redaction reduces risk, and misses things. Microsoft notes that detection varies by language and locale. A person can also be identified by their story without any named entity. So transcripts remain a separate audited permission.

## Exports
| Point | Aggregate export | Row-level export | Transcript export |
| --- | --- | --- | --- |
| Who | Analyst, admin | Admin only | Admin only |
| Contents | Tables as shown, after suppression | Coded answers, with screeners in bands. **No raw numeric values or timestamps.** | Redacted text |
| Filter | Any | Must match 10 or more, and each combination of screener bands must have 10 or more rows | Must match 10 or more |
| Approval | No | A second admin approves. All admins are notified. | Same |
| Live control plane check | No | Yes, and blocked if it's down (Q29) | Yes, and blocked if it's down |
| IDs and order | None | A random ID per export, with rows shuffled | Same |

**Row-level export is outside the minimum of 10.** An admin can compute suppressed cells from the file. Two overlapping exports can also be matched on row content, so random IDs don't stop joining them. A second export overlapping an earlier one needs approval again, and the audit records it. We say this openly to clients.

**How it runs:**
1. The API records the export and starts an Inngest job, carrying IDs only.
2. A NestJS export worker reads from a replica and writes CSV to Blob Storage in chunks, so it can resume.
3. At most 2 run at once per stack (ASSUMED). They still use replica capacity, so they are in the sizing.
4. The requester downloads through a **user delegation SAS** lasting 15 minutes (ASSUMED), directly from Blob Storage. This keeps large files off the unbuffered gateway (ADR 0002). Role and control plane are checked again when the SAS is issued, and each SAS is audited.
5. A lifecycle policy deletes the file after 24 hours (ASSUMED). Blob logs give download evidence (ASSUMED, not checked).

| Point | Decision |
| --- | --- |
| Size | About 160 MB for coded answers at 500k x 80, and about 5 GB for transcripts (DERIVED from ASSUMED). |
| Storage | Regional, zone-redundant, with no geo-replication. Public anonymous access off. |
| Spreadsheet safety | Cells starting with `=`, `+`, `-`, `@`, tab or carriage return are escaped, because participants wrote the text. |
| Withdrawal | **A downloaded file can't be recalled** (Q18), and the export screen says so. A withdrawal deletes undownloaded files that contain that response. |

## Retention
All periods are ASSUMED. A retention period set per collection (Q18) can shorten them. A collection that never closes is treated as closed after 2 years.

| Store | Kept until |
| --- | --- |
| Write table, read table, redacted transcripts | Collection's retention period |
| Raw transcripts, turn records | Collection close + 30 days |
| Export files | 24 hours |
| Chat history | 30 days, or a source's withdrawal, which removes that quote |
| Audit | 1 year |
| Gateway, WAF and application logs | 30 days |
| Backups | Age out with their retention window |

## Privacy and residency
| Component | Impact |
| --- | --- |
| Front Door | Serves app files only. |
| Browser | Holds aggregates in memory, and redacted text only for analysts and admins. Nothing is stored locally except UI preferences. |
| Query layer | The one path to aggregates, so one bug affects everything, which also means one place to test. |
| Redis | Suppressed aggregates and held small-segment values. Filter hashes are keyed. |
| Azure Language | Processes raw transcript text. Regional resource. Whether all processing stays in the region is ASSUMED, and on the risk list. |
| Blob Storage | Files with row-level health data, in the region, for 24 hours. A SAS is a bearer link until it expires. |
| PostHog, Sentry | No answers, filter values or chart data. Hosting regions unverified. |
| Audit table | Pseudonymous user IDs and filter hashes. In the region. |

## Consequences and risks
- The query layer is a single point of correctness. It needs the heaviest test coverage in the system, including tests that try to recover suppressed values.
- Rounding makes all filtered numbers slightly inexact, and rare values can't be filtered on. Researchers must be told. This is the price of making the suppression hold.
- A leaked SAS link works until it expires. Microsoft notes revoking one can be delayed.
- Redaction depends on Azure Language capacity, which isn't checked. A backlog delays transcript browsing, not interviews.
- An open filtered view can show data up to about 67s old (Q37).
- The query limit of 5 per organization could leave one large client queuing. The number of organizations isn't assumed anywhere.
- The first paint is slower than a server-rendered page.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Server-rendered dashboard | Adds a service per region, and puts health data in rendered HTML, for a login-only tool on decent networks. |
| GraphQL | Clients could compose expensive filtered queries. Cost control and suppression are simpler per endpoint. |
| tRPC | Good for a single TypeScript client, but ties the contract to it. |
| Pre-computing cross-tabs | 3,160 pairs per collection, and filters multiply them further. |
| Cubes by screener fields | Would speed common filters, but adds a store to keep correct, before the benchmark shows a need. First thing to try if scans miss the target. |
| Sampling for fast estimates | Researchers expect real counts, and small samples worsen disclosure rules. |
| Suppression in database views | Harder to test with secondary suppression and rounding. Row-level security still applies per collection as a second check (ASSUMED). |
| Adding noise to counts | Stronger against comparison, but harder to explain to clients. Rounding plus the clause minimum handle the known attack. Noise is the next step if tests still recover values. |
| Exact counts for segments of 100 or more | Lets anyone subtract two large segments to reveal a small one. |
| Row-level export with no extra limits | Would make one admin's export the easiest way around every other control. |
| Conversation PII | Lists only 4 languages. |
| Exports through the API | Large files would hold unbuffered connections through the gateway. |
| Excel files | CSV is smaller to generate and stream. Excel can be added later. |
