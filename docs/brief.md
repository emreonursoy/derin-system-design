# Surgo Health: Full-Stack Engineer (TypeScript) case study, "Live results dashboard"

Surgo's Derin platform runs AI-led interviews with thousands of participants. You are designing two web applications.

**1. Client results dashboard.** Used by researchers and client organizations while a survey is in the field:
- live response counts and charts per question (single and multiple choice, scales, numeric)
- filters and segments (e.g. "women, 30-50, Nairobi"), combinable and saved as reusable filter sets
- cross-tabulation tables
- an AI-generated executive summary
- a "chat with your data" assistant that streams its answers
- transcript browsing and exports
- roles: admin, analyst and viewer per organization; features can be enabled per collection, e.g. early access

**2. Participant survey web app.** Participants answer on their phones, often on slow mobile networks. It supports multiple languages, including right-to-left, and must work for both deterministic questions and conversational AI questions.

**Scale and constraints:**
- A collection can have up to 500k responses x 80 questions.
- 300 concurrent dashboard users at peak.
- Participant data is sensitive (health-related).
- Small team: you own frontend and backend for these apps.

**Time:** you have 2 days from receiving this brief. With good AI tools we expect a few focused hours of work, not two full days.

**What to send us before the deadline:**
1. An architecture diagram (required). Use any tool you like (Excalidraw, draw.io, Mermaid, Figma, Lucid...), with or without AI help. It must show the components, the data and request flows, and your technology choices. Send the editable source as well as an image: in the interview you will change it live.
2. Short notes or slides (optional), only to support the diagram. 5 pages or slides at most.
3. An AI workflow note (required, about half a page):
   a. which tools and harness you used and how you gave the AI context (e.g. Claude Code, Cursor, ChatGPT, MCP servers, rules files, subagents)
   b. 2-3 prompts that mattered
   c. where the AI was wrong or shallow, and what you changed
   d. Attach the session export or transcript. We may ask you to walk us through it in the interview.

**AI policy:** Surgo is an AI-driven company, and we expect you to use AI tools for this exercise just as you would at work. In the interview we will discuss your design in depth, including parts the diagram doesn't show. We will also ask you to adapt it live with your own tools while sharing your screen.

**Your design should cover:**
- architecture for both apps: rendering strategy, API layer, data access, auth
- how the dashboard stays fast and fresh at that scale
- how the participant app stays reliable on bad networks
- the 2-3 biggest trade-offs and what you rejected

## My own stack and choices (not from the brief)
TypeScript, Postgres, NestJS, React, Next.js, shadcn/ui, Inngest (background jobs, AI workflows), PostHog (analytics), Sentry (errors), Azure, a CDN, an API gateway with authentication (Better Auth), rate limiting and routing. Data residency by region. Participant concurrency target: at least 10k at once. Design only, nothing is built.