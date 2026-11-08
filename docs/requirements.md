# Requirements checklist (from the brief)

All C and F rows were revised to match ADRs 0001-0005 after the architecture review. Nothing is benchmarked. The architecture is frozen for submission. Manual tasks are in submission/CHECKLIST.md.

| # | Requirement | Where covered | Status |
| --- | --- | --- | --- |
| D1 | Architecture diagram: components | diagram/architecture.excalidraw | source final, PNG export is a manual task |
| D2 | Diagram: data flows and request flows | diagram | source final, PNG export is a manual task |
| D3 | Diagram: technology choices | diagram | source final, PNG export is a manual task |
| D4 | Editable source + image sent | diagram/ | not yet sent |
| D5 | Notes, max 5 pages (optional) | docs/design.md | drafted, 5 pages in compact A4 |
| D6 | AI workflow note a-d | submission/ai-workflow-note.md | revised, in my review |
| D7 | Session export attached | submission/transcript/ | manual task, missing |
| C1 | Rendering strategy, both apps | design.md section 2 | drafted |
| C2 | API layer, both apps | design.md section 3 | drafted |
| C3 | Data access, both apps | design.md section 4 | drafted |
| C4 | Auth, both apps (roles admin/analyst/viewer, per-collection features) | design.md section 5 | drafted; early-access owner is an assumption |
| C5 | Dashboard fast and fresh at 500k x 80, 300 users | design.md section 6 | drafted, not benchmarked |
| C6 | Participant app reliable on bad networks (RTL, multi-language, conversational AI) | design.md section 7 | drafted |
| C7 | 2-3 biggest trade-offs and what was rejected | docs/adr/ + design.md section 9 | drafted |
| F1 | Live counts, charts per question type | ADR 0001, ADR 0004 | ADR drafted, in my review |
| F2 | Filters, segments, saved filter sets | ADR 0004 | ADR drafted, in my review |
| F3 | Cross-tabs | ADR 0004 | ADR drafted, in my review |
| F4 | AI executive summary | ADR 0005 | ADR drafted, in my review |
| F5 | Chat with your data, streaming | ADR 0005, ADR 0003 | ADR drafted, in my review; AI contract unconfirmed |
| F6 | Transcript browsing, exports | ADR 0004 | ADR drafted, in my review; approval policy pending |
| F7 | Sensitive health data | All ADRs, docs/risks.md | ADRs drafted, in my review; Surgo policies pending (R23) |
| I1 | Be ready to change the diagram live | diagram/README.md | todo (mine) |