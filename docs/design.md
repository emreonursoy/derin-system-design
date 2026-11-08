# Derin: live results dashboard and participant app, system design

Notes to support `diagram/architecture.png`. Decisions and rejected options are in `docs/adr/`. Nothing is built or benchmarked.

Sizing figures are tagged **(A)** assumed or **(D)** derived, with arithmetic and sources in `docs/assumptions.md`. Policy values set by decision, such as the minimum of 10, the 5s poll and the 5-minute token, are recorded in `docs/open-questions.md` and also unproven. Open items are in `docs/risks.md`.

## 1. Context and capacity
Sizes are per regional stack.

| Input | Value |
| --- | --- |
| Largest collection | 500k responses x 80 questions (brief) |
| Dashboard users | 300 (brief) |
| Concurrent participants | 10,000, my target |
| Large collections at once | 3 (A) |
| Answers | 500/s average, 1,500/s burst (D), at one per 20s and 3x burst (A) |
| AI | 2,000 participants in AI questions (D); 100 turns/s average, 200/s maximum (D) |
| Model load | About 12M tokens per minute, as an order of magnitude (A) |
| Dashboard reads | 60 polls/s, 10 ad-hoc queries/s, 7.5 chat queries/s (D) |
| Query capacity | 70 cores with no cache (D), from a 1s scan on 4 workers (A), so 3 replicas of 32 vCores (D) |
| Storage | About 19 GB per large collection, as a floor (D) |

## 2. Rendering strategy
Both apps are Next.js static exports, served from Front Door, which handles **static files only**. No Next.js server runs anywhere.

| App | Why static | What it costs | Privacy and residency |
| --- | --- | --- | --- |
| Participant, with a service worker and IndexedDB outbox | The next question needs no round trip, and answers survive a dropped connection | Heavier first load, and a failed first load means no offline. Skip logic runs on the phone, so the server revalidates every answer | Files hold no data. Unsynced answers sit on the phone until acknowledged, or until the link expires |
| Dashboard, with shadcn/ui, Recharts and client-side data | A login-only tool gains little from server rendering, and each server is one more place data is decrypted | Slower first paint, as data loads after the app. Code in the browser holds the token, so it needs a strict content security policy | No health data in HTML. The JWT stays in memory |

**Both apps gain** one set of files for every region, and one less service per region. Charts load near the viewport, and every chart has a table view.

## 3. API layer and gateway
| Part | Decision | Privacy and residency |
| --- | --- | --- |
| Entry | Regional Application Gateway with WAF, for all API traffic | Data is decrypted and inspected inside the region. WAF logs are scrubbed |
| API | NestJS and its workers on Azure Container Apps, versioned REST, OpenAPI, generated TypeScript client | Filters go in POST bodies, so they stay out of URLs and logs |
| Guard | Token, org against collection, role, feature, rate limit, on every request | One place to enforce and test |
| Limits | Counters in regional Redis, so they stay accurate as instances scale | Opaque IDs and counts only |
| Streaming | SSE for AI turns and chat, with a long-poll fallback | Tokens in headers, never URLs |
| Live updates | Dashboards poll every 5s | Polls read only Redis |

Microsoft states Front Door doesn't support SSE, and documents it for Application Gateway with response buffering off. **That doesn't limit the WAF**, but it applies to the whole gateway, so slow phones can hold connections longer. Stream and long-poll paths use a 90s backend timeout (A), because the 20s default would kill a 25s long-poll (A).

**The cost:** without the edge, each new API connection takes about 300 to 1,000 ms longer from Nairobi (D, from assumed round trips).

## 4. Data access
| Part | Decision | Privacy and residency |
| --- | --- | --- |
| Write table | Append-only, one row per answer, source of truth, safe to retry | Pseudonymous. Regional only |
| Read table | One compact row per response. Only screeners are indexed | Same. Row-level security on collection |
| Projector | NestJS worker. Batches every 2s (A), using a transaction-safe watermark, so late commits aren't skipped | Inside the region |
| Rollups | Counters by question, option or bin, and state, for all and completed | Suppressed before reaching Redis |
| Replicas | 3, for filtered scans and exports | Same region |
| Redis | Azure Managed Redis: rollups, cache, limits | Suppressed aggregates, counters and opaque IDs. Never answers or text. No geo-replication |
| Blob Storage | Export files, deleted after 24 hours (A) | Regional, zone-redundant. Public endpoint, because browsers download by a 15-minute link (A), so a leaked link works until it expires |

Each collection pins to one region, in one stack per region. Stacks share no participant data.

**Main tables** (panel beside the diagram). Control plane: `user`, `organization`, `member`, `collection_region`. Regional: `collection`, `survey_version`, `respondent`, `answer_write`, `response_read`, `conversation_turn`, `rollup`, `saved_filter`, `collection_feature`, `audit_log`. A random invite ID maps to a response ID only in restricted `identity.invite_map`, and `respondent` holds no names. Losing Redis loses no participant data.

## 5. Auth
**Dashboard.** Better Auth runs in a small control plane in Sweden Central, holding users, orgs, roles and the collection-to-region map. It holds no participant data, but **dashboard users' details may leave their region**. Regional APIs verify a 5-minute JWT locally through JWKS, which Better Auth documents. The org role in the token is **assumed**, with a regional copy as the fallback, and is first on the test list.

| Action | Viewer | Analyst | Admin |
| --- | --- | --- | --- |
| Charts, cross-tabs, saved filters, summary, chat over aggregates | Yes | Yes | Yes |
| Edit saved filters, aggregate export | No | Yes | Yes |
| Read redacted transcripts | No | With a per-collection grant | With a per-collection grant |
| Row-level or transcript export | No | No | Yes, with a second admin's approval |
| Manage members and features | No | No | Yes |

- **Features:** per-collection switches, such as early access, live in a regional table. Admins turn them on, and the server enforces them.
- **Org:** one org owns each collection. Cross-org access is out of scope.
- **Stale access:** a removed user keeps access for up to 5 minutes, and we say so. Row-level export also checks the control plane live, and is blocked if it's down.
- **Account security:** MFA for admins and analysts. Per-role enforcement is assumed.

**Participants.** A signed invite link in the URL fragment, with no account, is exchanged for a 15-minute token (A). We hold no names by default. The invite-to-response mapping sits in a restricted schema. Another device needs a resume code.

## 6. Dashboard: fast and fresh at 500k x 80 and 300 users
| View | How | Age at worst (D) |
| --- | --- | --- |
| Unfiltered charts | Projector writes suppressed counters to Redis after each batch, and dashboards poll | About 8s |
| New filter or cross-tab, on a cache miss | One scan on a replica, under 3s (A) | About 10s, if replica lag is under 4s |
| Open filtered view, saved or ad-hoc | 30s refresh and cache bucket (A), with "as of" time and refresh button | About 67s |
| Small segments | Update only after 10 new responses | Exempt |

- **Charts:** counts per option for choice and scale questions. Up to 20 bins (A) for numeric, with mean and estimated median.
- **Filters:** validated JSON compiled to parameterized SQL. No free-text filters. Saved filter sets store only the definition, and run on the same path.
- **Cross-tabs:** run on demand, because 3,160 question pairs per collection (D) rule out pre-computing.
- **Overload:** a 10s statement timeout and per-user limits (A), then a "busy" response.

**Disclosure.** All aggregates pass through one suppression function:

| Control | Rule |
| --- | --- |
| Minimum | Fewer than 10 responses in a group or cell is hidden, for every role |
| Filter values | Each ORed value must reach 10 alone, which stops subtracting two filters |
| Rounding | All filtered counts round to the nearest 5 (A) |
| Others | Secondary suppression, complement checks, no minimum or maximum, alerts on near-identical filters |

This **reduces** the risk of recovering hidden values, and doesn't prevent it. Filtered numbers are slightly inexact.

**Transcripts and exports.** Text is redacted in-region with Azure Language Text PII, which lists Arabic and Swahili. A language must pass a test first. Row-level export uses banded screeners, and **sits outside the minimum of 10**, which we state openly. Downloaded files can't be recalled.

**Chat and summary.** Chat calls fixed tools through the same query layer, using the user's rights, with no free-form SQL. Transcripts are treated as untrusted, and numbers not found in tool results are removed. The summary uses aggregates only, because viewers read it too. Search uses Postgres full-text, with no pgvector.

## 7. Participant app: reliable on bad networks
| Situation | What happens |
| --- | --- |
| No connection, deterministic question | Answer goes to the outbox, and syncs later with a unique ID |
| AI question, no connection | "Waiting for connection". No canned questions |
| Stream fails or is blocked | The reply is saved in Postgres first, so the client long-polls for it, or sees it on return |
| WAF blocks the text | Not retried. The participant can reword or skip, and the text is deleted from the phone |
| Token expires mid-stream | The stream closes, the client renews and fetches the saved reply |
| Process dies | A lease expires and a sweeper re-runs the turn. One reply per turn |
| AI slow, down or over quota | Answer is saved. Summary pauses first, then chat, then new interviews. Affected turns are marked in the data. Data never leaves the geography |
| Content filter refuses | Never retried. A fixed message written for the survey, with support contacts |

- **Languages:** `dir` per language, CSS logical properties, tested in Arabic. Interface strings load per language.
- **Budget:** under 150 KB of JS (A) on low-end Android on slow 3G. No PostHog in this app.
- **Consent:** a first screen says an AI asks some questions, and gives a withdrawal code.
- **Interview safety:** context holds one response only, with no tools, and replies are plain text. A Derin engine, if reused behind the interface, must meet the same rules.

## 8. Biggest trade-offs and what I rejected
| Trade-off | Chosen | Rejected | Cost accepted |
| --- | --- | --- | --- |
| **Analytics store** | Postgres only, with a write table feeding a read table | One row per response (close on data volume, but mixes saves with vacuum, and can't rebuild). Columnar store now (a second system before numbers justify it) | Filtered views about 67s old, and capacity that rests on a scan not yet measured |
| **Entry and streaming** | Application Gateway for APIs, SSE, one stack per region | WebSocket through Front Door (faster setup, but decrypts at the edge). One shared stack (residency would rest on every query) | Slower connection setup, no edge protection for APIs, more to run per region |
| **Participant rendering** | Static offline-first shell | Server-rendered per question (a round trip per question). Plain pages (lightest, but loses answers) | Heavier first load, and answers on a shared phone until synced |

**Columnar trigger:** adding replicas first, then moving the read table, and only then a columnar store. It triggers if mean query cost passes 5 core-seconds (D), p95 reaches 3s, or clients need filtered views fresher than this.

## 9. Security, privacy, residency
| Boundary | Position |
| --- | --- |
| Residency | Pinned to a region, with no in-country promise. Azure lists no Kenya region, so a Kenyan collection leaves the country. **Phones and downloaded exports sit outside the geography:** unsynced answers stay on the phone, and a downloaded file goes wherever its user is |
| Models | Azure-hosted, regional or provisioned only. Microsoft commits to the geography, not the single region. No suitable model means no AI questions there. South Africa North has chat on Provisioned only |
| Abuse monitoring | Flagged prompts may be stored in the geography for human review. Modified monitoring is assumed, and the notice covers it otherwise |
| Third parties | Inngest gets IDs only. PostHog is dashboard only. Sentry strips URLs, breadcrumbs and IP addresses. Hosting regions are unverified |
| Internal access | Private endpoints and separate database roles (A). One logged break-glass role. Logs kept 30 days (A) |
| Withdrawal | A tombstone blocks re-creation. Deletes cover answers, transcripts and quotes. Rollups and summary update |
| Law | We claim no specific law |

## 10. Risks and what I would test first
| Test first | Why | Can change |
| --- | --- | --- |
| Query benchmark at 500k x 80, cache off, 8 and 80 charts | All sizing rests on the 1s scan | Replicas, then the columnar trigger |
| Role and region claims in the JWT | Undocumented, and access control relies on them | Regional role copy |
| SSE and long-poll through the gateway on a low-end phone | Microsoft's two timeout descriptions differ | Timeouts, or long-poll by default |
| Suppression attacks through dashboard and chat | One gap exposes health data | Controls, then noise |
| First load on slow 3G | A failed first load means no offline | Shell design |

**Top risks:**
- **Safeguarding:** the content filter isn't crisis detection, and isn't trained on many target languages. A client protocol is needed.
- **Re-identification:** controls reduce it. Admins and stories in transcripts remain exposures.
- **AI availability:** our quota tier and South Africa North pricing are unknown.
- **Not decided or estimated:** cost isn't estimated anywhere, and DDoS Protection on the gateways is undecided.
