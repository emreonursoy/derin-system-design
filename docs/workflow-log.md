# Workflow log (feeds submission/ai-workflow-note.md)

Add one entry per phase.

## Phase N: <name>
- Prompt (paste or summarize):
- What the AI produced:
- What was wrong or shallow:
- What I changed or decided:
- Subagent used and what it found:

## Phase 1: Outline, risks and questions
- Prompt (paste or summarize): "Read CLAUDE.md and the docs. Don't write the design yet. Propose the outline, the top risks for each brief item, and the questions you need me to answer before deciding. Save the questions to docs/open-questions.md."
- What the AI produced: outline and risks in chat; 19 questions (8 blocking) with proposed defaults; Azure source list in assumptions.md.
- What was wrong or shallow: Foundry model availability for South Africa North is unresolved because the search results disagreed. Inngest, PostHog and Sentry hosting regions are unverified. Proposed defaults and their figures are unvalidated assumptions.
- What I changed or decided: rejected accepting the defaults as a batch and answered the 8 blocking questions myself. See Phase 2.
- Subagent used and what it found: none; no design draft exists yet to review.

## Phase 2: Blocking answers and impact check
- Prompt (paste or summarize): my own answers to Q1-Q8. Record them with me as decider, mark what cannot be verified as ASSUMED, give the Microsoft pages for the Front Door and region claims, and report what my answers change and where the AI disagrees before any ADR. Then stop.
- What the AI produced: Q1-Q8 recorded; 7 new questions (Q20-Q26); source table showing full page or excerpt.
- What was wrong or shallow: Phase 1 cited pages from search excerpts only. Its Q2 default, "pick models by what that region offers", missed that South Africa North has no chat model on Standard/Regional, only on Regional Provisioned. It also said "regional" where Microsoft commits to the geography. Phase 1 offered an "accept all" option on questions that needed individual judgment.
- What I changed or decided: Q1-Q8 answers. Q9-Q19 stay open because my exceptions list was left as a placeholder. Q20-Q26 pending.
- Subagent used and what it found: none; no design draft exists yet to review.

## Phase 3: Remaining decisions and ADRs 0001-0003
- Prompt (paste or summarize): my decisions on Q9-Q26, individually. Asked for a check of local token verification with the Better Auth MCP, and a comparison in ADR 0001 of one row per response against a narrow write table feeding a read table. Then write the ADRs and stop for my review.
- What the AI produced: all decisions recorded; capacity derivations; three ADRs with status Proposed; five new questions (Q27-Q31).
- What was wrong or shallow: its Q9 and Q10 defaults (regional gateway, API traffic off the CDN) were not what I chose. While writing ADR 0001, the arithmetic showed the narrow table does not reduce write volume and batching merges almost nothing, so the case rests on isolation and rebuilds, not efficiency. Its 7s freshness budget had ignored the projector step; it is 5s. The docs do not show an org role claim in the JWT, so that part of local verification is unproven. LLM quota is unchecked against the derived token rates.
- What I changed or decided: Q9-Q26 as recorded. ADR review pending; Q27-Q31 open.
- Subagent used and what it found: none yet. The ADRs have not had an independent review.

## Phase 4: ADR challenges and first independent review
- Prompt (paste or summarize): my decisions on Q27-Q31, rejecting per-instance counters and adding Redis. Three challenges: is 44 vs 40 GB just final storage, and does the recommendation hold once autosave rewrites count; how is 38.7M tokens split, what brings it down most, and what degrades first; what happens if SSE doesn't work through Front Door. Then update the ADRs, run design-reviewer, and rank without fixing.
- What the AI produced: total-written table in WAL, token split with levers and a degrade order, a reply design independent of streaming, four new questions (Q32-Q35).
- What was wrong or shallow: my SSE question exposed a real miss. The AI had called SSE "unproven" and said it found no page, but Microsoft's Front Door WebSocket page states SSE is not supported. ADR 0003 had also rejected WebSockets on that wrong basis. The 40 vs 44 GB comparison was not final storage, and left out WAL and rewrites. It had applied the tap-question pace to typed AI turns, which inflated 38.7M. It had said A needed a separate dedupe record, when a version inside the row would do.
- What I changed or decided: Q27-Q31. Redis added to the stack. Q32-Q35 open.
- Subagent used and what it found: design-reviewer. Top defects in the ADRs: the projector has no safe way to find new rows; the query cache can store stale results under a new version; no owner for in-flight AI turns, and no path for content filter refusals; 300 AI turns/s contradicts a 10s stream (the maximum is 200/s, or 25.8M tokens per minute, not 38.7M); the pending reply exposes data through a forwarded link. Also noted that the dashboard design and diagram are unwritten. Its doubts about gpt-4.1-mini availability and quota rows were checked against pages already read: both are listed for Standard, but assumptions.md did not record the availability. Nothing fixed yet, pending my review.

## Phase 5: Refocusing ADR 0002 on AI failure behaviour
- Prompt (paste or summarize): "Change of plan on 0002: don't go deep on token numbers, since the brief doesn't ask for them. Keep one rough estimate marked as assumed, and focus on what happens when the AI is slow, down or over quota."
- What the AI produced: one rough estimate; a table of slow, down and over-quota behaviour; degradation order and recovery; simplified AI section in assumptions.md.
- What was wrong or shallow: the AI had spent effort on token levers and quota tiers the brief doesn't ask for, and I had followed that with my own question. Its failure behaviour was only a quota ordering, with no signals or user-visible states for slow or down.
- What I changed or decided: scope of ADR 0002's AI section. Q33 and Q35 reworded, still open.
- Subagent used and what it found: none. This change has not been independently reviewed.

## Phase 6: SSE through Application Gateway
- Prompt (paste or summarize): I researched this myself and changed the transport to SSE for AI turns and chat, with Front Door for static files only and all API traffic through a regional Application Gateway with WAF. Keep the saved reply and long-polling, close streams at token expiry, dashboard live updates poll. Check Microsoft's docs on whether turning off response buffering limits the WAF, and what gateway timeouts mean for streams. Show slower connection setup as a cost, and record WebSocket through Front Door as rejected.
- What the AI produced: updated Q9, Q10, Q32; gateway settings, timeout and cost tables in ADR 0002; stream rules in ADR 0003; sources in assumptions.md; new Q36.
- What was wrong or shallow: the AI had proposed WebSocket through Front Door and would have kept edge decryption resting on an ASSUMED reading. I chose differently. It found that the default 20s request timeout would also break its own 25s long-poll, and that Microsoft's two pages describe the timeout differently. Connection cost figures rest on assumed round trip times, not measurements.
- What I changed or decided: Q9, Q10, Q32. Q36 open.
- Subagent used and what it found: none. This change has not been independently reviewed.

## Phase 7: Review fixes and dashboard design
- Prompt (paste or summarize): align ADR 0001, CLAUDE.md and brief.md; 5s dashboard poll; put Q36 on the risks. Don't reopen infrastructure, and mark what can't be verified as ASSUMED or a risk, without asking. Finish what's left from the review, then design the dashboard: rendering, API style, charts, filters, cross-tabs, exports and chat. Ask only if it changes the architecture, three questions at most.
- What the AI produced: review fixes in ADRs 0001-0003; ADR 0004 (rendering, API, query layer, charts, filters, cross-tabs, transcripts, exports); ADR 0005 (chat, summary); docs/risks.md; one question (Q37).
- What was wrong or shallow: the earlier projector, cache and turn designs had real gaps found by the reviewer. Polling the 10s target with a 10s poll didn't add up. Serving live rollups from a replica was unnecessary: writing them to Redis after each batch removes replica lag from the live path. Filtered views can't meet 10s at this size without about 120 cores, which the AI had not stated. Sizing relied on a 70% cache hit it couldn't support; it now sizes cache-off. Reviewer's doubt about gpt-4.1-mini was only partly right.
- What I changed or decided: scope and poll interval. Q33, Q34, Q35 recorded as AI defaults under my instruction. Q37 open.
- Subagent used and what it found: privacy-security-reviewer found that exact counts on large segments let anyone recover a hidden group by subtracting two filters, that row-level export sat outside the minimum of 10 with a false "can't be joined" claim, that analysts could rebuild records through transcript references, and that identity separation, consent and interview injection had no design. capacity-checker found that open filtered views can show data about 67s old rather than 30s, that replica sizing left out chat, and that the gateway count used 2,300 connections when the design holds about 10,300. Fixes applied, and not re-reviewed. The AI declined to act on the reviewer's point that Q21 alone couldn't deliver "reduce" by changing the decision itself; it tightened controls within it. Tightened beyond my answers, for my review: Q17 (no PostHog in the participant app) and Q25 (resume code).