# AI workflow note

**a. Tools and context.** I used Claude Code, with a `CLAUDE.md` rules file: label figures derived or assumed, record decisions in ADRs, and ask before guessing. Context came from repo files, including the brief, assumptions, ADRs, open questions and a workflow log. MCP servers gave Better Auth and Microsoft Learn docs. Subagents (`design-reviewer`, `privacy-security-reviewer`, `capacity-checker`) did most reviews. They run inside Claude Code, so they aren't external reviews. I developed later architecture feedback with ChatGPT, and pasted it in.

**b. Prompts that mattered** (the last two summarized)

1. "Don't write the design yet. Propose the outline, the top risks for each brief item, and the questions you need me to answer before deciding."
2. What happens if SSE doesn't work through Front Door?
3. Treat the brief as primary, classify each decision, define the Derin boundary, cut speculation, and plan before changing anything.

**c. Where the AI was wrong or shallow**

- **SSE:** it called this "unproven", but Microsoft says it's unsupported. I moved API traffic to a regional gateway, kept as an assumption to validate.
- **Scope:** it went deep on model quota and replica sizing. I cut these.
- **Correctness:** reviews found that a projector could restore a withdrawn response. The redesign is untested.
- **Over-design:** it recommended ranked attempts per invitation. I chose one invitation, one response.
- **Diagram:** a script check passed, but the render showed overlaps.

Nothing was built or benchmarked.

**d. Transcript:** `submission/transcript/claude-code-session.md`
