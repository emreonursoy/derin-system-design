# Final submission checklist

The architecture is frozen. Unverified Derin contracts, performance targets, privacy policies and operational assumptions are documented in `docs/risks.md`. They aren't blockers.

## Ready
- [x] Editable diagram source: `diagram/architecture.excalidraw`
- [x] Notes within 5 A4 pages: `docs/design.md`
- [x] Decisions and rejected options: `docs/adr/0001` to `0005`
- [x] Assumptions and risks: `docs/assumptions.md`, `docs/risks.md`
- [x] AI workflow note, sections a to c: `submission/ai-workflow-note.md`

## Manual tasks (required)
- [ ] **Export the PNG.** Follow `diagram/README.md`. The current `diagram/architecture.png` is out of date.
- [ ] **Check the PNG** for overlaps and unreadable text. The AI can read it for you.
- [ ] **Add the AI session transcript.** Export this Claude Code session into `submission/transcript/`, which is empty. Then put the file name in section (d) of the AI note.
- [ ] **Decide on the ChatGPT session.** The note credits ChatGPT, so attach that conversation too if you can.
- [ ] **Read the AI note,** and confirm that it's accurate in your own words.
- [ ] **Commit when you're ready.** Nothing is committed.

## Send to Surgo before the deadline
- [ ] `diagram/architecture.png`
- [ ] `diagram/architecture.excalidraw`
- [ ] `docs/design.md`, as optional notes (check the page count in the format you send)
- [ ] `submission/ai-workflow-note.md`
- [ ] Session transcript

## Before the interview
- [ ] Practise changing the diagram live: see `diagram/README.md`.
- [ ] Be ready to say plainly: nothing is benchmarked, the Derin contract is unknown, Application Gateway is an assumption, and Surgo's policies are pending.
