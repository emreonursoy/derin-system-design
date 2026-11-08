# ADR 0005: Chat through fixed tools, and a summary from aggregates only

**Status:** Proposed, for my review. **Inputs:** Q2, Q7, Q15, Q16, Q17, Q18, Q20, Q21, Q23, Q32. **Risks:** docs/risks.md.

## Context
- Chat must stream its answers, and read a health survey's results while it is in the field.
- Participants wrote the transcripts, so their text is **untrusted input**. It may contain instructions aimed at the model.
- The model must never be able to see what the user can't, or get around suppression.
- No pgvector (Q15).

## Chat: how the model reaches data

| Criterion | Fixed tools (chosen) | Text-to-SQL, read-only | Send the data to the model |
| --- | --- | --- | --- |
| Suppression | Same query layer as the dashboard | Bypassed unless SQL is rewritten | Raw data reaches the model |
| Injection impact | Limited to allowed tools and the user's rights | Could read any table the role can | Everything already exposed |
| Question range | Limited to what tools offer | Very wide | Wide, but limited by size |
| Effort | Tools to build | Validator to build, hard to get right | Small |

**Decision:** the model calls a small set of tools. The server runs them through the query layer (ADR 0004), using **the user's identity from their token**. The model can't supply a user, role or collection.

| Tool | Returns | Role |
| --- | --- | --- |
| `list_questions` | Question wording and types | Viewer |
| `get_results(question, filter)` | Suppressed counts for one question | Viewer |
| `get_crosstab(row, column, filter)` | Suppressed table | Viewer |
| `count_responses(filter)` | Suppressed count | Viewer |
| `search_transcripts(terms, filter)` | Short redacted quotes with one-time references, only if the filter **plus search terms** match 10 or more | Analyst with transcript grant |

Limits: 5 tool calls per question, and 75s per turn (ASSUMED). All tool calls are audited, and count towards the same comparison detection and daily transcript limit as manual use.

### Flow
1. The browser posts a question with a client turn ID, and gets the turn.
2. A NestJS instance takes the turn's lease (ADR 0003), and renews it while working. It then loops between the model and tools.
3. Tokens stream over SSE, through Application Gateway.
4. The finished answer is saved to Postgres. If the stream drops, or the token expires, the browser renews and fetches the saved answer by long-poll.

### Identity on a detached turn
Generation outlives the connection, and a re-run may land on another instance. The turn row stores user, org, role and claim expiry, never the token itself.

- A tool call after the claim expiry is refused, and the turn waits for the user's next renewed request.
- Every read of a turn or history entry checks the owner and the current role.
- Quotes are hidden from a user who no longer holds the transcript grant.

## Defending against prompt injection
| Risk | Control |
| --- | --- |
| A transcript tells the model to reveal other data | Tools run with the user's rights and suppression, so an injected instruction can't widen access. |
| Text is mistaken for instructions | Transcript quotes, including model-written turns, are wrapped and labelled as data, and the system prompt says never to act on them. Helps, but isn't a guarantee, so the controls above don't rely on it. Injected text could still mislead an answer's conclusions, which is why sources are always shown. |
| An answer leaks data through a link or image | Answers render as plain text and tables. No model-supplied links or images load. |
| The model tries to act on something | No tool writes or sends anything. |
| A user tries to work out hidden values through chat | Same suppression, rate limits and audit as the dashboard. |
| Injection reaches a viewer | Viewers have no transcript tool, so transcripts never enter their conversation. |

Azure's prompt attack detection can be turned on as well (ASSUMED suitable, and its data path isn't checked). It is an extra layer, and off until that's confirmed.

## Keeping answers honest
| Point | Decision |
| --- | --- |
| Numbers | Each number must come directly from a tool result. The server compares numbers in the answer with that turn's results. A number it can't match, including one the model worked out by subtracting, is removed and replaced with "not available". |
| Sources | Each answer lists its queries, with an "as of" time. One click opens the same chart or table. |
| Quotes | Shown with a reference to the redacted transcript, for roles allowed to read it. |
| Causes | The system prompt says results show what was reported, not why. The UI labels answers as AI-generated. |
| No data found | The model says so, and doesn't estimate. |

## History
| Point | Decision |
| --- | --- |
| Storage | Regional Postgres, per user and collection. Not shared within the org. |
| Retention | 30 days (ASSUMED). |
| Contents | Questions, answers and tool results, all already suppressed, and redacted quotes. |
| Old context | Earlier tool results are dropped from context, and answers are summarized, so a role change or withdrawal doesn't linger in the prompt. |
| Stateful model features | Not used. Conversation state stays in our Postgres (ADR 0002). |

## Executive summary

| Point | Decision |
| --- | --- |
| Input | **Aggregates only**, through the query layer, with no transcripts. All roles read the summary, including viewers. |
| Trigger | When responses grow by 10%, at most once an hour, or when an admin asks (ASSUMED). |
| Pipeline | An Inngest job carrying IDs only. A step loads results inside the region, calls the model, writes to Postgres, and returns only the summary ID. |
| Check | Numbers are compared with the input, as for chat. A failed check keeps the previous summary, marked stale. |
| Display | Labelled AI-generated, with an "as of" time and response count. |
| Under quota | First thing to pause (ADR 0002). The last summary remains. |
| Withdrawal | Triggers regeneration, ahead of the hourly limit (Q18). |
| Deployment | Standard or Provisioned regional. Batch isn't allowed, because Microsoft documents it as a Global type (Q23). |

A summary of a small survey could be specific enough to identify someone, so it isn't generated until the collection has 100 responses (ASSUMED).

## Privacy and residency
| Component | Impact |
| --- | --- |
| Model | Receives the question, suppressed results, and redacted quotes for allowed roles. Processing stays in the geography. Abuse monitoring may store flagged prompts there unless modified monitoring is approved (ASSUMED). |
| Postgres | History and summaries, in the region. |
| Redis | Budgets and turn IDs only. No questions or answers. |
| Inngest | IDs only, in events and step results. |
| PostHog | Records that chat was used, never its text. |
| Sentry | No prompts or replies in error reports. |

## Consequences and risks
- Chat can only answer what its tools allow. Questions outside them get a clear "can't answer that".
- Keyword search will miss meaning and phrasing, and is weakest in languages without full-text support. Transcript answers will be less complete than with embeddings.
- The number check catches mismatches, not bad reasoning. A correct number can still support a wrong claim.
- Chat is an LLM cost per user, and pauses before interviews do.

## Rejected
| Alternative | Why rejected |
| --- | --- |
| Text-to-SQL | Wide reach, but suppression and injection control would rest on an SQL validator. |
| Sending result tables or transcripts in the prompt | No user-specific control over what reaches the model, and size limits at this scale. |
| pgvector | Declined (Q15). Search stays on Postgres full-text. |
| Transcripts in the summary | The summary is shared with viewers, who may not read transcripts. |
| Inngest for chat generation | Built for background work, and would risk content in Inngest. |
| Azure Assistants or Responses API threads | Store conversation state in the service. We keep it in our own Postgres. |
| Shared chat history | People could see each other's questions, which might reveal who was looked at. |
