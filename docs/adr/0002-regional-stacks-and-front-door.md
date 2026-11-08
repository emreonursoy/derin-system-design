# ADR 0002: One stack per region, a small control plane, Application Gateway for APIs, Front Door for static files

**Status:** Proposed, for my review. **Inputs:** Q1, Q2, Q9, Q10, Q11, Q12, Q17, Q18, Q22, Q23, Q24, Q25, Q27, Q28, Q29, Q30, Q31, Q32, Q33, Q34, Q35. **Risks:** docs/risks.md.

## Context
- Each collection pins to one region. There is no promise of in-country storage (Q1).
- AI runs on Azure-hosted models within the geography, and never silently elsewhere (Q2, Q23).
- All API traffic goes through a regional Application Gateway with WAF. Front Door serves static files only (Q9, Q10).
- AI turns and chat stream over SSE, which Microsoft documents for Application Gateway and states Front Door does not support (Q32).
- We claim no specific law (Q18). This design supports a client's compliance work, but does not assert compliance.

## Decision

### Layout
| Layer | Scope | Holds |
| --- | --- | --- |
| Front Door | Global | Static files only. No API traffic, answers or tokens. |
| Application Gateway, WAF_v2 SKU | One per region | All API traffic. The control plane gets its own listener on the Sweden Central gateway (ASSUMED). |
| Control plane | Sweden Central (Q30) | Dashboard users, organizations, roles, and the collection-to-region map. Better Auth runs here. **No participant data.** Dashboard users' details may leave their region. |
| Regional stack | One per region | NestJS, Postgres and replicas, Azure Managed Redis, Blob Storage for exports, Azure-hosted models, Azure Language for redaction, Inngest functions. All participant data. Next.js runs as static exports only (ADR 0003, 0004), so no Next.js server. |

Stacks share no data, and no query crosses regions. The first stack is sized on Sweden Central as an example. The design treats the region as a variable, deployed from one template.

### Who does what
| Job | Where | Note |
| --- | --- | --- |
| Static delivery | Front Door | The participant shell and dashboard assets. |
| TLS for API traffic | Application Gateway | Decrypts in the region, then re-encrypts to NestJS. |
| Routing | DNS and Application Gateway | One API hostname per region. The gateway routes by path. |
| Finding the region | Invite link or control plane | The invite link carries the regional host. The dashboard gets the region from the control plane. |
| Request inspection, coarse abuse limits | Application Gateway WAF | Per client IP, with high thresholds. |
| Token verification | NestJS | Local, on every request. |
| Per-token rate limits and LLM budgets | NestJS, with counters in regional Redis | Shared counters, so limits stay accurate as instances scale (Q27). |
| Roles, features, suppression | NestJS | In one shared guard and query layer. |

API traffic never reaches Front Door. NestJS also sends `Cache-Control: no-store`, so an API response stays uncached even if routing is misconfigured.

### Application Gateway settings for streams
All from Microsoft's docs (sources in assumptions.md). Our values are ASSUMED, to tune in testing.

**Does turning off response buffering limit the WAF? No.** Microsoft states "Response buffering isn't impacted by the WAF." SSE requires it off. The limits are elsewhere:

| Point | What Microsoft says | Effect on us |
| --- | --- | --- |
| Request buffering | The WAF SKU always buffers the request | A request reaches NestJS only once complete. Fine for small answers and turn requests. |
| Scope | Buffer settings apply to the whole gateway, not per listener | Every response on the gateway is unbuffered, not only streams |
| Slow clients | Buffering is what protects the backend from slow clients | A slow phone can now hold a NestJS connection longer. Keep responses small, and don't send large exports through the gateway. |
| Setup | Not available in the portal; use CLI or a template. Test before production. | Set in the deployment template, and include it in the first test |

The WAF inspects requests, so AI replies aren't inspected on the way out (ASSUMED, as I found no page saying it inspects responses). Replies come from our own model call.

**What the timeouts mean for our streams:**

| Timer | Microsoft's value | Effect | Our setting |
| --- | --- | --- | --- |
| Backend request timeout | Default 20s. Up to 86,400s for private backends. Set per backend setting. | The SSE page says it must exceed the gap between events. The general page says it's the wait for a response. **The default would also kill a 25s long-poll.** | 90s on stream and long-poll paths, above the 60s model stop, so it holds under either reading. 20s elsewhere. |
| Retry on timeout | v2 tries a second backend pool member | A slow request could arrive twice | Same turn ID never starts a second model call. Streams send their first event at once. |
| HTTP/2 frontend idle | 180s, not configurable | Silent connections close | Heartbeat every 15s during a stream. A phone idle for over 180s pays connection setup again. |
| HTTP/1.1 keep-alive and TCP idle | 120s; TCP 4 minutes by default, up to 30 | Same, on older connections | Heartbeat covers these |
| Connection draining | Default 30s. A configuration update ends ongoing connections after this timeout. | Gateway changes can cut streams | Set above 90s. A cut stream falls back to the saved reply. |
| Autoscaling | New instances take 3 to 5 minutes. Scale-in drains for 5 minutes. | A burst arrives before new capacity | Set minimum instances for peak load |
| Persistent connections | 2,500 per capacity unit | 10,000 phones plus 300 dashboards, each reusing one connection, are about 5 units (DERIVED). Connections aren't the likely limit. | WAF compute is unmeasured, and is the number to test |

Streams close when the token expires (ADR 0003), so a removed user can't keep one open past the 5-minute limit.

### Cost: slower connection setup
Front Door shortens connection setup by terminating close to the phone. API traffic now loses that. Microsoft says setting up a connection takes 3 to 5 round trips.

| Path (extra round trip is ASSUMED, not measured) | Extra time per new connection (DERIVED) |
| --- | --- |
| Nairobi to South Africa North, 100 ms | 300 to 500 ms |
| Nairobi to Sweden Central, 200 ms | 600 to 1,000 ms |

- **When it's paid:** the first API call, after each drop, and after 180s idle. Bad networks drop often, so participants on them pay it most.
- **Also lost:** retransmissions now cross the whole path, not only the last mile. Not quantified.
- **Not affected:** the static shell's first load, which still comes from the edge.
- **Mitigations:** reuse one HTTP/2 connection, batch outbox writes, and use TLS resumption (ASSUMED available).
- **What it buys:** SSE, and participant data decrypted only in the region.

### Dashboard tokens (checked with the Better Auth MCP)
| Point | Finding | Design |
| --- | --- | --- |
| Local verification | Documented: JWKS verification with no verify call or database check | Regional APIs cache the public key, and re-fetch on an unknown `kid` |
| Expiry | Default 15 minutes, configurable through `expirationTime` | Set to 5 minutes (Q29). **A removed user keeps access for up to 5 minutes**, and we say so openly. About 1 renewal per second (DERIVED). |
| Row-level export | Our own rule | Also checks the control plane live. **Blocked if the control plane is down** (Q29). |
| Payload | Default is the entire user object | Use `definePayload` for a minimal claim set, with no email in tokens |
| Audience | Defaults to the base URL. The docs show it as one static setting. | A token for one region should fail in another. ASSUMED: a `regions` claim that NestJS checks, because a per-token audience isn't documented. **Test first**, with the role claim. |
| Use | Documented as not replacing the session | Session stays at the control plane. The JWT is only for regional APIs. |
| Org role in the token | **Not documented.** The docs show `definePayload` with the user only | ASSUMED that the role sits in the token, with a regional copy pushed from the control plane as the fallback. **Test first** (Q28). |
| Roles | Defaults are owner, admin, member. Custom roles use `createAccessControl`. | Define admin, analyst and viewer. Mapping ASSUMED. |
| Per-collection features | I found no built-in feature | Our own table in the regional stack, checked by NestJS |

Participants sit outside Better Auth (Q25), with tokens issued by the regional stack and a separate signing key.

| Point | Decision |
| --- | --- |
| Org in the token | The role is only valid for the org in the same token. The guard checks it against the collection's org (ADR 0004). |
| Regional role copy (fallback) | Has a time limit, and fails closed when stale. |
| Signing key | One key can reach every region, so "stacks share no data" doesn't mean the regions are isolated from a stolen key. Stored in the control plane's key store (ASSUMED), and on the risk list. |
| Collection names | The control plane stores opaque IDs, since a name like "HIV study, Nairobi" is health context. Names stay in the region. |

### Models
- At collection creation, check that the region has a suitable model. If not, **the collection cannot have AI questions in that region** (Q22).
- Allowed: Standard/Regional and Regional Provisioned. Excluded: Global, Data Zone and Batch, because Microsoft documents Batch as a Global type.
- Conversation state stays in our Postgres. We do not use stateful model features such as the Responses API's stored history.
- We apply for modified abuse monitoring. Approval is ASSUMED, and without it Microsoft may store flagged prompts in the geography for human review.

| Region | Chat on Standard/Regional | Chat on Regional Provisioned |
| --- | --- | --- |
| Sweden Central | Yes, including gpt-5.1 and gpt-4.1-mini | Not checked |
| South Africa North | **None** | gpt-4o (2024-05-13), gpt-5.1 |

**Quota decides suitability.** gpt-5.1 Standard tops out at 4M tokens per minute at Tier 6, against the rough load of 12M. That is about 667 AI participants, not 2,000 (DERIVED), and degradation would be the normal state. gpt-4.1-mini Standard reaches 225M at Tier 6. A model is suitable only if quota covers the rough load, and our tier isn't known.

**Abuse monitoring:** if modified monitoring is refused, flagged prompts may be stored in the geography for human review. Health surveys may flag often. The participant notice must say this unless approval is confirmed (ADR 0003).

### When the AI is slow, down or over quota
**Rough load, ASSUMED:** about 12M tokens per minute for interviews, perhaps double at peak. This is an order of magnitude, not a sizing. It says quota will matter, so the design must behave well when it runs out.

**Rules that hold in all three cases:**
- Deterministic questions and answer saves use no LLM, so they keep working.
- A participant's text is saved before any model call, so no answer is lost (ADR 0003).
- Keep interviews running longest. A participant's time can't be recovered, and a researcher can wait.
- Data never moves to another geography or a Global deployment to get a reply (Q2).
- Every affected turn is marked in the data, so results stay honest.

All timings below are ASSUMED, to be tuned in testing.

| | Slow | Down | Over quota |
| --- | --- | --- | --- |
| **How we know** | No first token within 5s, or reply unfinished at 30s | Half of calls failing over 30s, which opens a circuit breaker | Azure refuses with 429, or the Redis budget is nearly used |
| **Server does** | No parallel calls for a turn, each stopped at 60s. Sheds dashboard AI first. | Stops calling the model, and probes every 30s. Queues turns. | Waits as long as Azure asks, then follows the order below |
| **Participant in an AI turn sees** | Typing indicator. At 10s: "Still working, your answer is saved." | "Your answer is saved, and the reply is coming." They can close the app and return with the same link. | Same as slow, then same as down |
| **Participant starting an AI interview** | Starts normally | Waits at the start, with an option to return later | Waits at the start, with an option to return later |
| **Dashboard** | Charts unaffected. Chat is slower, then limited. | Charts unaffected. Chat unavailable. Last summary, with an "as of" time. | Charts unaffected. Chat limited or paused. Summary paused. |
| **Marked in the data** | Nothing unless the call is stopped | Turn delayed, or unanswered | Turn degraded, delayed or unanswered |

**If a turn stays unanswered for 5 minutes,** it is marked unanswered, and the survey moves to the next question. An AI question reached during an outage is marked not asked. Researchers see how many turns were affected, and can filter them out.

**Order of degradation at quota,** by share of the per-minute budget used, counted in Redis (thresholds ASSUMED):

| Step | Trigger | Action |
| --- | --- | --- |
| 1 | 70% | Pause executive summary refresh. |
| 2 | 80% | Limit dashboard chat per organization, then pause it. |
| 3 | 90% | Hold new AI interviews at the start. Sessions in progress continue. |
| 4 | Azure returns any 429 | Use shorter context and fewer follow-ups in sessions in progress. |
| 5 | 429s continue for 30s | Delay replies. |

**Recovery:**
- Allow traffic back gradually, not all at once.
- Serve waiting turns oldest first, with random spacing, so the backlog doesn't cause a second overload.
- Resume new interviews next, and dashboard AI last.

**Other cases:**
- **Derin engine (Q5):** the interface must report the same states: slow, unavailable, over limit and refused.
- **Region-wide model outage:** AI questions stop in that region until it recovers. There is no failover outside the geography. This is the cost of Q2, and we state it openly.
- **Process death and content filter refusals:** handled by the turn states in ADR 0003. Refusals are not outages, and are never retried.

### If Redis is down
Losing Redis loses no participant data.

| Function | Behaviour without Redis |
| --- | --- |
| Per-token rate limits | Saves continue, protected only by the WAF. |
| LLM budgets | Dashboard AI pauses. Interviews continue, relying on Azure's limits and backing off when refused. |
| Live rollups | Polls read the replica, one read per collection per instance every 5s, through the same suppression function. Data is about 13s old plus lag, over the 10s target (DERIVED). |
| Query cache | Queries run on replicas with no cache. Replicas are sized for this. |
| Query concurrency limits | A smaller per-instance limit applies. |
| Waking waiting replies | Each instance checks Postgres every 2s for its waiting turns as one grouped query (ASSUMED), about 3 reads/s (DERIVED), not one per stream. |
| Small-segment release state | Can't be rebuilt from Redis alone. Small segments show "too few responses" until new state builds. They never release early. |

### Internal access and logs
| Point | Decision |
| --- | --- |
| Network | Postgres, Redis, Blob Storage and Azure Language on private endpoints inside the regional network, with NestJS as the only caller (ASSUMED available in each region). NestJS hosting is ASSUMED to be Azure Container Apps, as a private backend for the gateway. |
| Database roles | Separate roles for interview, projector, query, redaction and export. No replica grant on raw text. |
| Staff access | No standing access to participant data. One break-glass role, time-limited and logged. |
| Gateway logs | Hold IP, time and path, which can link a person to a health survey. Paths carry opaque IDs. Kept 30 days (ASSUMED), in the region. |
| Application and database logs | No parameter values. Errors are sanitized before reaching Sentry or Inngest. |
| Inngest | One environment per region, with signed calls. Only batch or opaque job IDs, so it never sees a per-participant timeline. Hosting and storage remain unverified. |

## Privacy and residency
| Component | Impact |
| --- | --- |
| Front Door | Serves generic static files. Sees no answers or tokens, which removes the earlier ASSUMED in-transit reading. |
| Application Gateway | Decrypts and inspects participant data inside the region. |
| WAF logs | Microsoft logs matched request data in clear by default. Enable log scrubbing, which needs OWASP CRS 3.2 or DRS 2.1. Microsoft still keeps IP addresses in internal logs. Logs stay in a regional workspace, with limited access. |
| Control plane | Stored in Sweden Central, so **dashboard users' names and emails may leave their region**. No health data. |
| Redis | In the regional stack, with no geo-replication. Suppressed aggregates, counters and opaque IDs only (Q34). Never answers, transcripts, prompts or replies. |
| Blob Storage | Export files, in the regional stack, zone-redundant with no geo-replication (ADR 0004). |
| Azure Language | Processes transcript text for redaction, in a regional resource (ADR 0004). |
| JWT | Pseudonymous user ID, org and role only. Sent to the collection's region. |
| Regional Postgres | Holds all participant data. Backup and replica copies stay in the geography. |
| Models | Processing stays in the geography, and may move between its regions. |
| Inngest | Events and step results carry IDs only, whatever its storage behaviour turns out to be. Hosting region unverified. |
| PostHog | Dashboard only. Removed from the participant app. Replay off. Hosting region unverified. |
| Sentry | An allowlist strips URLs, breadcrumbs and IP addresses. No replay. Hosting region unverified. |

## Consequences and risks
- **Gateway limits stay coarse.** The WAF can count per client IP. Phones on one carrier may share an IP (ASSUMED), so a low threshold could block real participants during a survey. Thresholds stay high, and NestJS with Redis carries the real limits.
- **Edge protection is lost for APIs.** The regional public IP takes traffic directly. Microsoft says WAF doesn't stop volumetric network attacks, and recommends pairing it with DDoS Protection. This is on the risk list (Q36).
- **WAF can block real answers.** Microsoft documents false positives on JSON and free text with SQL-like or HTML-like content, and participants write free text. Enable log scrubbing, tune rules on test data, then run in prevention. A blocked answer stays in the outbox, and we alert on it, so it isn't lost silently.
- **More to run and pay for.** Each region adds a gateway, with a fixed cost plus capacity units. Cost and availability in South Africa North are not checked.
- **Redis is a new dependency in every region.** If it fails, saves continue with no per-token limit, and queries lose their cache. Its data can be rebuilt, so persistence isn't needed. Regional capacity isn't guaranteed, and cost isn't checked.
- **The AI will sometimes be unavailable.** Quota is per region, subscription and model, and data can't leave the geography, so there is no overflow elsewhere. Our tier, and whether a given model is good enough, are not known (Q35). Provisioned capacity in South Africa North is not priced.
- **Stale roles.** A removed user keeps access for up to 5 minutes, except for row-level export.
- **Control plane outage.** Dashboards fail within 5 minutes, sooner than with 15-minute tokens, and row-level export fails at once. Participants are unaffected, since they never use it.
- **Each region adds operations work**, even from one template. A small team should run few regions.
- **Fails closed, at a price.** A client on South Africa North either pays for provisioned capacity, pins elsewhere, or runs without AI questions.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| One shared stack with a region column | Residency would rest on every query being correct, and a failure stops interviews and dashboards for all clients. The control plane is also shared, but its failure stops only dashboards, and it holds no participant data. |
| **WebSocket through Front Door** | Documented as supported. It keeps fast connection setup and edge protection, and it was the earlier proposal. Rejected because I chose SSE (Q32): participant data would still be decrypted at the edge, which rests on an ASSUMED residency reading, sockets add server state, and the WAF checks only the connection setup. |
| SSE through Front Door | Microsoft states Front Door doesn't support it. |
| Front Door in front of Application Gateway | Brings back edge decryption, and SSE still wouldn't work. |
| No gateway, Front Door plus NestJS | My earlier Q9 answer. Leaves no documented path for SSE. |
| API Management as the gateway | Not evaluated in depth. Application Gateway has documented SSE support and a WAF, which is what we need. |
| A second gateway only for streams | Would keep response buffering on for other traffic, but adds a second fixed cost. Reconsider if testing shows slow clients holding too many backend connections. |
| Separate accounts per region | Clients with collections in two regions would need two logins. |
| Calling the control plane on every request | Adds latency and an outage dependency across regions. Used only for row-level export (Q29). |
| Per-instance rate limit counters | Limits drift as instances scale, and LLM budgets need to be accurate (Q27). |
| Postgres for rate limit counters | Adds high-churn writes to the database we are trying to protect. |
| Azure Cache for Redis | Retiring. Azure Managed Redis is the replacement. |
| Failing over to another geography when the AI is down | Would keep interviews running, but breaks Q2. |
| Canned questions when the AI is down | Would mix scripted and AI answers in the results without researchers choosing it. |
| Blocking the whole survey during an outage | Wastes the participant's time, when deterministic questions still work. |
| Retrying slow calls in parallel | Adds load exactly when the service is struggling. |
| Global or Data Zone models | Could process outside the geography. Would solve model availability by breaking Q2. |
