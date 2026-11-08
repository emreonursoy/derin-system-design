# Derin: live results dashboard and participant app, system design

Notes to support `diagram/architecture.png`. Decisions and rejected options are in `docs/adr/`. **Nothing is built or benchmarked.** Performance figures are targets, and capacity figures are estimates. They are tagged **(A)** assumed or **(D)** derived, with arithmetic and sources in `docs/assumptions.md`. Open items are in `docs/risks.md`.

## 1. Scope and approach
The aim is the simplest credible design a small team can own: one TypeScript monorepo and managed Azure services, with no Kubernetes or microservices.

| Class | Decisions |
| --- | --- |
| **Required** (brief) | Both apps, live charts, filters, saved sets, cross-tabs, summary, streaming chat, transcripts, exports, org roles, per-collection features, languages and right-to-left, 500k x 80, 300 users |
| **Necessary** | Outbox and idempotent sync, saved replies, write/read separation, one query layer, audit, fixed chat tools, withdrawal handling |
| **Assumed** | 10,000 participants (my target), per-region stacks, specific regions, the WAF, Redis uses, disclosure values, freshness targets, retention, who owns what in AI |
| **Out of scope** | Model deployment and quota, safeguarding protocol, legal basis, cross-org access, survey authoring |

**Load estimates** (D, from one answer per 20s, 20% in AI questions and a 3x burst, all A): 1,500 answers/s at burst, up to 2,000 open AI streams, 60 polls/s, about 17.5 filtered or chat queries/s.

## 2. Rendering strategy
Both apps are Next.js static exports on Front Door, which handles **static files only**. No Next.js server runs.

| App | Why static | What it costs | Privacy and residency |
| --- | --- | --- | --- |
| Participant, with a service worker and IndexedDB outbox | The next question needs no round trip, and answers survive a dropped connection | Heavier first load, and a failed first load means no offline. Skip logic runs on the phone, so the server revalidates every answer | Files hold no data. Unsynced answers sit on the phone until acknowledged |
| Dashboard, with shadcn/ui, Recharts and client-side data | A login-only tool gains little from server rendering, and each server is one more place data is decrypted | Slower first paint. Code in the browser holds the token, so it needs a strict content security policy | No health data in HTML. The JWT stays in memory |

## 3. API layer and deployment
| Part | Decision | Privacy and residency |
| --- | --- | --- |
| API deployment | NestJS on Azure Container Apps: REST, OpenAPI, generated client, SSE, and live AI calls | Filters go in POST bodies, so they stay out of URLs and logs |
| Worker deployment | Same codebase: projector, redaction, exports, turn recovery, background jobs | Inside the region. Inngest gets IDs only |
| Participant API | Sessions, survey, answer batches, turns, withdrawals (ADR 0003) | Scoped to one response by token and device credential |
| Guard | Token, org against collection, role, feature, rate limit, on every request | One place to enforce and test |
| Streaming | SSE for AI turns and chat, with a saved reply and long-poll fallback | Tokens in headers, never URLs |

A dedicated AI deployment is added only if the confirmed integration or measured streaming load justifies it.

**Ingress (assumed).** A regional Application Gateway with WAF takes all API traffic for now, protecting unauthenticated public endpoints and decrypting data in the region. Microsoft documents SSE through it, and states Front Door doesn't support SSE. **Its necessity and SSE behaviour are untested assumptions (T3), and it isn't the only option.** The alternative is Container Apps ingress alone: simpler, with no WAF, and SSE also untested. New connections cost about 300 to 1,000 ms more from Nairobi (D, from assumed round trips).

## 4. Data access
| Part | Decision | Privacy and residency |
| --- | --- | --- |
| Write table | Append-only, one row per answer, source of truth, idempotent | Pseudonymous. Regional only |
| Read table | One compact row per response, rebuildable. Only screeners are indexed | Same. Row-level security on collection |
| Projector | Worker. Batches about every 2s (A) | Inside the region |
| Rollups | Counters for all and completed | Suppressed before reaching Redis |
| Read replica | **Start with one.** Filtered scans and exports. Scale on benchmark results | Same region |
| Redis | Rollups, query cache, rate limits | Suppressed aggregates, counters and opaque IDs. Never answers or text |
| Blob Storage | Export files, 24 hours (A), and the withdrawal log | Regional. A leaked 15-minute download link (A) works until it expires |

**Rules.** Saves and batch replays are idempotent. A crash replays a whole batch or none of it. A late commit isn't skipped. A changed answer replaces the old one in rollups. Saves continue if the projector stops.

**One invitation, one response.** One invitation contributes at most one participant, enforced by a unique mapping, so all counts, suppression, chat and summaries count invitations. This stops duplicates within an invitation, and **doesn't establish unique real people**.

**Withdrawal.** A log outside the database, then a tombstone, rejects new saves. The projector subtracts and deletes in one transaction. Other workers recheck under a lock. Exports exclude tombstones and recheck the primary. A restore replays the log. Filtered aggregates can lag until the replica catches up, shown by "as of". Downloaded exports can't be recalled.

**Main tables** are in the diagram's panel. Invitations map to responses only in restricted `identity.invite_map`.

**Regional boundary.** Each collection pins to one region's stack. Deploy one first, with region as a template variable. Regions are assumed examples, and we promise neither region nor geography until each service is checked.

## 5. Auth
**Dashboard.** Better Auth runs in a small control plane in one home region, holding users, orgs, roles and the collection-to-region map. One org owns each collection. It holds no participant data, but **dashboard users' details may leave their region**. Regional APIs verify a 5-minute JWT locally through JWKS, which Better Auth documents. The org role in the token is **assumed**, with a regional copy as the fail-closed fallback (T2).

| Action | Viewer | Analyst | Admin |
| --- | --- | --- | --- |
| Charts, cross-tabs, saved filters, summary, chat over aggregates | Yes | Yes | Yes |
| Edit saved filters, aggregate export | No | Yes | Yes |
| Read redacted transcripts | No | With a per-collection grant | With a per-collection grant |
| Row-level or transcript export | No | No | Yes, see section 6 |
| Manage members, withdraw or revoke invitations | No | No | Yes |

- **Features:** per-collection switches live in a regional table, and the server enforces them. **Who activates early access is a product assumption**, not decided.
- **Stale access:** a removed user keeps access for up to 5 minutes, and we say so. Row-level and transcript exports also check the control plane live, and are blocked if it's down.
- **Account security:** MFA for admins and analysts, with per-role enforcement assumed.

**Participants.** A signed invite link in the URL fragment is exchanged for a 15-minute token (A). We hold no names by default. **The first answer binds the response to that device, checked on every save, and no endpoint returns saved answers.** A new device is refused. Cross-device recovery is a product assumption. Cost: a replaced phone stops the participant, and a forwarded link can bind first.

## 6. Dashboard: fast and fresh at 500k x 80 and 300 users
All figures are targets (A, or D from A), and none is measured.

| Class | How | Target |
| --- | --- | --- |
| Unfiltered live views | Projector writes suppressed counters to Redis, and dashboards poll every 5s | Under 1s p95, about 8s old |
| Filtered queries, saved or ad-hoc | One scan on the replica, with a short cache | Under 3s p95 on a cache miss. An open view can be about 67s old, with "as of" and a refresh button |
| Cross-tabs | Same path, run on demand. 3,160 question pairs per collection (D) rule out pre-computing | Same as filtered |
| Slow or background | Exports, summary, redaction | No interactive target. They show progress |

**Scaling.** Start with one replica. On a measured miss: tune, add replicas, move the read table to its own server, then add a columnar store, if p95 stays at or above 3s or clients need fresher filtered views. A rough estimate of 70 cores with no cache (D) isn't a deployment. Losing the replica or Redis means "busy" on filtered views, while saves continue. Per-org query limits are set from T1 (R21e).

**The query layer** is the key decision. All aggregates, including AI tools, use one module: validated filter JSON, parameterized SQL, one disclosure policy, cache, audit.

**Disclosure.** Enforcement in one place is the requirement. **The values are proposed, for review, because Surgo's policy isn't known.** They include a minimum group of 10, complement and secondary suppression. They also include per-value minimums, rounding to 5, holding small live segments and alerts on near-identical filters. These reduce the risk of working out hidden values. **They don't fully stop differencing attacks.**

**Transcripts.** Redacted in-region, with a per-language test before browsing. **Redaction reduces identification risk, and doesn't guarantee anonymity,** so transcripts need a separate grant, and reads are audited.

**Exports.** **An admin can start row-level or transcript exports** (transcripts need the grant), with banded screeners, MFA, a recent session, a live control plane check, audit and notice to admins. Second-admin approval is an **optional org policy**, and if on, a one-admin org waits for a second admin. Further requirements are pending Surgo's policy. Row-level export sits outside the minimum group, which we state openly.

## 7. Participant app: reliable on bad networks
| Situation | What happens |
| --- | --- |
| No connection, deterministic question | Answer goes to the outbox, and syncs later with a unique ID |
| AI question, no connection | "Waiting for connection". No canned questions |
| Stream fails or is blocked | The POST is the stream. The reply is saved first, so the client long-polls for it on any instance |
| Token expires mid-stream | The stream closes, the client renews and fetches the saved reply |
| Process dies | A lease expires, and the worker re-runs the turn. Fencing allows one saved reply. Streamed text is provisional until "done" |
| Postgres primary down | Answers wait in the outbox. AI turns wait. Audited dashboard actions return "busy" |
| AI slow, down or over limit | Answer is saved. Summary pauses first, then chat, then new interviews. Affected turns are marked in the data |
| AI refuses | Never retried. A fixed survey message, with support contacts |

- **Local data:** deleted on acknowledgement. After link expiry it's cleaned up **the next time the app runs**, so we promise no deletion time. Position and coded branch flags stay until the response finishes.
- **Languages:** `dir` per language, CSS logical properties, tested in Arabic.
- **Budget:** under 150 KB of JS (A) on low-end Android on slow 3G, with no PostHog.
- **Consent:** a first screen says an AI asks some questions. A withdrawal code follows, and works from any device.

## 8. AI integration boundary
Derin already runs AI-led interviews, but **its contract is unknown, and we claim nothing about it.** Our apps call an `AiService` interface.

| Our application owns | The AI service may own (assumed) |
| --- | --- |
| Authorization, analytics tools, query layer, audit | Model selection and deployment |
| Saving turns and replies, streaming to browsers | Prompt orchestration, inference |
| Reconnection and failure handling | Token optimization, quota, capacity |

**Orchestration is configurable, depending on the contract.** If the service orchestrates chat, it calls our tool endpoint with a credential scoped to one turn. Otherwise a small bounded loop in our API calls the same executor. Rights, suppression and audit are ours either way.

**Chat** uses fixed tools with the user's rights, and no free-form SQL. Transcripts are untrusted input. Our server renders figures from structured tool results, with sources. All prose is labelled AI-written, because free text can't be reliably validated. **The summary** uses aggregates only, and if generation fails the last one stays, marked stale.

**The contract must confirm:** location, retention, deletion on withdrawal, safe repeats, streaming, one-response context and refusals. Without a geography commitment, a collection has no AI questions.

## 9. Biggest trade-offs and what I rejected
| Trade-off | Chosen | Rejected | Cost accepted |
| --- | --- | --- | --- |
| **Analytics store** | Postgres only, with a write table feeding a read table, and one replica to start | One row per response (close on data volume, but mixes saves with vacuum, and can't rebuild). Columnar store now (a second system before numbers justify it) | A projector to keep correct, filtered views about 67s old, and capacity not yet measured |
| **Entry and streaming** | Assumed Application Gateway, SSE, regional stacks | WebSocket through Front Door (decrypts at the edge). Container Apps ingress alone (no WAF, still the fallback). One shared stack (residency would rest on every query) | Slower connection setup, more to run, cost not estimated |
| **Participant rendering** | Static offline-first shell, one response per invitation | Server-rendered per question (a round trip each). Several ranked attempts (too complex for this case) | Heavier first load, health answers on the phone until synced, no cross-device recovery |

## 10. Security, privacy, residency
| Boundary | Position |
| --- | --- |
| Residency | Assumed requirement, with no in-country promise, and no claim of any specific law. **Phones and downloaded exports sit outside any geography** |
| AI service | Location and retention unconfirmed. Refusals aren't crisis detection, so clients need a safeguarding protocol |
| Others | Private endpoints, separate database roles (A). Inngest gets IDs only, PostHog is dashboard only, and Sentry is allowlisted. Hosting regions are unverified |

## 11. Validate before production
| Priority | Test | Can change |
| --- | --- | --- |
| 1 | Postgres queries at 500k x 80, cache off, plus projector and replica lag (T1, T8) | Replicas, then the columnar trigger |
| 2 | SSE through the chosen ingress on a low-end phone (T3) | Ingress, long-poll default, a dedicated AI deployment |
| 3 | Role and region claims in the JWT (T2) | Regional role copy |
| 4 | Offline sync, crashes, withdrawal races (T5, T9) | Shell, projector mechanism |
| 5 | Residency across services, and the AI contract (R6, R10, R11, T7) | Vendors, orchestration mode, regions offered |
| 6 | Disclosure and export policies, attacks (T4, R23) | Policy values |

