# Risks and what to test first

Standing rule from me: anything that can't be verified is marked ASSUMED or listed here. Risks aren't ranked, and their IDs are only stable labels. The validation list below is in priority order. Unverified Derin contracts, performance targets, privacy policies and operational assumptions stay here as documented risks.

## Validate before production
In priority order. This is a short list, not a production readiness program.

1. Real Postgres query performance (T1, T8)
2. SSE through the chosen ingress (T3, T6)
3. Authentication and authorization claims (T2)
4. Offline sync and recovery, including withdrawal races (T5, T9)
5. Data residency across external services (R6, R10, R11, R21a)
6. AI service integration contract (T7, R6)
7. Privacy and export policies (T4, R2a, R23)

## Tests
| # | Test | Why first | Decision it can change |
| --- | --- | --- | --- |
| T1 | Query benchmark at 500k x 80, with filters and cross-tabs, cache off | The 1s cold scan underlies every capacity estimate | Replica count, then columnar trigger (ADR 0001) |
| T2 | Role and region claims in the Better Auth JWT | Undocumented, and all access control relies on them | Regional role copy (ADR 0002) |
| T3 | SSE and long-poll through Application Gateway with WAF and Container Apps on a low-end phone. Include the request timeout, and whether open streams slow answer saves | Gateway need and SSE behaviour are assumptions, and Microsoft's two pages differ | Timeouts, long-poll by default, ingress choice, or a dedicated AI deployment (Q40, Q41) |
| T4 | Suppression recipes, through dashboard and chat: subtract `city in [big, small]` from `city in [big]`; compare equivalent filters; compare live polls on a slow day; subtract "completed" from "all"; rebuild a record from transcript references; narrow a filter with search terms; overlap two row-level exports; check that no invite or response ID reaches analytics, exports or AI context | One bug or gap here exposes health data | Controls in ADR 0004, then adding noise |
| T5 | First load on slow 3G against 150 KB of JS | A failed first load means no offline ability | Shell design (ADR 0003) |
| T6 | Connection setup time from target networks to the region | The cost in ADR 0002 uses assumed round trips | Batching and region choice |
| T7 | AI service integration: contract items in ADR 0003 and 0005, plus refusal behaviour in each survey language | Derin's interface isn't known | Orchestration mode, and whether a collection can have AI questions (Q39) |
| T8 | Projector and replica lag at burst, with `pg_stat_wal` | Write volume is modelled, not measured | Read table on its own server (ADR 0001) |
| T9 | Kill the projector mid-batch and an API instance mid-turn, and pause one past its lease. Withdraw during a batch, a save, a chat turn and a running export. Fail over the primary. Restore a backup, and replay the withdrawal log | Correctness rules in ADR 0001 are unproven | Projector mechanism, or the one-row fallback |

## Risks
| # | Risk | Harm | What we do | Status |
| --- | --- | --- | --- | --- |
| R1 | **A participant in distress isn't helped.** The content filter is not crisis detection, and is untested in Arabic, Swahili and other languages. Participants are pseudonymous, so reaching them may be impossible. | Serious harm to a person | Fixed support message per survey, required before AI questions run (ADR 0003). The client's safeguarding protocol sits outside these apps. | Open, needs a client protocol |
| R2 | **Someone is re-identified despite suppression.** Proposed controls reduce the risk, and can't remove it. Stories in transcripts can identify a person with no named entity. | Health data exposed | Controls and audit (ADR 0004). Transcripts are a separate permission. | Accepted, with controls |
| R2a | **Row-level export is outside the minimum group.** An admin can compute suppressed cells, and match overlapping exports on row content. Bands and row checks reduce this, and don't remove it. | Health data exposed by a trusted user | Audit, notice to all admins, optional second approver (ADR 0004). Say this openly to clients. | Accepted, with controls |
| R2b | **Stolen or misused admin account.** Second-admin approval is now optional (Q43), so one account can export. MFA enforcement per role is ASSUMED. | Bulk export that can't be recalled | MFA, recent sign-in, live check, audit, notice. Optional approval. | Open, test with T2 |
| R2c | **One signing key reaches every region.** | A stolen key opens all stacks | Key storage (ADR 0002). Key location isn't verified. | Open |
| R2d | **Interview AI gives health advice or is manipulated.** Prompt rules help, and aren't a guarantee. Whether the AI service enforces them is unconfirmed. | A participant acts on bad advice | One-response context, no tools and consent screen, as contract items (ADR 0003). | Open, needs the contract |
| R3 | **Redaction misses identifiers.** Microsoft notes detection varies by language and locale. | Identifiers reach researchers | Audited, permission-gated, and hidden until redacted. Test per language. | Accepted |
| R4 | **Exported files can't be recalled.** | Data remains outside after withdrawal | Admin only, audited, and stated on screen (Q18). | Accepted |
| R5 | **Cross-border transfer.** A Kenyan collection pinned to Sweden Central moves health data out of the country. We claim no specific law. | Legal or contractual problem for a client | No in-country promise (Q1). The client decides where to pin. | Accepted, client decides |
| R6 | **The AI service contract is unknown.** Its interface, processing location, retention, deletion on withdrawal, review of flagged text and capacity aren't confirmed, and it may not serve every region. | Health text in the wrong place, or no AI questions there | Fail closed (Q22, Q39). Shedding order (ADR 0002). | Open, needs the contract |
| R7 | **No volumetric DDoS protection on the gateways** (Q36). Microsoft says WAF doesn't stop these attacks, and recommends DDoS Protection. API traffic no longer has Front Door in front of it. | A survey goes down in the field | Decide before production, once cost is known. | Open, by my decision |
| R8 | **WAF false positives block real answers.** Microsoft documents this for JSON and free text. | Lost answers | Tune on test data. The participant can reword or skip, and we count blocks with no content (ADR 0003). | Open, needs testing |
| R9 | **The AI provider may store flagged text for human review.** Azure-hosted models do this by default, unless modified monitoring is approved. Whether it applies depends on the contract. | Health text stored, and possibly read by provider staff | Contract item. The consent screen says so if it applies. | Open, needs the contract |
| R10 | **Regional processing is not fully verified** for Azure Language. | Text may be processed outside the assumed boundary | Regional resource. Confirm in Microsoft's data terms. | ASSUMED |
| R11 | **Vendor data handling is unverified:** Inngest, PostHog and Sentry hosting regions, Inngest's step storage, and how Inngest reaches a worker in an internal environment. | Metadata in another region | IDs only in Inngest. No content in the others. | ASSUMED |
| R12 | **Dashboard capacity is unproven, and one replica may not be enough.** The 70-core estimate rests on an assumed 1s scan. Losing the only replica stops filtered views and chat tools until it's replaced. Redis down with one replica means "busy" responses. | Slow or refused dashboard queries, while saves continue | T1, T8. Measured escalation in ADR 0001. | Open |
| R13 | **Open filtered views, saved or ad-hoc, can show data about 67s old** (Q37). | A researcher may read older figures | Visible "as of", refresh button. Only unfiltered views stay within 10s. A columnar store is the trigger if clients need fresher filtered views (ADR 0001). | Accepted |
| R14 | **A single control plane.** An outage in its home region stops dashboards everywhere within 5 minutes. | Researchers lose access, interviews continue | Zone redundancy within the region (ASSUMED). | Accepted |
| R15 | **Dashboard users' details may leave their region** (Q30). | Contractual concern for some clients | Stated openly. | Accepted |
| R16 | **Stale access for up to 5 minutes** after removal, except for row-level and transcript exports. | A removed user keeps read access briefly | Stated openly (Q29). | Accepted |
| R17 | **Open streams and live AI calls share the API deployment,** and response buffering is off across the gateway, so slow phones hold connections longer. | Slower answer saves | T3. A dedicated AI deployment is the fix (Q41). | Open, needs testing |
| R18 | **Search is weak in some languages,** with no pgvector (Q15). | Transcript answers are incomplete | Say so in the UI. | Accepted |
| R19 | **Cost isn't estimated anywhere.** Gateway, replicas, Redis and DDoS Protection all add up per region. | Design may be too expensive to run | Estimate before committing to a second region. | Open |
| R20 | **Replica lag under sustained writes.** Microsoft warns it can grow. | Ad-hoc results older than 10s | "As of" time. Alert at 4s. Read table on its own server. | Open |
| R21 | **Multiple-choice rows aren't sized,** and row size and encoding are assumed. | Storage and scan estimates are low | T1 and T8. | Open |
| R21a | **Backup location isn't verified,** especially for South Africa, where the other region is restricted. Withdrawn data stays in backups until they age out. | Data outside the expected region | Check geo-redundant settings before production. | ASSUMED |
| R21b | **Proposed controls may frustrate researchers.** Filtered numbers are inexact, and rare values can't be filtered on. | Lower product value | Policy under review (Q44). Explain in the UI. | Open, needs Surgo's policy |
| R21c | **No cross-device recovery.** A replaced phone or cleared storage stops the participant from continuing, and a forwarded link can bind first and lock out the intended person. | Fewer finished surveys | Synced answers count as partial. A client can revoke a forwarded link, which stops further use, and doesn't restore the intended person. Recovery is a product assumption (Q50). | Accepted, needs Surgo's view |
| R21e | **Org query limits may not fit one large client.** With all 300 users in one org, 17.5 queries/s at 1s needs about 18 concurrent (D). The number of orgs isn't assumed anywhere. | Slow or refused filtered views | Set limits from T1 and real org sizes. | Open |
| R21f | **A leaked export link works from anywhere for 15 minutes.** Blob Storage has a public endpoint so browsers can download. Revocation can be delayed. | A row-level health file is exposed | Link goes only to the requester, short expiry, audit, file deleted after 24 hours. | Accepted, with controls |
| R21g | **A Redis failover blanks held small segments** until 10 new responses arrive. A missing key counts as held, so nothing releases early. | Small segments unavailable for a while | Accept for now. Moving hold state to Postgres is the fix if it proves disruptive. | Accepted |
| R22 | **In-app browsers and storage eviction.** Links opened inside messaging apps may limit storage and service workers, and a later open in another browser after answering is refused. Not tested. | Offline features fail, and more restarts | Test on target phones. Online path still works. | Open |
| R25 | **Counts are per invitation, not per verified person.** One person with two invitations counts twice, and a forwarded invitation counts once. | The minimum group can overstate distinct people | Stated in the docs. No identity resolution. | Accepted |
| R27 | **Filtered aggregates can include a withdrawn response** until the replica replays the withdrawal, plus cache time. Lag has no fixed bound. | A withdrawn person stays in a filtered count for a while | "As of" time, lag alert. Row-level and transcript paths check the primary. | Accepted |
| R26 | **Primary Postgres failure isn't sized or tested.** High availability and failover time are ASSUMED. | Saves wait on phones, and AI turns and audited actions stop | Behaviour in ADR 0002. T9 fails it over. | Open |
| R23 | **Surgo's policies aren't known:** disclosure values, export approval, retention, and who activates early access (Q43, Q44, Q46). | Design values may be wrong for Surgo | All marked proposed or assumed. Central enforcement makes changing them cheap. | Open, needs Surgo |
| R24 | **Skip logic leaves coded branch flags on the phone** until the response finishes. A flag can imply an answer, and on a shared phone it may stay if the app never runs again. | Limited exposure on the device | Coded only, "clear this device", cleanup on next run. | Accepted |
