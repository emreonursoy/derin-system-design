# Derin live results dashboard: system design (case study)

## Mission
Produce a system design, not code. Deliverables are tracked in docs/requirements.md. Time budget: a few focused hours.

## Read first, every session
1. docs/brief.md (the case; never contradict it)
2. docs/requirements.md (checklist; update status as you go)
3. docs/assumptions.md and docs/adr/ (decisions already made)
4. docs/workflow-log.md (append an entry after each phase)

## Non-negotiable rules
- Cover what the brief asks: rendering strategy, API layer, data access and auth for both apps; dashboard fast and fresh at 500k x 80 and 300 users; participant app reliable on bad networks; the 2-3 biggest trade-offs and what was rejected.
- Every number is DERIVED (arithmetic shown in docs/assumptions.md) or marked ASSUMED. No unlabeled figures.
- Participant data is health data. Every component choice states its privacy and residency impact.
- Disagree with me when warranted: give the reason and an alternative, then wait for my decision.
- Decisions go in docs/adr/NNNN-title.md before they appear in design.md or the diagram.
- Ask instead of guessing when an answer changes the architecture; record it in docs/open-questions.md.
- The stack is fixed unless I approve a change: TypeScript, Postgres, NestJS, React, Next.js, shadcn/ui, Inngest, PostHog, Sentry, Better Auth, Azure, CDN, gateway with auth + rate limiting + routing.
- For anything about Better Auth capabilities, use the Better Auth MCP. Do not invent plugin or API names.
- Check Azure limits and regional availability with the Microsoft Learn MCP, and cite the page in docs/assumptions.md.
- The .excalidraw file is the diagram's source of truth. After every change, tell me to open it in Excalidraw and export diagram/architecture.png. Then read that PNG and list overlaps and unclear parts before saying it is done.
- Use the subagents in .claude/agents/ for review. Never review your own draft in the same context.

## Style
Short sentences, tables for comparisons, plain words. design.md is at most 5 pages.

## Logging
After each phase append to docs/workflow-log.md: the prompt used, what the AI got wrong or shallow, what I changed.