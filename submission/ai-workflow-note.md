# AI workflow note

**a. Tools and context**

I used Claude Code. A `CLAUDE.md` rules file set the mission and rules: label every figure derived or assumed, write decisions in ADRs first, ask before guessing, and never review a draft in the same context. Context came from repo files: the brief, a requirements checklist, assumptions, ADRs, open questions and this workflow log. MCP servers gave access to Better Auth docs and Microsoft Learn. Three subagents did the reviews: `design-reviewer`, `privacy-security-reviewer` and `capacity-checker`.

**b. Prompts that mattered**

1. "Read CLAUDE.md and the docs. Don't write the design yet. Propose the outline, the top risks for each brief item, and the questions you need me to answer before deciding. Save the questions to docs/open-questions.md."
2. Three challenges to the ADRs (summarized): is 44 vs 40 GB just final storage, and does the recommendation hold once autosave rewrites count; how is 38.7M tokens split and what degrades first; what happens if SSE doesn't work through Front Door?
3. "Change of plan on 0002: don't go deep on token numbers, since the brief doesn't ask for them. Keep one rough estimate marked as assumed, and focus on what happens when the AI is slow, down or over quota."

**c. Where the AI was wrong or shallow, and what I changed**

- **SSE:** it called SSE through Front Door "unproven", but Microsoft's page says it isn't supported. I moved all API traffic to a regional Application Gateway with WAF, with Front Door for static files only.
- **Numbers:** the 40 vs 44 GB comparison left out WAL and rewrites. The 38.7M tokens applied a tap-question pace to typed AI turns.
- **Privacy:** a reviewer found that subtracting two filters could reveal a small hidden group, so the controls were tightened.
- **Regions:** its first model default missed that South Africa North has no Standard/Regional chat model.
- **Pinned filter sets:** added without defining membership, backfill or withdrawal. I dropped them.
- **Diagram:** a script check passed, but the first render showed overlaps. I found these by exporting and checking the PNG.

**d. Transcript:** submission/transcript/<export>.zip
