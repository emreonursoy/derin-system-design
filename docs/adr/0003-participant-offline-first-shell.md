# ADR 0003: Participant app as a static offline-first shell, with replies that don't depend on streaming

**Status:** Proposed, for my review. **Inputs:** Q5, Q6, Q8, Q9, Q10, Q13, Q14, Q17, Q20, Q22, Q25, Q26, Q27, Q32, Q34. **Risks:** docs/risks.md.

## Context
- Participants use phones on slow networks, in many languages, including right-to-left.
- The app must handle deterministic and conversational AI questions.
- Target (ASSUMED): first load under 150 KB of JS gzipped, on low-end Android on slow 3G.
- Deterministic questions work offline after first load. AI questions need a connection (Q13).
- Load (DERIVED from ASSUMED): 1,500 answers/s at burst, up to 2,000 open AI streams.
- The shell comes from Front Door. All API traffic goes through a regional Application Gateway (ADR 0002).

## The comparison

| Criterion | Static shell, client rendered | Server-rendered per question | Plain server pages, no offline |
| --- | --- | --- | --- |
| Next question on a bad network | Instant, no round trip | Needs a round trip | Needs a round trip |
| Survives a dropped connection | Yes, with an outbox | No | No |
| First load | Heavier: must download the app and survey | Lighter | Lightest |
| Server load per answer | One small write | Render plus write | Render plus write |
| CDN caching | All shell files, same for everyone | Personal pages can't be cached | Personal pages can't be cached |
| Health data on the phone | Unsynced answers, in IndexedDB | None | None |

## Decision
Build the participant app as a **Next.js static export**: a client-rendered shell with a service worker, served through Front Door. No server rendering, and no participant data in any cached file.

### Flow
1. The invite link opens the shell, with the signed token in the URL **fragment**. Fragments are not sent to servers, so the token stays out of Front Door and gateway logs, and the shell stays cacheable. The shell clears the fragment before any SDK loads, so it can't reach Sentry.
2. The shell posts the token to the regional stack, and receives a participant token lasting 15 minutes (ASSUMED). The server checks revocation.
3. The shell fetches the survey definition, in the chosen language, through the gateway. API traffic isn't cached at the edge, so the service worker stores it on the phone.
4. Deterministic answers, and any autosave, stay in an IndexedDB outbox on the phone. Final answers sync in small batches.
5. The server acknowledges by answer. Each client answer ID is unique, so retries are safe (ADR 0001). The phone deletes answers once acknowledged (Q13).

### Identity and consent
| Point | Decision |
| --- | --- |
| Who holds contacts | The client, by default. We hold no names or phone numbers. If a client needs us to send links, that is a separate store with its own credential. |
| Invite ID | Random, in the signed token with the collection and expiry. Never derived from the response ID. |
| Mapping | Invite ID to response ID sits in a separate schema with its own database role. It's used for revocation, and never appears on the dashboard or in exports. |
| The limit of this | A client can't tie a token to a response through our product. Someone with both the client's list and database access could, which is why the mapping is restricted. |
| Link expiry | 30 days, or the end of fieldwork (ASSUMED). |
| Consent | A first screen, before any answer: what is collected, that **an AI asks some questions**, that answers aren't medical advice, how to withdraw, and any abuse monitoring exposure (ADR 0002). |
| Withdrawal | A withdrawal code shown at the start. A participant with no account and a lost link can still use it. A client can also pass on a request by invite ID, and the server resolves it. |

### Resuming on a new device
A new device must not reveal earlier answers (Q25), and a forwarded link shouldn't let someone else continue a health survey.

| Point | Decision |
| --- | --- |
| First device | The link binds to the first device, using a device key in IndexedDB. |
| Another device | Needs the resume code shown at the start (ASSUMED). This costs some completions on shared or replaced phones. |
| No code | They may start again as a new, separate submission if the survey allows it. Nothing earlier is shown or overwritten. |
| Where to resume | With the code, the server works out the next question from stored answers, and returns its ID. |
| Skip logic | The server sends the true or false result of each branch that depends on earlier answers, never the answers. A branch result, or the next question's wording, can hint at an answer. The resume code limits who sees this. |
| Pending AI reply | Shown only on the device that sent the turn. A reply can repeat what the participant said. A new device sees the question's original wording instead. |
| Changed answers | Each answer gets a new client answer ID, scoped to the response, so a new device can't collide with an earlier one. |

### Getting the participant their reply
**The reply never depends on the connection that requested it.** Streaming is an improvement on top. Delivery comes from a durable record in Postgres.

1. The client sends the turn with an ID it created. The participant's text is saved first, and only then does the model run.
2. Generation runs in a NestJS process, detached from the connection, and continues if the phone disappears.
3. The finished reply is saved to Postgres before it is marked done.
4. The client gets the reply by the best path that works:

| Path | How it works | When used |
| --- | --- | --- |
| 1. SSE | Tokens arrive as they are generated, through Application Gateway. | Default, if the first event arrives |
| 2. Long-poll | A normal request held for up to 25s (ASSUMED), returning the whole reply, or "still working" so the client asks again. | No first event within 5s of the connection opening (ASSUMED), or after a drop. Slow 3G can take many seconds just to connect, so the wait starts after that. |
| 3. Return later | The reply waits in Postgres. Reopening the link on the same device shows it. | Phone offline or closed |

| Failure | What happens |
| --- | --- |
| Stream blocked, or buffered somewhere on the network | The first event doesn't arrive in time, so the client long-polls for that turn, and after two failures for the session. The participant sees "typing", then the whole reply. |
| Stream drops mid-reply | The client keeps the text so far, and fetches the finished reply, which replaces it. No token-by-token resume, because partial replies aren't stored anywhere shared. |
| **Token expires during a stream** | The server closes the stream at expiry. The client renews the token and fetches the saved reply. The client also renews before starting a turn if the token is close to expiry. |
| Gateway configuration update or scale-in cuts streams | Same as a drop. |
| SSE fails everywhere | A setting in the survey definition turns it off by region, with no app release. Everyone long-polls. Load is 100 polls/s on average and at most 200/s (DERIVED), each a lookup by ID. |
| Model call fails with an error or timeout | The turn is released for retry, up to 3 attempts (ASSUMED). The participant sees "Your answer is saved, and the reply is coming." |
| **Content filter refuses the turn** | **Never retried.** See "Refusals and safeguarding" below. |
| **NestJS process dies, or a deploy kills it** | The lease expires, and the sweeper re-runs the turn. |
| AI is slow, down or over quota | Follows ADR 0002, "When the AI is slow, down or over quota". |
| Attempts are exhausted, or 5 minutes pass (ASSUMED) | The turn is marked unanswered and flagged in the data, and the survey moves to the next question. The participant's answer is kept. |
| **WAF blocks a turn or answer (HTTP 403)** | The request never reaches NestJS, so no turn exists and no recovery rule applies. Retrying won't help, so the app doesn't. The participant sees "We couldn't send this. You can reword it, or skip this question." Skipping records the question as blocked, with no text, and the survey moves on. The text is deleted from the phone either way. The app reports a count with no content, so we can tune the rule. |
| Same request sent twice, including a gateway retry | The same turn ID, scoped to the response from the token, returns the existing turn. It never creates a second message, and never returns someone else's turn. |

Waiting long-polls and streams are woken through Redis with the turn ID only. The reply itself is read from Postgres, because no participant content goes in Redis (Q27, Q34). A reconnect that lands on another instance gets no tokens, only the finished reply. That is the normal outcome during a deploy.

### Who owns a turn
A turn is a row in Postgres, with a state and a lease.

| State | Meaning | Next |
| --- | --- | --- |
| `received` | Participant text saved | `generating` |
| `generating` | An instance holds a 75s lease (ASSUMED), above the 60s model stop, and renews it while working. Chat loops also stop at 75s overall (ADR 0005). | `done`, `refused`, or back to `received` on failure |
| `done` | Reply saved | Final |
| `refused` | Content filter blocked it | Final |
| `unanswered` | Attempts exhausted, or 5 minutes passed | Final |

- **One reply per turn:** taking a lease and saving a reply are both compare-and-set on the row, so a slow instance that lost its lease can't overwrite the winner. A re-run is a new model call, but never a second reply.
- **Sweeper:** an Inngest cron job every 30s (ASSUMED), carrying IDs only, returns expired leases to `received`, and marks turns older than 5 minutes `unanswered`.
- **Deploys:** an instance stops taking turns, finishes those it holds, then exits. The gateway's connection draining is set above 90s.

### Refusals and safeguarding
Microsoft documents two cases. A blocked prompt returns HTTP 400 with `content_filter`, before any streamed event. A reply cut off during generation ends with `finish_reason: content_filter`, naming the category, such as `self_harm`.

| Point | Decision |
| --- | --- |
| Retry | Never. The same text gives the same result. |
| Partial reply | Discarded on the phone. |
| What the participant sees | A fixed message written for the survey, not generated. For `self_harm`, it includes support contacts for that survey and language. |
| Data | Their text is kept. The turn is flagged, and the survey moves on. |
| Requirement | A collection can't have AI questions until its owner supplies the fixed message and support contacts. |
| Researchers | They see counts of refused turns by question, subject to the minimum of 10, and can read the turn under the usual transcript permission. |
| Not promised | This isn't crisis detection. Microsoft says its classifiers are trained on 8 languages, and performance varies in others, such as Arabic and Swahili. False positives and misses will happen. Health surveys may trigger false positives often. |

The client's own safeguarding protocol, including who responds and whether a pseudonymous participant can be contacted, is outside these apps. It is on the risk list.

### Interview injection and output rules
A participant can try to manipulate the model. These rules bind our engine and a Derin engine alike.

| Rule | Why |
| --- | --- |
| Context holds one response only | Injection can't reach another participant's data. This is the key limit. |
| No tools | The interview model can't read or change anything. |
| No secrets in the prompt | A leaked research guide is a nuisance, not a breach. |
| No clinical advice | The prompt forbids it, and the consent screen says so. A determined participant may still get something, which stays on the risk list. |
| Plain text only, no links | A reply can't send someone to a phishing page. Transcript pages also render as plain text. |
| Strict content security policy on the shell | It may hold the invite token and clear-text answers. |

Injected text stored in a transcript can later reach analyst chat. ADR 0005 treats it as data.

**The cost of the fallback:** participants on long-polling lose token-by-token display, and wait for the whole reply, typically 10s (ASSUMED) and at most the 60s model stop. On a very slow network this may be better anyway.

### Stream rules
| Point | Rule |
| --- | --- |
| Reader | `fetch` with a stream reader, so the token goes in a header. `EventSource` can't set headers (ASSUMED, general browser knowledge). |
| Auth | Token in the `Authorization` header, never the URL. Checked when the stream opens. |
| Expiry | The server closes the stream when the token expires. This holds for participants and dashboard users. |
| First event | Sent at once, so the client can detect a blocked stream, and the gateway sees a response |
| Heartbeat | Every 15s (ASSUMED), below the gateway's 180s HTTP/2 idle limit |
| Maximum length | Ends by the 60s model stop (ADR 0002), below the 90s gateway timeout |
| Headers | `text/event-stream` and `no-cache`, as Microsoft advises |
| Gateway | Response buffering off, and a separate 90s backend setting on stream and long-poll paths (ADR 0002) |
| Dashboard chat | Uses the same pattern: SSE, saved reply, long-poll fallback |
| Dashboard live updates | Poll. No stream. |

### Engine interface
AI turns run behind a `ConversationEngine` interface, so the Derin engine or our own module can sit behind it (Q5). The contract covers: start a turn with a client ID, emit tokens, write a final reply, and process within the collection's geography. A Derin engine must commit to the geography rule, because these apps cannot enforce it.

While offline, an AI question shows "waiting for connection". It never shows canned questions. Text typed meanwhile waits in the outbox. If the region has no suitable model, the collection has no AI questions at all (Q22). Turns carry a content type, so voice can be added later (Q8).

### Languages and right-to-left
| Point | Decision |
| --- | --- |
| Direction | `dir` set per language, with CSS logical properties only |
| Components | Only the shadcn/ui components needed, tested in Arabic (Q14) |
| Text | Survey translations arrive with the definition. Interface strings load per language. |
| Fonts | System fonts first. Subsets load per language only if needed. |

### Budget
150 KB gzipped (ASSUMED) must hold React, Next.js, the app and interface strings. Bundle sizes are not measured. **No PostHog in the participant app**, which is tighter than Q17, because even page views reveal that a person is in a health survey. Drop-off comes from first-party counts on the server. Sentry loads only after an error, on an allowlist that strips URLs, breadcrumbs and IP addresses, with no replay.

## Test first
1. SSE and long-poll through Application Gateway with WAF, on a real low-end phone with a throttled connection. Include what the request timeout does to a live stream, because Microsoft's two pages describe it differently.
2. Connection setup time from the target networks to the region (ADR 0002 gives the cost).
3. First load on slow 3G against the 150 KB target.

## Privacy and residency
| Component | Impact |
| --- | --- |
| Front Door | Serves only generic files. Sees no answers or tokens. |
| Application Gateway | Decrypts and inspects answers and AI text inside the region. WAF logs need scrubbing (ADR 0002). |
| Invite link | A bearer credential. Forwarding it gives access to the response. Mitigated by revocation, and by not showing earlier answers or pending replies on a new device (Q25). |
| Fragment token | Stays in browser history on a shared phone. The shell removes it from the address bar after exchange. |
| IndexedDB outbox | Unsynced health answers sit in clear on the phone. Encrypting them with a key on the same phone adds little, so I don't claim it. Deleted after acknowledgement, or at link expiry at the latest. The app has a "clear this device" action. WAF-blocked text is deleted once the participant rewords or skips. |
| Service worker cache | Survey definition only. It could reveal the survey's topic to someone using a shared phone. |
| Free text | Participants may type names or places. Pseudonymity doesn't cover this. Text is redacted in-region before any dashboard display (ADR 0004). |
| Model calls | Within the geography (ADR 0002). |
| Redis | Turn IDs only for wake-ups, never reply text. |
| Inngest | IDs only. The job reads text from Postgres inside the region, and its step results return IDs only. |
| Sentry | Could capture answers or the invite token if controls fail, so no request bodies, URLs or breadcrumbs are sent. SDK behaviour isn't verified. |

## Consequences and risks
- **First load is the biggest bet.** On slow 3G, a failed first load means no offline ability at all.
- **Reconnecting costs more.** Each new API connection pays the full path, 300 to 1,000 ms extra (DERIVED from ASSUMED, ADR 0002). Participants on bad networks reconnect most.
- **Open streams hold server connections.** Up to 2,300 (DERIVED), and without response buffering a slow phone holds its NestJS connection longer. Deploys drop them, and the fallback covers this.
- **WAF can block a real answer.** Free text can trigger a false positive. The participant can reword or skip, so some answers are lost to rule tuning (risk R8).
- **Token expiry while offline.** A short-lived token can expire with answers in the outbox. The shell keeps the invite token in memory or local storage to exchange again, which extends its time on the phone. A long gap may need the link reopened.
- **Sync after long gaps.** Answers arrive late, so they don't count against the 10s target (Q26). Many phones returning at once could exceed the 3x burst, so clients use random backoff, and the server accepts small batches.
- **Background sync may be limited.** Browser support is not verified. The outbox always also retries when the app is open, which works everywhere.
- **Abandoned phones.** Answers never synced are lost if the browser clears storage or the participant leaves. The UI shows sync status.
- **Client-side logic can be tampered with.** Skip logic runs on the phone, so the server revalidates every answer against the definition.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Server-rendered per question | Every question needs a round trip, on the networks where that fails most. |
| Plain server pages with no offline | Lightest first load, but loses answers on a dropped connection. Its load speed is the honest counterargument. |
| Native app | Install friction on phones with limited storage, and outside the stack. |
| Encrypting the outbox on the phone | The key would live on the same phone, so it claims more than it gives. |
| **WebSocket through Front Door** | Documented as supported, with faster connection setup. Rejected by my Q32 decision. The client sends one message per turn, so two-way channels add little. It also adds server state, and full details are in ADR 0002. |
| SSE through Front Door | Microsoft states Front Door doesn't support it. |
| Long-polling only | Simplest, and needs no response buffering change. Rejected because the brief asks chat to stream. It is the fallback. |
| A separate worker service for generation | A cleaner split, but more to run. Leases give the same recovery inside NestJS. |
| A durable queue for every turn | Adds latency to an interactive path, and either a new component or content in Inngest. |
| Storing partial replies in Redis or Postgres for resume | Redis must hold no participant content. Postgres would take about 1,000 extra updates/s at 1 per open stream per second (DERIVED from ASSUMED). |
| Streaming through Inngest | Built for background work. It would also put participant content in Inngest, against Q17. |
| Token in the URL path or query | Would reach Front Door and gateway logs. |
