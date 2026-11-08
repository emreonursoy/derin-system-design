# Assumptions and derivations

Rule: every figure is DERIVED (show the arithmetic) or ASSUMED (say why). Update when a decision changes.

## Inputs
| Input | Value | Source |
| --- | --- | --- |
| Rows per large collection | 500k responses x 80 questions = 40M | brief |
| Dashboard concurrent users | 300 | brief |
| Concurrent participants | 10,000 per regional stack | my target (brief says "thousands"); Q12 |
| Dashboard users | 300 per regional stack | brief; Q12 |
| Large collections at once | 3 per regional stack | ASSUMED, Q12 |
| Answer pace | one answer per 20s per participant | ASSUMED, Q11 |
| Share in AI questions | 20% | ASSUMED, Q11 |
| Burst | 3x the average rate | ASSUMED, Q11. Applied to rates; concurrency stays 10,000. |
| Cloud | Azure | my choice |

## Capacity (per regional stack)
Sizing inputs below are ASSUMED and unbenchmarked. They exist to expose risk, not to promise performance.

### Writes
| Figure | Arithmetic | Result |
| --- | --- | --- |
| Answers, average | 10,000 / 20s | 500/s DERIVED |
| Answers, burst | 500 x 3 | 1,500/s DERIVED |

### AI
| Figure | Arithmetic | Result |
| --- | --- | --- |
| Participants in AI questions | 10,000 x 20% | 2,000 DERIVED |
| AI turns, average | 2,000 / 20s | 100/s DERIVED |
| Reply length | why: rough figure for a short spoken-style question | 10s ASSUMED |
| AI turns, upper limit | 2,000 / 10s, since each participant has one reply at a time | 200/s DERIVED |
| Open streams, average | 100 x 10s | 1,000 DERIVED |
| Open streams, upper limit | one per participant | 2,000 DERIVED |
| Tokens per turn | why: round figure for prompt, history and reply | 2,000 ASSUMED |
| Rough interview load | 100 x 2,000 x 60s | about 12M per minute, up to double at peak. ASSUMED, an order of magnitude only. |

If the Derin engine is reused (Q5), this load may sit outside these apps. Dashboard chat and summaries are not included. A burst of 3x does not apply to AI turns, because the upper limit is reached first. This replaces the earlier 38.7M figure, which contradicted the reply length. Detailed token sizing was dropped on my instruction, since the brief doesn't ask for it.

### Connection setup cost (ADR 0002)
Round trips per new connection (3 to 5) come from Microsoft's Front Door page. Extra round trip times are ASSUMED, as rough distances, and not measured.

| Path | Extra round trip | Arithmetic | Extra per new connection |
| --- | --- | --- | --- |
| Nairobi to South Africa North | 100 ms ASSUMED | 3 to 5 x 100 ms | 300 to 500 ms DERIVED |
| Nairobi to Sweden Central | 200 ms ASSUMED | 3 to 5 x 200 ms | 600 to 1,000 ms DERIVED |

### Gateway settings (ADR 0002)
All ASSUMED, as starting values to tune in testing: 90s backend timeout on stream and long-poll paths, 15s heartbeat, 25s long-poll hold, 5s wait for the first event.

### AI failure handling (ADR 0002)
All ASSUMED, as starting values to tune in testing.

| Figure | Value |
| --- | --- |
| Slow: no first token | 5s |
| Slow: reply unfinished | 30s |
| Participant message shown | 10s |
| Model call stopped | 60s |
| Circuit breaker opens | half of calls failing over 30s |
| Probe while open | every 30s |
| Turn marked unanswered | 5 minutes |

### Connections and tokens
| Figure | Arithmetic | Result |
| --- | --- | --- |
| Open streams, worst case | 2,000 participants + 300 dashboard users in chat | 2,300 DERIVED |
| Persistent gateway connections | 10,000 phones + 300 dashboards, each reusing one | 10,300 DERIVED |
| Gateway capacity units for those | 10,300 / 2,500 per unit | about 5 DERIVED |
| Long-poll requests if streams fail for all | one completed poll per turn | 100/s average, 200/s upper limit DERIVED; excludes chat |
| Participant token renewals | 10,000 / 900s | 11/s DERIVED |
| Redaction rate | one text per turn | 100 to 200/s DERIVED; Azure Language quota not checked |
| Dashboard token renewals | 300 users / 300s | 1/s DERIVED |

### Storage (one large collection)
| Figure | Arithmetic | Result |
| --- | --- | --- |
| Read row | | 1 KB ASSUMED |
| Read table | 500k x 1 KB | 500 MB DERIVED |
| Narrow row, with overhead | | 100 B ASSUMED |
| Narrow index entry | | 50 B ASSUMED |
| Write table | 40M x 100 B | 4 GB DERIVED |
| Write table indexes | Unique key and transaction ID: 2 x 40M x 50 B | 4 GB DERIVED |
| Transcript per response | | 10 KB ASSUMED |
| Transcripts, raw and redacted | 2 x 500k x 10 KB | 10 GB DERIVED, until raw text is deleted |
| Read table at fill factor 70 | 500 MB / 0.7 | about 714 MB DERIVED; about 2.1 GB for three |
| Total | 0.7 + 4 + 4 + 10 | about 18.7 GB DERIVED |
| Three large collections | 18.7 x 3 | about 56 GB DERIVED, a floor |

Excludes read table indexes, full-text index, turn records, chat history, audit, rollups, dead rows, WAL and multiple-choice extra rows. The 100 B write row now holds more fields, so it may be tight. WAL per save may be nearer 0.25 KB with the second index.

Row sizes are rough estimates from field counts: they have no measurement behind them, and T1 and T8 replace them.

### Total data written (ADR 0001)
The earlier 40 GB and 44 GB were row bytes written over a collection's life. They were not final storage, which is 0.5 GB and 6.5 GB. They also left out WAL, indexes and rewrites. This table replaces them, and uses WAL because every byte also reaches the replica.

| Input | Value | Source |
| --- | --- | --- |
| Final answers per collection | 40M | brief |
| WAL per save, write table row plus index | 0.2 KB | ASSUMED |
| WAL per read row update, row moves to a new page | 1.05 KB | ASSUMED, high case |
| WAL per one-row update, high case, with per-question versions | 1.05 + 0.16 = 1.21 KB | ASSUMED; versions replace the narrow table's duplicate protection |
| WAL per read row update, same page | 0.2 KB | ASSUMED, low case. Postgres can log only the changed part. |
| s = saves per final answer | 1, 1.5, 3, 5 | ASSUMED scenarios |

Formulas: one row per response = 40M x s x row cost. Narrow plus read table = (40M x s x 0.2 KB) + (40M x row cost), because autosave drafts are not projected to the read table.

| Saves per answer | One row, high | Narrow plus read, high | One row, low | Narrow plus read, low |
| --- | --- | --- | --- | --- |
| 1 | 48.4 GB | 50 GB | 8 GB | 16 GB |
| 1.5 | 72.6 GB | 54 GB | 12 GB | 20 GB |
| 3 | 145.2 GB | 66 GB | 24 GB | 32 GB |
| 5 | 242 GB | 82 GB | 40 GB | 48 GB |

All DERIVED, per collection. Three large collections triple these.

| Figure | Arithmetic | Result |
| --- | --- | --- |
| Break-even, high case | 48.4s = 8s + 42 | s = 1.04 DERIVED |
| Break-even, low case | 8s = 8s + 8 | none; the narrow design stays 8 GB higher DERIVED |
| Extra WAL, narrow design, average, at most | 500 x 0.2 KB | 0.1 MB/s DERIVED |
| Extra WAL, narrow design, burst, at most | 1,500 x 0.2 KB | 0.3 MB/s DERIVED |
| Total WAL, narrow design, burst, high case | 1,500 x 1.25 KB | 1.9 MB/s DERIVED |
| Time to write 40M, three collections sharing the average rate | 40M / (500 / 3) | 240,000s, about 67 hours DERIVED |
| Dead row versions, one row design | 40M x s | grows with autosave DERIVED |
| Dead row versions, read table | 40M | fixed DERIVED |
| Answers per response in a 2s batch, burst | 2s / (20s / 3) | 0.3 DERIVED, so batching merges almost nothing |
| Draft storage if kept in the write table | 40M x s x 150 B | 6 GB x s DERIVED |

Not included: full-page images after checkpoints, vacuum, heap page writes and rollup updates. These need measuring. Random response IDs would scatter index writes and add many full-page images, so IDs should be time-ordered.

### Dashboard
| Figure | Arithmetic | Result | Why, if ASSUMED |
| --- | --- | --- | --- |
| Live polls | 300 / 5s (my decision) | 60/s DERIVED, served from Redis | |
| Filtered view refresh or action | one per 30s per user | ASSUMED | Open views refresh on a timer, and actions are less frequent |
| Ad-hoc queries | 300 / 30s | 10/s DERIVED | |
| Cold scan of the largest collection | 1s on 4 workers | 4 core-seconds ASSUMED | 500 MB in memory; to be benchmarked |
| Dashboard queries, no cache | 10 x 4 | 40 cores DERIVED | |
| Chat questions | 300 users / 120s | 2.5/s DERIVED, at one question per 2 minutes ASSUMED | Occasional use during analysis |
| Chat tool queries | 2.5 x 3 per question | 7.5/s DERIVED, at 3 ASSUMED | Typical, below the limit of 5 |
| Chat core demand | 7.5 x 4 | 30 cores DERIVED | |
| Total core demand, no cache | 40 + 30 | 70 cores DERIVED | Excludes exports, search and reconciliation |
| Replica size | | 32 vCores ASSUMED | A size that fits several parallel scans at once |
| Replicas | 70 / 32 = 2.2, rounded up | 3 DERIVED | Sized cache-off, so a Redis outage doesn't overload them |
| Saturation point | 96 cores / 17.5 queries/s | 5.5 core-seconds per query DERIVED | Escalation trigger is set at 5 |
| Shared cache hit rate | | not used in sizing | Unproven; a hit needs another user with the same filter in the same bucket |
| Core demand if refreshed every 10s | 300 / 10s x 4 = 120, against 40 | 80 cores more DERIVED | Rejected alternative, Q37 |
| Age of an open ad-hoc filtered view | 3s + 4s lag + 30s bucket + 30s on screen | about 67s DERIVED | Q37 |
| Pinned sets per collection | | 10 ASSUMED | Enough for standing segments, and bounds projector work |
| Pinned counters per large collection | 10 sets x 80 questions x 5 options (ASSUMED) x 2 states | 8,000 DERIVED | |
| Pinned counter writes, worst case | 8,000 x 3 collections x 100 B / 2s | 1.2 MB/s DERIVED, at 100 B ASSUMED | Every counter touched in every batch; real load is lower |
| Pinned set age | 2s + 1s + 5s poll | about 8s DERIVED | |
| Full view cost | 500k x 80 | 40M counter updates DERIVED | 4 core-seconds may not hold; benchmark 8 and 80 charts |
| Projector batch interval | | 2s ASSUMED | Short enough for the 10s target |
| Projector batch run time at burst | 1,500 x 2s = 3,000 rows | 1s ASSUMED | To be benchmarked |
| Live rollup age, worst case | 2s + 1s + 5s poll | 8s DERIVED | |
| Replica lag allowed, cold query | 10s - 2s - 1s - 3s | 4s DERIVED | |
| Cross-tab pairs per collection | 80 x 79 / 2 | 3,160 DERIVED | |

### Other sizing
All ASSUMED, with reasons, except where marked. Cost is not estimated (risk R19).

| Figure | Value | Why or arithmetic |
| --- | --- | --- |
| Primary | 16 vCores | Writes are about 2 MB/s of WAL at burst, plus turn records; to be benchmarked |
| PgBouncer pool | 48 on the primary, 96 per replica | 3 x vCores, within Microsoft's 2 to 5 |
| API request rate, burst | about 1,770/s DERIVED | 1,500 saves + 200 turns + 60 polls + 10 queries |
| NestJS throughput | 400 requests/s per 2-vCPU instance | Rough figure for small JSON requests |
| NestJS instances | at least 6 DERIVED, a floor | 1,770 / 400 = 4.4, rounded to 5, plus 1 spare. Excludes up to 2,000 model calls and 2,300 streams in these processes, which aren't measured. |
| Projector workers | 1 active, 1 standby | One owner by lock |
| Export workers | 2 | Keeps exports from slowing queries |
| Redis memory | under 1 GB | 10/s x 20 KB per result x 60s = 12 MB DERIVED, plus rollups and counters |
| Model call stop | 60s | Well above a normal reply |
| Turn lease | 75s | Above the model stop |
| Sweeper | every 30s | Recovers a dead turn within about 2 minutes |
| Participant token | 15 minutes | Long enough for short gaps, and short enough to limit a leak |
| Statement timeout | 10s | Limits replica bloat, and bad queries |
| Query limits | 2 per user, 5 per organization | Stops one user taking a replica's capacity |
| Fill factor | 70 | Room for same-page updates |
| Rounding | nearest 5, under 100 | Common disclosure practice; policy choice, no cited standard |
| Numeric bins | up to 20 | Enough detail for a histogram |
| Cross-tab limit | 30 x 30 | Readable, and bounds query cost |
| Chat tool calls | 5 per question | Bounds cost and comparison attempts |
| Summary start | 100 responses | Small surveys could be too specific |
| Answer save target | 500 ms p95 | A save the phone can wait for |
| Quota thresholds | 70%, 80%, 90% | Starts shedding before Azure refuses |
| Export cell size | 4 B | Short coded values |
| Coded export size | 500k x 80 x 4 B = 160 MB DERIVED | |
| Export SAS | 15 minutes | Long enough to start a download |
| Export file life | 24 hours | Limits time on disk |
| Raw text | collection close + 30 days | Allows redaction reruns |
| Chat history | 30 days | Short, to limit stored context |
| Audit | 1 year | Time to investigate a complaint |
| First-load JS | 150 KB | Slow 3G budget, from Q14 |
| Retry attempts | 3 | Enough for a transient failure; the 5-minute cutoff still wins |
| Rollup rebuild through projector | 500k x 80 / 3,000 rows/s, about 3.7 hours DERIVED | Why a rebuild uses bulk load |
| Postgres check without Redis | every 2s | Replies still arrive promptly |
| Feature cache | 30s | Limits lookups; switches can wait that long |
| Transcripts per page | 20 | Readable, and slows bulk copying |
| Transcript reads | 200 per user per day | Above normal reading, below bulk copying |
| Chat turn limit | 75s | Below the 90s gateway timeout |
| Summary trigger | 10% growth, at most hourly | Fresh enough, without constant model calls |
| Invite link | 30 days, or end of fieldwork | Fits a typical field period |
| Log retention | 30 days | Enough to debug, and limits exposure |
| Collection treated as closed | 2 years | So retention always ends |
| Session freshness for exports | 15 minutes | Recent proof of identity for the riskiest action |
| 429s before delaying replies | 30s | Filters out brief spikes |

### To check
- "40M rows" holds only for one row per answer with every question answered. Multiple choice can add rows, skipped questions remove them, and transcripts are not counted (risk R21).

## Targets
| Target | Value | Source |
| --- | --- | --- |
| Rollup reads | under 1s p95 | ASSUMED (my decision, Q4) |
| Ad-hoc filters | under 3s | ASSUMED (my decision, Q4) |
| Data age | at most 10s | ASSUMED (my decision, Q4) |
| Minimum group size | 10 | ASSUMED policy choice, no cited legal standard (my decision, Q7) |
| Dashboard live poll | 5s | My decision |

## Azure sources (checked 2026-11-08, Microsoft Learn)
"Full" means I read the whole page or the relevant section. "Excerpt" means I saw only the search excerpt.

### Regions
| Fact | Page | Read |
| --- | --- | --- |
| Africa regions are South Africa North and South Africa West (restricted access). No region is listed in Kenya. | https://learn.microsoft.com/azure/reliability/regions-list | Full |
| Each region sits in one geography, which is a fixed data residency boundary | https://learn.microsoft.com/azure/reliability/regions-overview | Excerpt |

### Front Door
| Fact | Page | Read |
| --- | --- | --- |
| Terminates TLS and decrypts at the edge, then re-encrypts to the origin | https://learn.microsoft.com/azure/frontdoor/end-to-end-tls | Full |
| Global service; the only location chosen is for resource group metadata | https://learn.microsoft.com/azure/frontdoor/front-door-faq | Excerpt |
| Edge locations include Nairobi, Lagos, Cairo, Rabat, Cape Town and Isando | https://learn.microsoft.com/azure/frontdoor/edge-locations-by-region | Full |
| Traffic goes to the closest edge and moves to the next closest if that is unhealthy (page covers the classic tier) | https://learn.microsoft.com/azure/frontdoor/front-door-traffic-acceleration | Excerpt |
| Authenticated API routes should have caching disabled | https://learn.microsoft.com/azure/frontdoor/front-door-caching | Excerpt |
| Non-regional edge caching services should not be used for sensitive content you want to keep in your geography | https://learn.microsoft.com/azure/azure-government/documentation-government-overview-wwps | Excerpt |

The earlier ASSUMED in-transit reading is no longer needed, because API traffic no longer passes through Front Door (Q10).

### Models
| Fact | Page | Read |
| --- | --- | --- |
| Global may process in any region. Data Zone covers US, EU or APAC. Standard/Regional processes in the deployment's region. | https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability | Full for the Standard/Regional and Regional Provisioned sections |
| South Africa North Standard/Regional: text-embedding-3-large and text-embedding-ada-002 only | same page, Standard/Regional | Full |
| South Africa North Regional Provisioned: gpt-4o (2024-05-13) and gpt-5.1 | same page, Regional Provisioned, Azure OpenAI table | Full |
| Sweden Central Standard/Regional includes gpt-4.1 and gpt-5.1 | same page, Standard/Regional | Full |
| Processing stays in the geography but may move between its regions. Batch is a Global type. Abuse-monitoring data is stored in the geography, with possible human review. Responses API stores message history. | https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy | Full |
| Modified abuse monitoring needs an application and approval | https://learn.microsoft.com/azure/foundry/openai/concepts/abuse-monitoring | Excerpt |

ASSUMED: approval for modified abuse monitoring. Not checked: provisioned cost and minimum capacity. Availability changes, so re-check before an ADR relies on it.

### Postgres
| Fact | Page | Read |
| --- | --- | --- |
| Cosmos DB for PostgreSQL is retiring; Elastic Clusters is the Citus path | https://learn.microsoft.com/azure/cosmos-db/postgresql/concepts-benefits-database-postgresql | Excerpt |
| Up to five replicas, updated asynchronously. Lag is usually seconds to minutes, and can reach hours under heavy writes. | https://learn.microsoft.com/azure/postgresql/read-replica/concepts-read-replicas | Excerpt |
| hot_standby_feedback reduces query cancels but can cause table bloat on the primary | https://learn.microsoft.com/azure/postgresql/troubleshoot/troubleshoot-canceling-statement-due-to-conflict-with-recovery | Excerpt |
| Use PgBouncer in transaction mode, starting at 2 to 5 times vCores | https://learn.microsoft.com/azure/postgresql/configure-maintain/concepts-limits | Excerpt |

### Front Door limits (added for Q9)
| Fact | Page | Read |
| --- | --- | --- |
| WAF rate limits count per socket IP, over 1 or 5 minutes. Grouping by ASN or JA4 is Premium only, in preview. | https://learn.microsoft.com/azure/web-application-firewall/afds/waf-front-door-rate-limit | Excerpt |
| Use high rate limit thresholds to avoid blocking legitimate traffic | https://learn.microsoft.com/azure/web-application-firewall/afds/waf-front-door-best-practices | Excerpt |
| Origin response timeout is 16 to 240 seconds | https://learn.microsoft.com/troubleshoot/azure/front-door/troubleshoot-issues | Excerpt |
| WAF logs can hold sensitive values; sensitive data protection reduces this | https://learn.microsoft.com/azure/frontdoor/secure-front-door | Excerpt |
| Private origins need Front Door Premium with Private Link | https://learn.microsoft.com/azure/frontdoor/end-to-end-tls | Full |

| **Front Door does not support Server-Sent Events.** WebSocket is supported, with a 5-minute idle timeout and a 2-hour maximum. The WAF inspects only at connection setup. 1M connections per profile. | https://learn.microsoft.com/azure/frontdoor/standard-premium/websocket | Full |

Correction: an earlier version of this file said I had found no Front Door page on server-sent events. That page exists and rules them out.

ASSUMED: phones on one mobile carrier can share an IP; this is general network knowledge, not from these pages. Also ASSUMED: WebSockets work on the target phones and networks. Test it first.

### Application Gateway (added for Q9, Q10, Q32)
| Fact | Page | Read |
| --- | --- | --- |
| SSE is supported. Response buffer should be off. Request timeout must exceed the idle time between events. | https://learn.microsoft.com/azure/application-gateway/use-server-sent-events | Full |
| "Response buffering isn't impacted by the WAF." The WAF SKU always buffers requests. Buffers are set per resource, not per listener. Buffering protects against slow clients. Not in the portal. Test before production. | https://learn.microsoft.com/azure/application-gateway/proxy-buffers | Full |
| Request timeout: default 20s, per backend setting, 1 to 86,400s for private backends and up to 240s for external ones. Draining defaults to 30s, and a configuration update ends connections after it. | https://learn.microsoft.com/azure/application-gateway/configuration-http-settings | Full |
| On timeout, v2 tries a second backend pool member | https://learn.microsoft.com/troubleshoot/azure/application-gateway/application-gateway-troubleshooting-502 | Excerpt |
| HTTP/2 frontend idle 180s, not configurable. HTTP/1.1 keep-alive 120s. TCP idle 4 minutes by default, up to 30. | https://learn.microsoft.com/azure/application-gateway/application-gateway-faq | Excerpt |
| New instances take 3 to 5 minutes. Scale-in drains for 5 minutes. Up to 125 instances. | https://learn.microsoft.com/azure/application-gateway/application-gateway-autoscaling-zone-redundant | Excerpt |
| One capacity unit includes 2,500 persistent connections. Fixed cost plus capacity units. | https://learn.microsoft.com/azure/application-gateway/understanding-pricing | Excerpt |
| WAF rate limits can group by client address | https://learn.microsoft.com/azure/web-application-firewall/ag/rate-limiting-configure | Excerpt |
| WAF logs hold matched data in clear by default. Log scrubbing needs CRS 3.2 or DRS 2.1. Microsoft keeps IP addresses internally. | https://learn.microsoft.com/azure/web-application-firewall/ag/waf-sensitive-data-protection | Excerpt |
| WAF doesn't mitigate volumetric network attacks; pair with DDoS Protection | https://learn.microsoft.com/azure/networking/design-guide/web-application-firewall | Excerpt |
| WAF false positives on JSON and free text | https://learn.microsoft.com/troubleshoot/azure/application-gateway/troubleshoot-waf-blocking-legitimate-requests-403 | Excerpt |

Not checked: Application Gateway availability in South Africa North, WAF_v2 compute per request, gateway and DDoS Protection cost. ASSUMED: the WAF doesn't inspect response streams, since I found no page saying it does. The two request timeout descriptions differ, so the 90s setting is chosen to hold under either, and is on the test-first list.

### Dashboard services (added for ADR 0003-0005)
| Fact | Page | Read |
| --- | --- | --- |
| Text PII supports Arabic, Swahili, Somali, Amharic and others. Conversation PII lists only English, French, German and Spanish. | https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/language-support | Excerpt |
| Azure Language lists Sweden Central and South Africa North for some features. | https://learn.microsoft.com/azure/ai-services/language-service/concepts/regional-support | Excerpt, not the PII row |
| PII detection varies by language and locale (answer on Microsoft Q&A, not product docs) | https://learn.microsoft.com/answers/a/2039399 | Excerpt |
| A blocked prompt returns HTTP 400 with `content_filter`, before any streamed event | https://learn.microsoft.com/azure/foundry/agents/how-to/add-hosted-agent-guardrails | Excerpt |
| A filtered reply ends with `finish_reason: content_filter`, naming categories including `self_harm` | https://learn.microsoft.com/azure/foundry/openai/concepts/content-streaming | Excerpt |
| Classifiers are trained on 8 languages; others may vary. Results can be false positive or negative. | https://learn.microsoft.com/azure/foundry/openai/concepts/content-filter-severity-levels | Excerpt |
| User delegation SAS is the recommended SAS. Revocation can be delayed. | https://learn.microsoft.com/rest/api/storageservices/create-user-delegation-sas and https://learn.microsoft.com/azure/storage/blobs/storage-blob-user-delegation-sas-create-cli | Excerpt |
| Lifecycle policies can delete blobs | https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview | Excerpt |
| Replication slots hold WAL, and storage pressure can switch the server to read-only | https://learn.microsoft.com/azure/postgresql/read-replica/concepts-read-replicas | Excerpt |

ASSUMED: Text PII is available in Sweden Central and South Africa North, and processes within the region. Recharts suits shadcn/ui charts. `EventSource` can't set headers. Azure prompt attack detection suits chat. Not checked: Azure Language quota, or Blob log detail for proving downloads.

### Redis
| Fact | Page | Read |
| --- | --- | --- |
| Azure Cache for Redis is retiring; Azure Managed Redis is the replacement | https://learn.microsoft.com/azure/azure-cache-for-redis/retirement-faq | Excerpt |
| Azure Managed Redis tiers are listed for Sweden Central and South Africa North. Regional capacity is not guaranteed. | https://learn.microsoft.com/azure/redis/reserved-pricing | Excerpt |
| High availability is required for the SLA and zone redundancy | https://learn.microsoft.com/azure/redis/plan-reliability-and-durability | Excerpt |

### Quota and prompt caching
| Fact | Page | Read |
| --- | --- | --- |
| Quota tiers, tokens and requests per minute by model and deployment type | https://learn.microsoft.com/azure/foundry/openai/quotas-limits | Excerpt, Tiers 1, 2, 3, 6 |
| Quota is per region, subscription, model and deployment type | https://learn.microsoft.com/azure/foundry/openai/how-to/quota | Excerpt |
| Azure counts max_tokens in its token estimate, and checks requests over 1 or 10 second windows | https://learn.microsoft.com/microsoft-cloud/dev/dev-proxy/concepts/llm-rate-limits | Excerpt |
| Prompt caching lowers latency and cost. Needs 1,024 identical leading tokens. | https://learn.microsoft.com/azure/foundry/openai/how-to/prompt-caching | Excerpt |

ASSUMED: prompt caching does not reduce quota use. The page describes cost and latency, and I found nothing saying it lowers tokens per minute.

## Better Auth sources (checked 2026-11-08, Better Auth MCP, latest docs)
| Fact | Page | Read |
| --- | --- | --- |
| Tokens verify in your own service through JWKS, with no verify call or database check | /docs/plugins/jwt | Full |
| Public key can be cached; re-fetch when an unknown kid arrives | /docs/plugins/jwt | Full |
| Default expiry is 15 minutes, configurable through expirationTime; we set 5 minutes (Q29). Default issuer and audience are the base URL. | /docs/plugins/jwt | Full |
| Default payload is the entire user object; definePayload changes it | /docs/plugins/jwt | Full |
| The plugin is not meant to replace the session | /docs/plugins/jwt | Full |
| Default roles are owner, admin and member. Custom roles use createAccessControl. | /docs/plugins/organization | Access Control section only |
| Session stores the active organization ID | /docs/plugins/organization | Schema section only |

| Session freshness uses `freshAge`, 1 day by default | /docs/concepts/session-management | Excerpt |
| A two-factor plugin exists | /docs/plugins/2fa (linked from /docs/plugins/anonymous) | Link only, page not read |

ASSUMED: the org role can go in a token claim. The docs show definePayload with the user only. Also ASSUMED: analyst and viewer map cleanly to custom roles, and two-factor can be enforced per role in our own code.

## Vendor claims not verified
Inngest, PostHog and Sentry hosting regions. Inngest's storage of step output. Browser support for Background Sync. JS bundle sizes for React, Next.js and Sentry.