# ADR 0005: Chat through fixed tools, and a summary from aggregates only

**Status:** Accepted, frozen for submission. Nothing is built or benchmarked. **Class:** Streaming chat and the summary are Required. Fixed tools, query layer reuse, authorization and audit are Necessary. Orchestration ownership is Assumed until the contract is confirmed. Model deployment and quota are Out of scope. **Inputs:** Q7, Q15, Q16, Q17, Q18, Q20, Q21, Q32, Q39, Q44, Q50. **Risks:** docs/risks.md.

## Context
- Chat must stream its answers, and read a health survey's results while it's in the field (brief).
- Participants wrote the transcripts, so their text is **untrusted input**.
- The model must never see what the user can't, or get around suppression.
- The brief confirms that Derin runs AI-led interviews. It doesn't say whether Derin provides analytics chat, orchestration or only inference (Q39).

## Who owns what
| Our application always owns | The AI service may own (ASSUMED until confirmed) |
| --- | --- |
| Authorization, using the user's identity | Model selection and deployment |
| The analytics tools and running them | Prompt orchestration |
| The query layer, disclosure policy and audit | Inference and token optimization |
| Saving turns, answers and summaries | Quota and capacity planning |
| Streaming to the browser, reconnection and failure handling | |

**We make no claim about what Derin's platform provides.**

### Orchestration is configurable
The `AiService` interface (ADR 0003) has an analytics side. Which mode applies depends on the confirmed contract, and there's no default.

| | Service orchestrates | Our backend orchestrates |
| --- | --- | --- |
| Who runs the tool loop | The AI service | A small bounded loop in our API |
| How tools are called | The service calls our tool endpoint, with a credential scoped to one turn | Our loop calls the same executor in-process |
| Who enforces rights | **We do, in both modes** | **We do, in both modes** |
| When it applies | The service offers this, and can call our region under the contract | The service offers only model access |

The tool executor is the same module in both modes, so nothing about rights, suppression or audit changes. The contract decides the mode.

## How the model reaches data
| Criterion | Fixed tools (chosen) | Text-to-SQL, read-only | Send the data to the model |
| --- | --- | --- | --- |
| Suppression | Same query layer as the dashboard | Bypassed unless SQL is rewritten | Raw data reaches the model |
| Injection impact | Limited to allowed tools and the user's rights | Could read any table the role can | Everything already exposed |
| Question range | Limited to what tools offer | Very wide | Wide, but limited by size |

**Decision:** a small set of tools, run through the query layer (ADR 0004), using **the user's identity from their token**. The model can't supply a user, role or collection.

| Tool | Returns | Role |
| --- | --- | --- |
| `list_questions` | Question wording and types | Viewer |
| `get_results(question, filter)` | Suppressed counts | Viewer |
| `get_crosstab(row, column, filter)` | Suppressed table | Viewer |
| `count_responses(filter)` | Suppressed count | Viewer |
| `search_transcripts(terms, filter)` | Short redacted quotes with one-time references, only if the filter **plus search terms** meet the minimum | Analyst with grant |

All counts are per invitation (ADR 0001). The tools aren't designed to return invite or response IDs to the model, and T4 checks this. Limits: 5 tool calls per question, and 75s per turn (ASSUMED). Tool calls are audited, and count towards the same limits as manual use.

### Flow
1. The browser posts a question with a client turn ID, and gets the turn.
2. The API claims the turn, with a lease and fencing number (ADR 0003), and tools run as above.
3. The POST response is the SSE stream, so the working instance holds the connection.
4. The finished answer is saved to Postgres, and then "done" is sent. If the stream drops or the token expires, the browser renews and fetches the saved answer by long-poll.

Chat turns follow the same rules as interview turns (ADR 0003): at most one saved answer, saved before "done", and recovered by the worker after a crash. A run stops at 75s (ASSUMED). With retries and recovery, a turn with no answer after 5 minutes (ASSUMED) ends as failed, and the user can ask again.

### Identity on a detached turn
The turn row stores user, org, role and claim expiry, never the token.

- A tool call after the claim expiry is refused, and the turn ends as expired. The user can ask again.
- Every read of a turn checks the owner and the role in the current token.
- Quotes are hidden from a user who no longer holds the transcript grant.

## Defending against prompt injection
| Risk | Control |
| --- | --- |
| A transcript tells the model to reveal other data | Tools run with the user's rights and suppression, so an instruction can't widen access. |
| Text is mistaken for instructions | Quotes are labelled as data. This helps, but isn't a guarantee, so the control above doesn't rely on it. Injected text could still mislead a conclusion, which is why sources are always shown. |
| An answer leaks data through a link or image | Answers render as plain text and tables. No model-supplied links or images load. |
| The model tries to act on something | No tool writes or sends anything. |
| A user works out hidden values through chat | Same disclosure policy, rate limits and audit as the dashboard. |
| Injection reaches a viewer | Viewers have no transcript tool. |

## Keeping answers honest
**We don't claim that a natural-language answer can be validated.** Matching numbers against results is brittle: a correct number can be derived, a matched number can mislead, and tokens have already streamed by the time a check could run.

| Point | Decision |
| --- | --- |
| Figures | Tool results come back as structured data with IDs. The answer shows figures as result blocks, which **our server renders from that data**, not from model text. |
| Sources | Each answer lists its queries with an "as of" time. One click opens the same chart or table. |
| Prose | Constrained by the prompt to refer to result blocks. **All prose is labelled AI-written and unverified,** including any number in it. We run no number detection. |
| Derived figures | Percentages come with the results. A comparison the model works out is its own claim, and the source blocks sit next to it. |
| Quotes | Shown with a reference, for roles allowed to read them. |
| Causes | The prompt says results show what was reported, not why. The UI labels answers as AI-generated. |
| No data found | The model is prompted to say so, and not to estimate. |

If the service orchestrates, emitting result references is a contract item. Without it, answers still list sources, and the whole answer is labelled AI-written.

## History
| Point | Decision |
| --- | --- |
| Storage | Regional Postgres, per user and collection. Not shared within the org. |
| Retention | 30 days (ASSUMED). |
| Contents | Questions, answers and tool results, already suppressed, and redacted quotes. |
| Old context | Earlier tool results are dropped from context. Earlier answers can still repeat their content, which the cleanup rule below handles for withdrawals. |
| Withdrawal | Each turn records the response IDs behind its quotes, plus those of earlier turns in its context, so follow-ups that repeat a quote are covered (internal). A saver rechecks them under the lock in ADR 0001. Cleanup replaces a turn's quotes, prose and tool results with "Removed: a source was withdrawn". Aggregate-only turns stay, and age out at 30 days. |

## Executive summary
| Point | Decision |
| --- | --- |
| Input | **Aggregates only**, through the query layer, so counts are per invitation. No IDs. No transcripts, because viewers read it and may not have transcript access. |
| Trigger | When responses grow by 10%, at most once an hour, or when an admin asks (ASSUMED). |
| Pipeline | An Inngest job carrying IDs only. The worker loads results inside the region, calls `AiService`, writes to Postgres, and returns only the summary ID. |
| Figures | The same result blocks as chat, rendered by our server, with sources. |
| Failure | **The previous summary stays, marked stale, with its "as of" time.** Same when the service is unavailable or over its limit, when the summary is the first thing to pause (ADR 0002). |
| Display | Labelled AI-generated, with an "as of" time and response count. |
| Withdrawal | Triggers regeneration, ahead of the hourly limit (Q18). |
| Small surveys | Not generated until the collection has 100 responses (ASSUMED), since it could be too specific. |

## Privacy and residency
| Component | Impact |
| --- | --- |
| AI service | Receives the question, suppressed results, and redacted quotes for allowed roles. **Location and retention depend on the contract** (R6). |
| Tool credential | If the service orchestrates: scoped to one turn and short-lived. It carries no access beyond that user's rights, and the service gets results only, never database access. |
| Postgres | History and summaries, in the region. |
| Redis | Rate limits and turn IDs only. |
| Inngest | IDs only, in events and step results. |
| PostHog | Records that chat was used, never its text. |
| Sentry | No prompts or replies. |

## Consequences and risks
- Chat can only answer what its tools allow. Other questions get a clear "can't answer that".
- Keyword search will miss meaning and phrasing, and is weakest in languages without full-text support.
- Prose can still draw a wrong conclusion from correct figures, or attach a correct block to the wrong claim. Sources and labels help, and don't prevent it.
- Chat uses AI capacity, and pauses before interviews do.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Text-to-SQL | Suppression and injection control would rest on an SQL validator. |
| Matching every number in the answer against tool results | Brittle, and can't work after streaming. |
| Assuming Derin is inference only, or assuming it provides chat | The brief confirms neither (Q39). |
| Giving the AI service database access | Would bypass the query layer. |
| Sending result tables or transcripts in the prompt | No user-specific control, and size limits at this scale. |
| pgvector | Declined (Q15). |
| Transcripts in the summary | Viewers read it, and may not have transcript access. |
| Inngest for chat generation | Built for background work, and would risk content in Inngest. |
| Shared chat history | People could see each other's questions. |
