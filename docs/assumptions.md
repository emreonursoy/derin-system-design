# Assumptions and derivations

Rule: every figure is DERIVED (show the arithmetic) or ASSUMED (say why). Update when a decision changes.

## Inputs
| Input | Value | Source |
| --- | --- | --- |
| Rows per large collection | 500k responses x 80 questions = 40M | brief |
| Dashboard concurrent users | 300 | brief |
| Concurrent participants | 10,000 | my target (brief says "thousands") |
| Cloud | Azure | my choice |

## To derive (AI drafts, I check)
- Answer write rate: average and design peak
- Concurrent AI streams and LLM tokens per second
- Dashboard queries per second and cache hit assumption
- Storage per large collection (rows, indexes, transcripts)
- Replicas, connections, pool sizes