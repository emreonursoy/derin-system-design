# ADR 0003: Participant app as a static offline-first shell, with replies that don't depend on streaming

**Status:** Accepted, frozen for submission. Nothing is built or benchmarked. **Class:** Reliability on bad networks, languages, right-to-left and AI questions are Required. The outbox, saved replies and the AI interface are Necessary. Device binding, cross-device recovery and budgets are Assumed. **Inputs:** Q5, Q6, Q8, Q13, Q14, Q17, Q20, Q22, Q25, Q26, Q27, Q32, Q39, Q41, Q50. **Risks:** docs/risks.md.

## Context
- Participants use phones on slow networks, in many languages, including right-to-left (brief).
- The app must handle deterministic and conversational AI questions (brief).
- Target (ASSUMED): first load under 150 KB of JS gzipped, on low-end Android on slow 3G.
- Deterministic questions work offline after first load. AI questions need a connection (Q13).

## The comparison
| Criterion | Static shell, client rendered | Server-rendered per question | Plain server pages, no offline |
| --- | --- | --- | --- |
| Next question on a bad network | Instant, no round trip | Needs a round trip | Needs a round trip |
| Survives a dropped connection | Yes, with an outbox | No | No |
| First load | Heavier | Lighter | Lightest |
| CDN caching | All shell files, same for everyone | Personal pages can't be cached | Personal pages can't be cached |
| Health data on the phone | Unsynced answers, in IndexedDB | None | None |

## Decision
Build the participant app as a **Next.js static export**: a client-rendered shell with a service worker, served through Front Door. No server rendering, and no participant data in any cached file.

### Flow
1. The invite link opens the shell, with the signed token in the URL **fragment**. Fragments aren't sent to servers, so the token stays out of logs. The shell clears it before any SDK loads.
2. The shell posts the token to the regional stack, with its device credential if it has one, and receives a participant token lasting 15 minutes (ASSUMED). The server checks revocation.
3. The shell fetches the survey definition, in the chosen language. The service worker stores it on the phone.
4. Answers wait in an IndexedDB outbox. Final answers sync in small batches, with random backoff.
5. The server validates each answer against the definition, because skip logic runs on the phone and can be tampered with.
6. The server acknowledges by answer. Each client answer ID is unique, so retries are safe (ADR 0001). The phone deletes each answer's value once acknowledged.

### One invitation, one response
**Each invitation maps to one logical response (Q50).** There's no endpoint that returns saved answers, and an unbound device gets nothing from the response. A bound device can fetch saved AI replies, which can echo its own text.

| Situation | What happens |
| --- | --- |
| First open | Unless the invitation is withdrawn or revoked, the server creates the response if none exists, and issues an opaque device credential, stored in IndexedDB. |
| First answer or AI turn accepted | Binds that credential to the response, by compare-and-set. The participant token carries the credential, and the guard compares it on every save and turn, so a second device can't add to a bound response. Binding at the first answer, not at open, means opening a link in a messaging app and then the main browser doesn't lock anyone out (ASSUMED, untested). |
| Same device | The credential resumes the response. The phone keeps its position and coded branch flags for skip logic until the response finishes. Outbox answers sync against the same response, idempotently. |
| New device, or cleared storage | No valid credential. "This survey was started on another device." It sees no earlier answers, wording, branch results or replies. A device that loses the binding race clears its outbox. |
| Completion | Submitted idempotently. Later saves to that response are refused. |
| Revoked link or withdrawn response | New sessions are refused. A withdrawal rejects saves at once. After a revocation, a token already issued can save until it expires, up to 15 minutes (ASSUMED). The app says so when refused, and clears its outbox. Saved answers stay unless withdrawn. A withdrawn invitation stays closed, so reopening the link creates nothing. |

**Cross-device recovery is a product assumption, not implemented here.** If Surgo wants it, a client-requested reset could rebind the same response, returning no answers. It would need a decision on what the next question may reveal.

**What this costs:**
- **Usability:** a replaced phone or cleared storage stops the participant from continuing. Answers already synced still count as partial.
- **Forwarded links:** the first person to answer binds the response, so a forwarded link can lock out the intended participant. They can't read earlier answers, and a client can revoke the link.
- **What it doesn't establish:** counting is per invitation, not per real person (ADR 0001).

### Identity and consent
| Point | Decision |
| --- | --- |
| Who holds contacts | The client, by default. We hold no names or phone numbers. |
| Invite ID | Random, in the signed token with the collection and expiry. Never derived from the response ID. |
| Mapping | Invite ID to response ID sits in a separate schema with its own database role. It's used for revocation and withdrawal requests, and has no grant to the query, export or AI paths. T4 checks this. |
| Link expiry | 30 days, or the end of fieldwork (ASSUMED). |
| Consent | A first screen, before any answer: what is collected, that **an AI asks some questions**, that answers aren't medical advice, and how to withdraw. It also covers any review of text by the AI provider, once the contract says what applies. |
| Withdrawal | A code, shown with its survey's withdrawal link once the device is bound and synced, withdraws the response from any device. The link carries the regional host and collection. Before that, nothing is on the server, and "clear this device" removes local data. A client admin can also submit a request by invite ID (ADR 0004). Withdrawn answers aren't restored (ADR 0001). |

### Getting the participant their reply
**The reply never depends on the connection that requested it.** Streaming is an improvement on top. Delivery comes from a saved record in Postgres.

**What must always hold:**

| Guarantee | Rule |
| --- | --- |
| Input first | The participant's text is saved before any AI call. |
| At most one saved reply | The client creates the turn ID, scoped to the response from the token. A repeated request returns the existing turn, and a fencing number stops an outdated holder saving. |
| Reply saved first | The finished reply is saved to Postgres before the "done" event. **The client treats streamed text as provisional, and accepts the participant's next answer only after "done".** If the API dies first, a re-run's reply replaces the streamed text. |
| Survives a dropped connection | Generation continues if the phone disappears. |
| Survives a crash | An abandoned turn is picked up and re-run. A re-run may repeat an AI call, but saves at most one reply. |
| Ends while the worker runs | A turn finishes as done, refused or unanswered. Refusals aren't retried. The participant's saved answer is kept unless withdrawn. If the worker is down, the client stops waiting after 5 minutes (ASSUMED), and an alert fires. |
| Withdrawal wins | A reply for a withdrawn response is discarded (ADR 0001). |

**Who holds a turn.** The API claims the turn: it adds 1 to the fencing number, and sets a lease of 75s (ASSUMED). While the call runs, it renews the lease every 25s (ASSUMED). A renewal or save succeeds only if the fencing number is still its own, so an instance that paused and lost its lease discards its work. Retries renew the same claim, up to the 5-minute limit.

**Recovery.** A job in the background worker finds expired leases, claims the turn the same way, and re-runs it. This needs no Inngest. A re-run has no live stream, so the client gets the saved reply. A paused instance can still duplicate an AI call, which costs money and doesn't change the result.

| Delivery path | How it works | When used |
| --- | --- | --- |
| 1. SSE | `POST /v1/turns` responds with the stream itself, so the instance doing the work holds the connection | Default, if the first event arrives |
| 2. Long-poll | `GET /v1/turns/{turn}`, on any instance, held for up to 25s (ASSUMED), returning the saved reply or "still working" | No first event in 5s (ASSUMED), or after a drop |
| 3. Return later | The reply waits in Postgres. Reopening on the same browser shows it | Phone offline or closed |

| Failure | What happens |
| --- | --- |
| Repeated POST, including a gateway retry | Same turn ID, so no second call. It returns the saved reply, or "still working". |
| Stream blocked or buffered on the network | The client long-polls, for the turn, then the session. The participant sees "typing", then the whole reply. |
| Stream drops mid-reply | The client fetches the finished reply, which replaces the partial text. No token-by-token resume. |
| Token expires during a stream | The server closes the stream. The client renews, and fetches the saved reply. |
| Deploy or gateway update cuts a stream | Same as a drop. |
| SSE fails everywhere | A setting in the survey definition turns it off by region, with no app release. Everyone long-polls. |
| AI call fails or times out | Retried a limited number of times (ASSUMED). The participant sees "Your answer is saved, and the reply is coming." |
| AI service slow, down or over its limit | Follows ADR 0002. |
| WAF blocks a turn or answer (HTTP 403) | Retrying won't help. "We couldn't send this. You can reword it, or skip this question." Skipping records the question as blocked, with no text. The text is deleted from the phone either way. We count these, with no content. |

Waiting long-polls are woken through Redis with the turn ID only. The reply itself is read from Postgres (Q27).

**The cost of the fallback:** participants on long-polling lose token-by-token display, and wait for the whole reply. SSE and long-poll behaviour through the gateway is an assumption until T3.

### Refusals and safeguarding
The AI service reports `refused`, with a category if the contract provides one.

| Point | Decision |
| --- | --- |
| Retry | Never. |
| What the participant sees | A fixed message written for the survey, never generated. If a self-harm category is available, it includes support contacts for that survey and language. |
| Data | Their text is kept. The turn is flagged, and the survey moves on. |
| Requirement | A collection can't have AI questions until its owner supplies the fixed message and support contacts. |
| Not promised | This isn't crisis detection. Misses and false positives will happen, and language coverage isn't known. |

The client's own safeguarding protocol is outside these apps (R1).

### AI service interface
Interview turns go through an `AiService` interface (ADR 0005 has the analytics side). Our app doesn't need to know what sits behind it. **Whether Derin exposes anything like this is unconfirmed** (Q39).

| Our app sends | Our app receives |
| --- | --- |
| Turn ID, language, the question, and this response's history unless the service holds it | Streamed tokens, a final reply, an AI-question-finished signal, and a state: slow, unavailable, over limit or refused |

**The integration must confirm, or we can't use AI questions:**

| Requirement | Why |
| --- | --- |
| Processing location and content retention | Participant text is health data |
| Deleting a response's data on withdrawal | If it can't, the consent screen and withdrawal confirmation must state that limit |
| Safe to repeat a turn ID, or at-most-one reply handled by us | Crash recovery |
| Streaming, or a whole reply | Decides whether SSE adds anything for interviews |
| Context limited to one response, and no tools acting on other data | Injection can't reach another participant's data |
| Plain text output, and no clinical advice as a rule | Participant safety |

**Who orchestrates the interview depends on the contract (Q39),** including who holds conversation history, and who decides an AI question is finished. If our app must decide, a turn limit per question (ASSUMED) is the simplest rule. A copy of each turn stays in our Postgres in any case, because transcripts need it.

While offline, an AI question shows "waiting for connection". It never shows canned questions. Turns carry a content type, so voice can be added later (Q8).

### API endpoints
Versioned REST with OpenAPI, like the dashboard (ADR 0004). All except withdrawal go through the participant guard: token, device credential, response scope, rate limit.

| Endpoint | Purpose |
| --- | --- |
| `POST /v1/sessions` | Exchange the invite token and any device credential |
| `GET /v1/survey` | Survey definition, in a language |
| `POST /v1/answers` | Sync a batch of answers and any completion, acknowledged per item |
| `POST /v1/turns`, `GET /v1/turns/{turn}` | Start a turn, with the response as SSE. Fetch the saved reply by long-poll |
| `POST /v1/withdrawals` | Withdraw with the code. Needs no token, so it works from any device. Rate-limited, and audited |

### Stream rules
| Point | Rule |
| --- | --- |
| Reader | `fetch` with a stream reader, so the token goes in a header, never the URL |
| Expiry | The server closes the stream when the token expires, for participants and dashboard users |
| First event | Sent at once, so the client can detect a blocked stream |
| Heartbeat | Every 15s (ASSUMED) |
| Dashboard chat | Same pattern: SSE, saved reply, long-poll fallback |
| Dashboard live updates | Poll. No stream |

### Languages and right-to-left
| Point | Decision |
| --- | --- |
| Direction | `dir` set per language, with CSS logical properties only |
| Components | Only the shadcn/ui components needed, tested in Arabic (Q14) |
| Text | Survey translations arrive with the definition. Interface strings load per language. |
| Fonts | System fonts first. Subsets load per language only if needed. |

### Budget
150 KB gzipped (ASSUMED) must hold React, Next.js, the app and interface strings. Not measured. **No PostHog in the participant app**, because even page views reveal that a person is in a health survey. Sentry loads only after an error, on an allowlist, with no replay.

## Privacy and residency
| Component | Impact |
| --- | --- |
| Front Door | Serves only generic files. Sees no answers or tokens. |
| Application Gateway | Decrypts and inspects answers and AI text inside the region. |
| Invite link | A bearer credential until the first answer binds it. Mitigated by revocation, and because no endpoint returns saved answers. |
| Fragment token | Can stay in browser history on a shared phone. The shell removes it from the address bar after exchange. The shell keeps it in local storage to exchange again, which extends its time on the phone. |
| IndexedDB outbox | **Unsynced health answers sit in clear on the phone, outside any geography.** Each is deleted on acknowledgement. After link expiry, remaining data is cleaned up **the next time the app runs**. If it never runs again, the data stays until the browser clears it or the participant does. We don't promise a deletion time. The app has a "clear this device" action. |
| Position and branch flags | Remain on the phone until the response finishes. Coded, but a branch flag can imply an answer. |
| Service worker cache | Survey definition only. Could reveal the survey's topic on a shared phone. |
| Free text | Participants may type names or places. Redacted in-region before any dashboard display (ADR 0004). |
| AI service | Receives this response's text. Location and retention depend on the contract (R6). |
| Redis | Turn IDs only, never reply text. |
| Sentry | No request bodies, URLs or breadcrumbs. SDK behaviour isn't verified. |

## Consequences and risks
- **First load is the biggest bet.** On slow 3G, a failed first load means no offline ability at all.
- **Reconnecting costs more** without the edge (ADR 0002).
- **Open streams hold API connections.** A slow phone holds one longer. T3 measures this, and it's the trigger for a dedicated AI deployment (Q41).
- **Abandoned phones.** Answers never synced are lost if storage is cleared. The UI shows sync status.
- **Background sync support is not verified.** The outbox always retries when the app is open.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Server-rendered per question | Every question needs a round trip, on the networks where that fails most. |
| Plain server pages with no offline | Lightest first load, but loses answers on a dropped connection. |
| Native app | Install friction on phones with limited storage, and outside the stack. |
| Resume code | Removed. Costs completions, and the brief doesn't require cross-device resume. |
| Several ranked attempts per invitation | Tried, then dropped (Q50). Disproportionate for this case study. |
| Resuming on a new device with no check | The next question, and any AI reply, can reveal earlier answers. |
| Encrypting the outbox on the phone | The key would live on the same phone. |
| WebSocket through Front Door | Rejected by Q32. See ADR 0002. |
| Long-polling only | Simplest, but the brief asks chat to stream. It is the fallback. |
| Generating in the worker and relaying tokens to the API | Needs a token relay, and Redis must hold no reply text. |
| A durable queue for every turn | Adds latency to an interactive path, and a component or vendor exposure. A saved turn row is enough. |
| Storing partial replies for resume | Redis must hold no content, and Postgres would take many extra updates. |
| Streaming through Inngest | Built for background work, and would put participant content in Inngest (Q17). |
| Token in the URL path or query | Would reach Front Door and gateway logs. |
