# ADR 0002: Deployment, regional boundaries and ingress

**Status:** Accepted, frozen for submission. Nothing is built or benchmarked. **Class:** Protecting participant data and defined failure behaviour are Necessary. Per-region stacks, specific regions, the WAF and Application Gateway are Assumed. **Inputs:** Q1, Q9, Q10, Q17, Q18, Q23, Q24, Q25, Q27, Q28, Q29, Q30, Q32, Q34, Q36, Q39, Q40, Q41, Q47. **Risks:** docs/risks.md.

## Context
- The brief names no region, jurisdiction or customer contract. "Data residency by region" is my own requirement, so it is Assumed.
- A region is a location. A geography is the wider boundary Microsoft commits to. We promise neither until each service is checked (Q47).
- We claim no specific law (Q18). This design supports a client's compliance work, and doesn't assert compliance.

## Decision

### Deployment model
One TypeScript codebase, in one monorepo. Managed Azure services. No Kubernetes, and no microservices.

| Part | What it runs | Scales on |
| --- | --- | --- |
| **API deployment** (NestJS on Azure Container Apps) | REST, guard, query layer, SSE streams, and live calls to the AI service | Concurrent requests |
| **Background worker deployment** (same codebase) | Projector, redaction, exports, turn recovery, background jobs | Its own settings |
| Static apps | Next.js static exports on Front Door. No Next.js server | Not applicable |

**Why live AI calls stay in the API:** the POST that starts a turn is also the stream, so the instance doing the work is the one the browser is connected to. That needs no routing, token relay or session affinity, and no reply text passes through Redis. Inference runs in the AI service, so these calls mainly wait on the network.

**A dedicated AI deployment is added only if** the confirmed AI integration requires it, or measurement shows open streams slowing answer saves (T3, Q41).

### Layout
| Layer | Scope | Holds |
| --- | --- | --- |
| Front Door | Global | Static files only. No API traffic, answers or tokens. |
| Control plane | One home region (Sweden Central is an ASSUMED example) | Better Auth, dashboard users, organizations, members with roles, and the collection-to-region map. **No participant data.** Dashboard users' details may leave their region (Q30). Reached through its own gateway listener (ASSUMED), and also holds each org's export approval setting (ADR 0004). |
| Regional stack | One per region. **Deploy one first** (Q47) | Ingress, API, worker, Postgres and read replica, Redis, Blob Storage for exports and the withdrawal log, redaction service. All participant data we store. |

- Each collection pins to one region (Q1). Stacks share no data, and no query crosses regions.
- Region is a template variable, so a second stack is an operations cost, not a redesign. Add one only when a contract needs it.
- The AI service sits behind an interface (ADR 0003, ADR 0005). Its location is part of the integration contract.

### Who does what
| Job | Where |
| --- | --- |
| Static delivery | Front Door |
| TLS for API traffic, request inspection, coarse limits per IP | Application Gateway with WAF, in the region |
| Finding the region | The invite link carries the regional host. The dashboard asks the control plane. |
| Token verification, per-token rate limits | NestJS, with counters in regional Redis (Q27) |
| Org, role, feature, suppression | NestJS, in one guard and query layer |

API responses carry `Cache-Control: no-store`.

### Ingress: kept for now, assumed
**Application Gateway with WAF stays for now. Its necessity and its SSE behaviour are assumptions to validate (Q40).** It isn't the only possible production architecture. App code doesn't depend on it, so it can be swapped.

| | Application Gateway with WAF | Container Apps ingress only | Front Door for APIs |
| --- | --- | --- | --- |
| SSE | Documented by Microsoft, untested here | Not confirmed in the pages I read | Microsoft states it isn't supported |
| WAF | Yes | None found. Microsoft's route to a WAF puts Application Gateway in front | Yes |
| Rate limits | Per IP, plus NestJS | IP allow or deny rules, plus NestJS | Per IP, plus NestJS |
| Auth | NestJS either way | NestJS | NestJS |
| Where data is decrypted | In the region | In the region | At a global edge |
| Operations | Internal environment, private DNS, rule tuning | Simplest | Simple, but fails the SSE requirement |
| Cost | Fixed cost plus capacity units, not estimated | Lowest (ASSUMED, not priced) | Not compared |

**Why keep it:** invite and login endpoints are on the public internet, unauthenticated, for a health survey. A WAF is a defensible extra control there.

**What would drop it:** nothing in the brief requires a WAF. If T3 shows SSE problems, or tuning shows too many false positives on free text, Container Apps ingress alone is the next option. That accepts losing the WAF, and needs its own SSE test.

### What streaming needs from ingress
Microsoft documents these for Application Gateway, and **none has been tested here (T3).** Response buffering must be off, which applies to the whole gateway. Backend timeouts must exceed the longest stream, and heartbeats must beat idle limits. Starting values are in docs/assumptions.md. Large exports stay off this path, and download directly from Blob Storage (ADR 0004).

**The cost of skipping the edge:** Front Door shortens connection setup, and API traffic loses that. The estimate is 300 to 1,000 ms extra per new connection from Nairobi (DERIVED from ASSUMED round trips, T6). Bad networks reconnect most. It buys SSE, and decryption only in the region.

### Dashboard tokens (checked with the Better Auth MCP)
| Point | Finding | Design |
| --- | --- | --- |
| Local verification | Documented: JWKS, with no call or database check | Regional APIs cache the public key, and re-fetch on an unknown `kid` |
| Expiry | Default 15 minutes, configurable | 5 minutes (Q29). **A removed user keeps access for up to 5 minutes**, and we say so |
| Revocation | Short expiry only, except where noted | Row-level and transcript exports also check the control plane live, and are blocked if it's down (Q29) |
| Payload | Default is the entire user object | `definePayload`, with a minimal claim set and no email |
| Org role in the token | **Not documented.** The docs show `definePayload` with the user only | ASSUMED, with a regional role copy as the fallback. The copy has a time limit and fails closed when stale. **Test first (T2)** |
| Region in the token | A per-token audience isn't documented | ASSUMED: a `regions` claim that NestJS checks. **T2** |
| Roles | Custom roles use `createAccessControl` | Admin, analyst and viewer. Mapping ASSUMED |

- The session stays at the control plane. The JWT is only for regional APIs, and is held in memory.
- The role is valid only for the org in the same token. The guard checks it against the collection's org.
- One signing key reaches every region. This is on the risk list (R2c).
- The control plane stores opaque collection IDs, since a collection name can be health context.
- Participants sit outside Better Auth (Q25), with tokens from the regional stack and a separate signing key.

### When the AI service is slow, down or over its limit
The AI service's internals, including models, quota and capacity, are outside this design unless the contract says otherwise (Q39). This section covers only what our apps do. All timings are ASSUMED.

**Rules that hold in all cases:**
- Deterministic questions and answer saves don't call AI, but they share API capacity with AI calls (R17).
- A participant's text is saved before any AI call (ADR 0003).
- Keep interviews running longest. A participant's time can't be recovered, and a researcher can wait.
- Our app doesn't reroute data to a different provider or location to get a reply. The service's own routing is a contract item.
- Every affected turn is marked in the data.

| | Slow | Unavailable | Over its limit |
| --- | --- | --- | --- |
| **How we know** | No first token in 5s, or no finish by 30s | The interface reports it, or repeated failures open a circuit breaker | The interface reports it |
| **API does** | One call at a time per turn, stopped at 60s. Retries are limited and never parallel | Stops calling, and probes periodically | Waits as long as it's told, then sheds load |
| **Participant in an AI turn** | "Still working, your answer is saved." | "Your answer is saved, and the reply is coming." They can close the app and return | Same as slow, then unavailable |
| **Starting an AI interview** | Starts normally | Waits, with an option to return later | Waits, with an option to return later |
| **Dashboard** | Charts unaffected. Chat slower | Charts unaffected. Chat unavailable. Last summary with an "as of" time | Charts unaffected. Chat limited. Summary paused |

**What we shed first:** summary refresh, then dashboard chat, then new interviews. Interviews in progress go last. Recovery reverses the order, with waiting turns served oldest first, spaced out so the backlog doesn't overload the service again.

**A turn unanswered for 5 minutes** is marked unanswered, and the survey moves on. An AI question reached during an outage is marked not asked. Researchers see how many turns were affected, and can filter them out.

**If the AI service can't commit to a collection's geography, that collection can't have AI questions** (Q22, Q39).

### Redis
Clear purposes only: rate limits, the query cache and live rollups. No answers, transcripts, prompts or replies. Lifetimes are ASSUMED.

| Use | Lifetime |
| --- | --- |
| Rate limits, including AI request limits per user and org | 1 to 5 minutes |
| Query concurrency counters | Seconds |
| Query cache | 60s |
| Live rollups and data version | Until the next batch |
| Held small-segment values, if that proposed control is adopted | Until released |
| SSE and long-poll wake-ups | Not stored: a turn ID published once |

**If Redis is down,** no participant data is lost.

| Function | Behaviour |
| --- | --- |
| Per-token rate limits | Saves continue, protected only by the WAF and a smaller per-instance limit. |
| AI request limits | Dashboard chat pauses. Interviews continue. |
| Live rollups | Polls read the replica, through the same suppression function, with a server-side read cycle. Data is over the 10s target. |
| Query cache | Queries run with no cache. With one replica, expect "busy" responses. Suppression holds, and freshness and capacity drop. |
| Waking waiting replies | Instances check Postgres on a short interval, as one grouped query. |
| Held small segments | Show "too few responses" until new state builds. They never release early. |
| After recovery | The projector republishes rollups (ADR 0001). |

### If the Postgres primary is down
Zone-redundant high availability is ASSUMED. Its availability in a given region and its failover time aren't checked (R26).

| Function | Behaviour |
| --- | --- |
| Participant answers | Wait in the phone's outbox, and sync on recovery |
| AI turns | Can't start, because input must be saved first. "Waiting for connection" |
| Unfiltered views | Last rollups from Redis, with an "as of" time. The guard's feature lookup reads the replica |
| Anything that must be audited, including filtered views, transcripts and exports | Refused with "busy", because audit writes go to the primary |
| Projector | Stops, and resumes from its committed position |

### Internal access and logs
| Point | Decision |
| --- | --- |
| Network | Postgres, Redis and the redaction service on private endpoints (ASSUMED available in each region). The API runs in an internal Container Apps environment behind the gateway. Blob Storage keeps a public endpoint with anonymous access off, because browsers download through a short-lived link (ADR 0004). |
| Database roles | Separate roles for interview, projector, query, redaction and export. No replica grant on raw text. |
| Staff access | No standing access to participant data. One break-glass role, time-limited and logged. |
| Logs | Gateway logs hold IP, time and path, which can link a person to a health survey. Paths carry opaque IDs. No parameter values in application or database logs. Kept 30 days (ASSUMED), in the region. |

## Privacy and residency
| Component | Impact |
| --- | --- |
| Front Door | Serves generic static files. Sees no answers or tokens. |
| Application Gateway | Decrypts and inspects participant data inside the region. WAF logs hold matched data unless log scrubbing is on, so we enable it. |
| API and worker | In the regional stack. Not designed to store participant data locally. Temp files and crash dumps aren't verified. |
| Control plane | **Dashboard users' names and emails may leave their region.** No health data. |
| Redis | Regional, with no geo-replication. No participant content. |
| Blob Storage | Export files and the withdrawal log (pseudonymous IDs), regional with no geo-replication. A leaked link works from anywhere until it expires (ADR 0004). |
| Redaction service | Processes raw transcript text. Regional resource. Processing location not verified (R10). |
| AI service | **Receives participant and analyst text. Location and retention are unknown until the contract is confirmed** (R6). |
| Regional Postgres | All stored participant data. Backup location not verified (R21a). |
| Inngest | IDs only, in events and step results. Hosting not verified (R11). |
| PostHog | Dashboard only, removed from the participant app. Hosting not verified. |
| Sentry | An allowlist strips URLs, breadcrumbs and IP addresses. No replay. Hosting not verified. |

## Consequences and risks
- **WAF can block real answers.** Microsoft documents false positives on free text. What the participant sees is in ADR 0003 (R8).
- **Gateway limits stay coarse.** Phones on one carrier may share an IP (ASSUMED), so thresholds stay high, and NestJS carries the real limits.
- **No volumetric protection is decided** (R7). Microsoft says WAF doesn't stop these attacks.
- **Cost isn't estimated** (R19).
- **Control plane outage.** Dashboards fail within 5 minutes. Participants are unaffected.
- **Live AI calls share the API deployment.** A slow AI service holds connections there, which T3 measures, and the split above is the fix.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| One shared stack with a region column | Residency would rest on every query being correct, and a failure stops all clients. |
| WebSocket through Front Door | Supported and faster to connect, but decrypts participant data at the edge, adds server state, and the WAF checks only the connection setup. |
| SSE through Front Door | Microsoft states it isn't supported. |
| Front Door in front of Application Gateway | Brings back edge decryption, and SSE still wouldn't work. |
| API Management as the gateway | Not evaluated in depth, and adds a larger product than we need. |
| Kubernetes, or Application Gateway for Containers | Application Gateway for Containers targets Kubernetes. A small team doesn't need to run a cluster. |
| A dedicated AI deployment now | Adds a deployment before the integration or any measurement justifies it (Q41). |
| Separate accounts per region | Clients with collections in two regions would need two logins. |
| Calling the control plane on every request | Adds latency and an outage dependency. Used only for row-level and transcript exports. |
| Per-instance rate limit counters as the main control | Limits drift as instances scale (Q27). |
| Azure Cache for Redis | Retiring. Azure Managed Redis is the replacement. |
| Failing over to another provider or geography when the AI is down | Would keep interviews running, but breaks Q2. |
| Canned questions when the AI is down | Would mix scripted and AI answers without researchers choosing it. |
| Blocking the whole survey during an outage | Wastes the participant's time, when deterministic questions still work. |
