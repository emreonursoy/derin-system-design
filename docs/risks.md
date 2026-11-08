# Risks and what to test first

Standing rule from me: anything that can't be verified is marked ASSUMED or listed here. Ranked by harm if it goes wrong.

## Test first
| # | Test | Why first | Decision it can change |
| --- | --- | --- | --- |
| T1 | Query benchmark at 500k x 80, with filters and cross-tabs, cache off | The 1s cold scan underlies all dashboard sizing | Replica count, then columnar trigger (ADR 0001) |
| T2 | Role and region claims in the Better Auth JWT | Undocumented, and all access control relies on them | Regional role copy (ADR 0002) |
| T3 | SSE and long-poll through Application Gateway with WAF on a low-end phone, including what the request timeout does to a live stream | Microsoft's two pages differ | Gateway timeouts, or long-poll by default (ADR 0003) |
| T4 | Suppression recipes, through dashboard and chat: subtract `city in [big, small]` from `city in [big]`; compare equivalent filters; compare live polls on a slow day; subtract "completed" from "all"; rebuild a record from transcript references; narrow a filter with search terms; overlap two row-level exports | One bug or gap here exposes health data | Controls in ADR 0004, then adding noise |
| T5 | First load on slow 3G against 150 KB of JS | A failed first load means no offline ability | Shell design (ADR 0003) |
| T6 | Connection setup time from target networks to the region | The cost in ADR 0002 uses assumed round trips | Batching and region choice |
| T7 | Interview quality and content filter behaviour in each survey language | Classifiers aren't trained on many target languages | Whether a model counts as suitable (Q35) |
| T8 | Projector and replica lag at burst, with `pg_stat_wal` | Write volume is modelled, not measured | Read table on its own server (ADR 0001) |

## Risks
| # | Risk | Harm | What we do | Status |
| --- | --- | --- | --- | --- |
| R1 | **A participant in distress isn't helped.** The content filter is not crisis detection, and is untested in Arabic, Swahili and other languages. Participants are pseudonymous, so reaching them may be impossible. | Serious harm to a person | Fixed support message per survey, required before AI questions run (ADR 0003). The client's safeguarding protocol sits outside these apps. | Open, needs a client protocol |
| R2 | **Someone is re-identified despite suppression.** Comparison controls reduce the risk, and can't remove it. Stories in transcripts can identify a person with no named entity. | Health data exposed | Controls and audit (ADR 0004). Transcripts are a separate permission. | Accepted, with controls |
| R2a | **Row-level export is outside the minimum of 10.** An admin can compute suppressed cells, and match overlapping exports on row content. Bands and row checks reduce this, and don't remove it. | Health data exposed by a trusted user | Second approver, notification, audit (ADR 0004). Say this openly to clients. | Accepted, with controls |
| R2b | **Stolen admin account.** MFA enforcement per role is ASSUMED. | Bulk export that can't be recalled | MFA, recent sign-in, second approver. | Open, test with T2 |
| R2c | **One signing key reaches every region.** | A stolen key opens all stacks | Key storage (ADR 0002). Key location isn't verified. | Open |
| R2d | **Interview model gives health advice or is manipulated.** Prompt rules help, and aren't a guarantee. | A participant acts on bad advice | One-response context, no tools, consent screen (ADR 0003). | Accepted, with controls |
| R3 | **Redaction misses identifiers.** Microsoft notes detection varies by language and locale. | Identifiers reach researchers | Audited, permission-gated, and hidden until redacted. Test per language. | Accepted |
| R4 | **Exported files can't be recalled.** | Data remains outside after withdrawal | Admin only, audited, and stated on screen (Q18). | Accepted |
| R5 | **Cross-border transfer.** A Kenyan collection pinned to Sweden Central moves health data out of the country. We claim no specific law. | Legal or contractual problem for a client | No in-country promise (Q1). The client decides where to pin. | Accepted, client decides |
| R6 | **AI is unavailable in a region.** South Africa North has no Standard/Regional chat model. Provisioned cost isn't known. Our quota tier isn't known. | No AI questions there, or high cost | Fail closed (Q22). Degradation order (ADR 0002). | Open, needs pricing |
| R7 | **No volumetric DDoS protection on the gateways** (Q36). Microsoft says WAF doesn't stop these attacks, and recommends DDoS Protection. API traffic no longer has Front Door in front of it. | A survey goes down in the field | Decide before production, once cost is known. | Open, by my decision |
| R8 | **WAF false positives block real answers.** Microsoft documents this for JSON and free text. | Lost answers | Tune on test data. The participant can reword or skip, and we count blocks with no content (ADR 0003). | Open, needs testing |
| R9 | **Abuse monitoring stores flagged prompts**, with possible human review, unless modified monitoring is approved. | Health text is stored and may be read by Microsoft staff | Apply for it. Approval is ASSUMED. | Open |
| R10 | **Regional processing is not fully verified** for Azure Language. | Text may be processed outside the assumed boundary | Regional resource. Confirm in Microsoft's data terms. | ASSUMED |
| R11 | **Vendor data handling is unverified:** Inngest, PostHog and Sentry hosting regions, and Inngest's step storage. | Metadata in another region | IDs only in Inngest. No content in the others. | ASSUMED |
| R12 | **Dashboard capacity is unproven.** 70 cores with no cache rests on an assumed 1s scan, with no burst, and excludes exports, search and reconciliation. Losing any replica overloads the rest. An 80-chart view may cost far more. NestJS sizing excludes streams and model calls. | Slow dashboards and interviews | T1, T8. Escalation path in ADR 0001. | Open |
| R13 | **Open filtered views, saved or ad-hoc, can show data about 67s old** (Q37). | A researcher may read older figures | Visible "as of", refresh button. Only unfiltered views stay within 10s. A columnar store is the trigger if clients need fresher filtered views (ADR 0001). | Accepted |
| R14 | **A single control plane.** A Sweden Central outage stops dashboards everywhere within 5 minutes. | Researchers lose access, interviews continue | Zone redundancy within the region (ASSUMED). | Accepted |
| R15 | **Dashboard users' details may leave their region** (Q30). | Contractual concern for some clients | Stated openly. | Accepted |
| R16 | **Stale access for up to 5 minutes** after removal, except for row-level export. | A removed user keeps read access briefly | Stated openly (Q29). | Accepted |
| R17 | **Slow phones hold backend connections,** since response buffering is off across the gateway. | Fewer connections for others | Keep responses small. Second gateway is the fallback. | Open, needs testing |
| R18 | **Search is weak in some languages,** with no pgvector (Q15). | Transcript answers are incomplete | Say so in the UI. | Accepted |
| R19 | **Cost isn't estimated anywhere.** Gateways, replicas, Redis, DDoS Protection and models all add up per region. | Design may be too expensive to run | Estimate before committing to a second region. | Open |
| R20 | **Replica lag under sustained writes.** Microsoft warns it can grow. | Ad-hoc results older than 10s | "As of" time. Alert at 4s. Read table on its own server. | Open |
| R21 | **Multiple-choice rows aren't sized,** and row size and encoding are assumed. | Storage and scan estimates are low | T1 and T8. | Open |
| R21a | **Backup location isn't verified,** especially for South Africa, where the other region is restricted. Withdrawn data stays in backups until they age out. | Data outside the expected region | Check geo-redundant settings before production. | ASSUMED |
| R21b | **Rounding and rare-value limits frustrate researchers.** Filtered numbers are inexact, and rare values can't be filtered on. | Lower product value | Explain in the UI. Tune once tests show what's needed. | Accepted |
| R21c | **Resume code costs completions** on shared or replaced phones. | Fewer finished surveys | Allow a fresh start where the survey permits. | Accepted |
| R21d | **Turn recovery depends only on an Inngest cron job.** During an Inngest outage, a dead turn never returns to `received` or gets marked unanswered, and the participant keeps seeing "the reply is coming". Inngest's hosting is also unverified. | Participants stuck on an AI question | A sweeper in NestJS, with a lock like the projector's, is the fallback. | Open |
| R21e | **The query limit of 5 per organization doesn't fit one large client.** With all 300 users in one org, 17.5 queries/s at 1s needs about 17 concurrent (D), so "busy" becomes normal. The number of orgs isn't assumed anywhere. | Slow or refused filtered views | Tune the limit per org, once real org sizes are known. | Open |
| R21f | **A leaked export link works from anywhere for 15 minutes.** Blob Storage has a public endpoint so browsers can download. Revocation can be delayed. | A row-level health file is exposed | Link goes only to the requester, short expiry, audit, file deleted after 24 hours. | Accepted, with controls |
| R21g | **A Redis failover blanks held small segments** until 10 new responses arrive. A missing key counts as held, so nothing releases early. | Small segments unavailable for a while | Accept for now. Moving hold state to Postgres is the fix if it proves disruptive. | Accepted |
| R22 | **In-app browsers and storage eviction.** Links opened inside messaging apps may limit storage and service workers. Not tested. | Offline features fail for some participants | Test on target phones. Online path still works. | Open |
