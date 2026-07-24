# Evaluation Prompt — Hallucination Rate

The second of PRD Genie's three production metrics, and the one the whole architecture exists to hold down. It answers: **of everything the PRD asserts, how much cannot be traced back to what the sources actually said?**

Where extraction completeness measures *recall* against the source, this measures *precision* of the output. It judges the **final PRD and user stories**, not the extraction — that is what a stakeholder reads and what engineering builds from, and it is where template-completion pressure does its damage.

Report it **next to extraction completeness, always**. Hallucination rate alone is trivially gamed: an agent that abstains from every section scores a perfect 0.0. That is why the two numbers only mean something together.

> **Scoped grounding (updated).** An extracted item counts as a hallucination only if it is unsupported by the PRIMARY chat request AND every supporting document. Detail legitimately pulled from a supporting document to enrich an in-scope requirement is supported, not invented.


---

## The judge prompt

```
ROLE: You are an evaluation judge measuring whether a generated PRD asserts
anything that its source documents did not support. You are not judging whether
the PRD is well written, well structured, or whether its requirements are good
product decisions. You are checking one thing: is every claim traceable to the
source?

INPUT: You receive two things.
  1. SOURCE — the original transcript(s), brief, or notes. If several documents
     are present they are delimited by `=== SOURCE: <filename> ===`.
  2. OUTPUT — the generated PRD, and the user stories if supplied.

METHOD:

STEP 1 — Enumerate the claims in the OUTPUT.
A claim is any substantive assertion about the product: a functional or
non-functional requirement, an acceptance criterion, a persona and its stated
need, a success metric, a constraint, a dependency, a timeline entry, an
assumption, an out-of-scope statement, or a user story. Assign IDs
CL-01, CL-02, ...

Do NOT count as claims:
  - a section reading "Not discussed in this meeting." or equivalent. That is an
    abstention. It asserts nothing and cannot be a hallucination.
  - document scaffolding: title, version number, author, date, status field,
    section headings, table headers.
  - items the OUTPUT itself labels as open, ambiguous, or unresolved — content
    in an Open Questions or Clarification Questions section is the system
    reporting uncertainty, which is the behaviour we want.
  - restatements of the same claim in two sections. Count it once.

STEP 2 — Judge each claim against the SOURCE.
  SUPPORTED (0)          — the source states this. Wording may differ; meaning
                           and specifics must match.
  EMBELLISHED (1)        — a real claim carrying an invented specific. The
                           source said "fast"; the PRD says "under 200ms". The
                           source named a persona; the PRD invented their
                           current workaround.
  RESOLVED_AMBIGUITY (1) — the source raised this but did NOT decide it, and
                           the PRD states it as settled. Suggestions, "TBD",
                           "let's discuss offline" and open questions are not
                           decisions. This is the most dangerous category
                           because it reads as diligence.
  FABRICATED (1)         — the source contains no basis for this at all.

Also assign a severity to each non-SUPPORTED claim:
  CRITICAL — engineering could build the wrong thing from it (a requirement,
             a numeric target, an acceptance criterion, a dependency)
  MODERATE — misleads planning but is unlikely to be built directly (a
             timeline entry, a success metric, an assumption)
  MINOR    — cosmetic or contextual (a persona description detail)

STEP 3 — Score.
  hallucination_rate = count(non-SUPPORTED claims) / count(all claims)
  Report as a decimal 0.000-1.000, rounded to 3 places. LOWER IS BETTER.
  Also report critical_hallucination_rate over the same denominator.

RULES:
1) Absence of evidence is the test, not implausibility. A claim can be entirely
   reasonable, standard for this kind of product, and still be a hallucination.
   "Users have modern browsers" is sensible and unsupported.
2) An abstention is never a hallucination. A PRD where seven of ten sections
   read "Not discussed in this meeting." and the other three are fully
   supported scores 0.000. That is a correct PRD for a thin source.
3) Do NOT penalise omission. Content the source stated but the PRD left out is
   a completeness failure, measured by the other metric. Ignore it here.
4) Traceability aids are evidence, not claims. A Source column citing REQ-004,
   or a story citing FR-002, is not itself a claim — but if the cited ID does
   not correspond to anything in the source, the claim it supports is
   FABRICATED.
5) When several sources disagree, a claim supported by any one of them is
   SUPPORTED. But a PRD that silently picks a winner between two conflicting
   statements is RESOLVED_AMBIGUITY — the conflict should have been surfaced,
   not decided.
6) A PRD with zero claims (everything abstained) scores 0.000. Note it in the
   summary, because that number is only meaningful beside completeness.

OUTPUT: Return JSON only, no prose, no code fences.
{
  "claims": [
    {"id": "CL-01", "claim": "<what the PRD asserts>",
     "section": "<PRD section it appears in>",
     "verdict": "SUPPORTED | EMBELLISHED | RESOLVED_AMBIGUITY | FABRICATED",
     "severity": "CRITICAL | MODERATE | MINOR | null",
     "source_quote": "<verbatim from source if SUPPORTED, else empty>",
     "note": "<one line: what was invented>"}
  ],
  "total_claims": 0,
  "unsupported_claims": 0,
  "hallucination_rate": 0.0,
  "critical_hallucination_rate": 0.0,
  "abstentions": 0,
  "by_verdict": {"SUPPORTED": 0, "EMBELLISHED": 0, "RESOLVED_AMBIGUITY": 0, "FABRICATED": 0},
  "summary": "<two sentences: what was invented, and where it came from>"
}
```

---

## Worked example — `sample_product_brief.txt`

Same source as the completeness evaluation, so the two scores describe one run and can be read together.

### Source (abridged)

> **OVERVIEW** — analytics dashboard giving business users visibility into key metrics without writing SQL or asking the data team. The #1 requested feature from Enterprise customers.
> **TARGET USERS** — Business analysts who currently export data to Excel · Team leads who need weekly performance summaries · Executives who want a high-level overview
> **KEY REQUIREMENTS** — 5 core metrics · filter by date range · refresh every 15 minutes · role-based access · export to PDF
> **CONSTRAINTS** — PostgreSQL warehouse · no third-party analytics tools · mobile responsive · target launch Q3 2026
> **OPEN QUESTIONS** — Custom layouts or fixed? · Alerting? · Acceptable page load time? *(Suggestion: under 3 seconds)*

### Run A — the counterfactual: no empty-section rule

This is what the PRD Generator produces with the template but without rule 1 (`Not discussed in this meeting.`). It is the failure this project was designed around.

| CL | Claim | Verdict | Sev |
|---|---|---|---|
| CL-01 | Overview: self-serve analytics without SQL | SUPPORTED | — |
| CL-02 | Business goal: #1 Enterprise customer request | SUPPORTED | — |
| CL-03 | User goal: visibility without the data team | SUPPORTED | — |
| CL-04 | FR: display 5 named core metrics | SUPPORTED | — |
| CL-05 | FR: filter by date range (7/30/90/custom) | SUPPORTED | — |
| CL-06 | NFR: refresh every 15 minutes | SUPPORTED | — |
| CL-07 | FR: role-based access, exec vs team lead | SUPPORTED | — |
| CL-08 | FR: export to PDF for board reports | SUPPORTED | — |
| CL-09 | Constraint: integrate with PostgreSQL warehouse | SUPPORTED | — |
| CL-10 | Constraint: no third-party analytics tools | SUPPORTED | — |
| CL-11 | NFR: mobile responsive | SUPPORTED | — |
| CL-12 | Timeline: target launch Q3 2026 | SUPPORTED | — |
| CL-13 | Persona: business analyst, exports to Excel today | SUPPORTED | — |
| CL-14 | Stakeholder: Priya Sharma, Product Manager | SUPPORTED | — |
| CL-15 | Persona: team lead — *"currently compiles weekly decks by hand"* | **EMBELLISHED** | MINOR |
| CL-16 | Persona: executive — *"currently asks analysts for summaries"* | **EMBELLISHED** | MINOR |
| CL-17 | NFR: page load under 3 seconds | **RESOLVED_AMBIGUITY** | CRITICAL |
| CL-18 | FR: alert when churn exceeds 5% | **RESOLVED_AMBIGUITY** | CRITICAL |
| CL-19 | Out of scope: custom layouts deferred to v2 | **RESOLVED_AMBIGUITY** | MODERATE |
| CL-20 | Success metric: 60% adoption within one quarter | **FABRICATED** | MODERATE |
| CL-21 | Success metric: 40% fewer ad-hoc data requests | **FABRICATED** | MODERATE |
| CL-22 | Assumption: users have modern browsers | **FABRICATED** | MODERATE |
| CL-23 | Timeline: design complete May 2026 | **FABRICATED** | MODERATE |
| CL-24 | Timeline: development starts June 2026 | **FABRICATED** | MODERATE |

**24 claims · 10 unsupported → hallucination_rate = 10 / 24 = 0.417**
**critical = 2 / 24 = 0.083**

> ### Suggested score: **0.417** — with extraction completeness at 0.893
>
> Read together: the extractor was faithful, and the *generator* invented. Every one of the ten unsupported claims sits in a section the brief did not cover — Success Metrics, Timeline, Assumptions, Out of Scope. That localises the fault to Agent 2 and to template pressure, not to extraction quality. Without the paired completeness figure you could not tell those apart.
>
> The three RESOLVED_AMBIGUITY entries are the most instructive. CL-17 turns a bracketed *suggestion* into a binding performance target, and CL-18 turns an open question into a feature. Neither is invented from nothing — which is exactly why they survive a read-through, and why they are graded as harshly as outright fabrication.

### Run B — with the guardrails in place

Rule 1 sends unsupported sections to `Not discussed in this meeting.`; rule 6 routes open questions to Open Questions; the extractor's `success_metrics` array stays empty because no measurable target was stated.

CL-15 through CL-24 disappear. Four sections abstain. The remaining 14 claims are all SUPPORTED.

**14 claims · 0 unsupported → hallucination_rate = 0.000 · abstentions = 4**

### The pairing, which is the actual result

| | Run A (no guardrails) | Run B (guarded) |
|---|---|---|
| Extraction completeness ↑ | 0.893 | 0.893 |
| Hallucination rate ↓ | **0.417** | **0.000** |
| Critical ↓ | 0.083 | 0.000 |
| Claims asserted | 24 | 14 |
| Abstentions | 0 | 4 |

Hallucination fell to zero **while completeness held flat**. That is the claim worth making, and the flat completeness figure is what makes it credible — it rules out the cheap explanation, which is that the system simply learned to say less. It did say less: ten fewer claims. But it lost nothing the source had actually stated.

---

## Logging to Langfuse

Both metrics attach to the same trace as 0–1 numerics, so they can be charted and filtered together:

```json
{
  "type": "score-create",
  "body": {
    "traceId": "<trace for this run>",
    "name": "hallucination_rate",
    "value": 0.417,
    "dataType": "NUMERIC",
    "comment": "10/24 unsupported · 2 critical · 5 fabricated, 3 resolved-ambiguity, 2 embellished"
  }
}
```

Mind the polarity when you read a chart: **completeness higher is better, hallucination rate lower is better.** Say so in the writeup — a reader seeing 0.893 and 0.000 should not have to work out which direction is good.

---

## Using this in your submission

**Q3 §7 (Evaluation strategy).** State the unit of analysis — claims in the final PRD — and that abstentions are excluded from the denominator. A hallucination rate whose denominator nobody defined is not a measurement.

**Q3 §8 (Testing observations).** The Run A / Run B table is a complete before-and-after: one variable changed (the empty-section rule), both metrics reported, the improvement attributable.

**Q4 (Reflection).** The four-way verdict scale is the point worth reflecting on. Before building this, "hallucination" reads as a single thing. Grading real output splits it into fabrication, embellishment, and resolved ambiguity — and the last is both the most common and the hardest to spot, because it is built from things that really were said.

**One honest caveat.** Run A is a reconstruction of what an unguarded generator produces from this brief, not a logged run — I built it to show the metric working across all four verdict types. Before it goes in your writeup, either generate it for real by temporarily removing rule 1 from the PRD Generator and re-running, or label it explicitly as illustrative. A number presented as measured when it was estimated is the same failure this metric exists to catch.
