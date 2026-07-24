# Evaluation Prompt — Extraction Completeness

One of PRD Genie's three production metrics.
> **Scoped to the chat request (updated).** In the primary/supporting design the chat message defines scope, so the judge now builds ground truth from the **PRIMARY REQUEST only** — never from the supporting documents. Counting a supporting-doc requirement the user never asked for would penalise the system for correctly ignoring it (this is what produced misleading scores like 0.033 on the SSO test, where the judge measured a one-line request against a full dashboard corpus). Hallucination grounding still uses the full source, so legitimate enrichment from a supporting document about an in-scope requirement is *supported*, not invented.
 It answers: **of everything the source actually stated, how much did the extractor capture?**

Run it as an LLM-as-judge pass (gpt-4o, temperature 0) over each baseline test, or by hand for a small sample. Always report it **next to hallucination rate** — completeness alone rewards an agent that extracts everything including things nobody said, and hallucination rate alone rewards an agent that extracts almost nothing. Neither number means much without the other.

---

## The judge prompt

```
ROLE: You are an evaluation judge measuring how completely a requirements
extraction agent captured what a source document actually stated. You are not
judging writing quality, formatting, or whether the requirements are good ideas.
You are measuring recall against the source.

INPUT: You receive two things.
  1. SOURCE — the original transcript, brief, or notes.
  2. EXTRACTION — the JSON the Requirement Extractor produced from it.

METHOD: Work in this order. Do not skip step 1.

STEP 1 — Build the ground truth BEFORE looking at the extraction.
Read the SOURCE alone and list every distinct item that was explicitly stated:
  - functional requirements (something the product must do)
  - non-functional requirements (performance, security, scale, compatibility)
  - constraints (budget, timeline, technical, dependency)
  - personas (types of end user named in the source)
  - stakeholders (people named, with a position)
  - product overview / business goal / user goal, if stated
  - success metrics, but ONLY where a measurable target is given
Assign each an ID: GT-01, GT-02, ...

Do NOT include in ground truth:
  - topics raised but left undecided ("TBD", "let's discuss offline") — those
    belong in ambiguous_items and are not part of this metric
  - things a reasonable PRD would usually contain but this source never said
  - your own inferences about what the product probably needs

STEP 2 — Match each ground truth item against the EXTRACTION.
Award one of:
  FULL    (1.0) — captured, with its specific detail intact (numbers, versions,
                  named values preserved exactly)
  PARTIAL (0.5) — captured, but a stated detail was lost, generalised, or
                  several distinct stated items were collapsed into one
  MISCLASS(0.5) — present in the extraction but in the wrong bucket, e.g. a
                  stated requirement filed under ambiguous_items
  MISSING (0.0) — absent from the extraction entirely

STEP 3 — Score.
  completeness = sum(scores) / count(ground truth items)
  Report it as a decimal between 0 and 1, rounded to 3 places (0.893, not 89.3%).
  The per-item verdicts are already on this scale, so the aggregate stays on it.

RULES:
1) Judge only against the SOURCE. If the extraction contains something the
   source never stated, that is a hallucination — note it, but do NOT let it
   raise or lower the completeness score. It belongs to the other metric.
2) Wording may differ. "Results must load in under 2 seconds" captured as a
   non-functional requirement with target "under 2 seconds" is FULL, even if
   phrased differently. Meaning preserved is what matters.
3) Numbers, versions and identifiers must survive exactly. "10,000 concurrent
   users" recorded as "10K users" is PARTIAL, not FULL — rounding a stated
   figure loses information an engineer needs.
4) Collapsing is PARTIAL, not FULL. Five named metrics recorded as "display
   core metrics" loses four stated items.
5) Splitting is not penalised. One requirement captured as two well-formed
   requirements is FULL, provided nothing was invented.
6) An empty or near-empty source with an empty extraction scores 1.0. There
   was nothing to capture, and capturing nothing is correct.

OUTPUT: Return JSON only.
{
  "ground_truth": [
    {"id": "GT-01", "item": "<what the source stated>",
     "quote": "<verbatim from source>", "category": "FUNCTIONAL | NFR | CONSTRAINT | PERSONA | STAKEHOLDER | OVERVIEW | GOAL | METRIC"}
  ],
  "matches": [
    {"id": "GT-01", "verdict": "FULL | PARTIAL | MISCLASS | MISSING",
     "score": 1.0, "extraction_id": "<REQ-00n, or null>",
     "note": "<one line: what was lost, if anything>"}
  ],
  "completeness": 0.0,                 // 0.000 - 1.000
  "ground_truth_count": 0,
  "points_awarded": 0.0,
  "missed_items": ["<GT ids scored 0.0>"],
  "degraded_items": ["<GT ids scored 0.5>"],
  "hallucinations_noted": ["<extraction items with no basis in the source — recorded, not scored>"],
  "summary": "<two sentences: what was captured well, what was lost>"
}
```

---

## Worked example — `sample_product_brief.txt`

### Source (abridged to the graded content)

> **OVERVIEW** — We are building an analytics dashboard that gives business users visibility into key metrics without needing to write SQL queries or ask the data team. This is the #1 requested feature from our Enterprise customers.
>
> **TARGET USERS** — Business analysts who currently export data to Excel · Team leads who need weekly performance summaries · Executives who want a high-level overview
>
> **KEY REQUIREMENTS**
> 1. Dashboard should display 5 core metrics: revenue, active users, churn rate, NPS score, and support ticket volume
> 2. Users should be able to filter by date range (last 7 days, 30 days, 90 days, custom)
> 3. Data should refresh every 15 minutes (not real-time, to manage cost)
> 4. Must support role-based access: executives see all data, team leads see their team only
> 5. Export to PDF for monthly board reports
>
> **CONSTRAINTS** — Must integrate with existing PostgreSQL data warehouse · Cannot use third-party analytics tools · Must be accessible on mobile · Target launch Q3 2026
>
> **OPEN QUESTIONS** — Custom layouts or fixed? · Alerting? · Acceptable page load time? *(Suggestion: under 3 seconds)*

### Ground truth: 14 items

| ID | Item | Category |
|---|---|---|
| GT-01 | Dashboard displays 5 named core metrics | FUNCTIONAL |
| GT-02 | Filter by date range, incl. 7/30/90-day and custom | FUNCTIONAL |
| GT-03 | Data refreshes every 15 minutes | NFR |
| GT-04 | Role-based access: executives all, team leads own team | FUNCTIONAL |
| GT-05 | Export to PDF for board reports | FUNCTIONAL |
| GT-06 | Integrate with existing PostgreSQL warehouse | CONSTRAINT |
| GT-07 | No third-party analytics tools | CONSTRAINT |
| GT-08 | Mobile accessible / responsive | NFR |
| GT-09 | Target launch Q3 2026 | CONSTRAINT |
| GT-10 | Persona: business analyst (workaround: exports to Excel) | PERSONA |
| GT-11 | Persona: team lead (weekly summaries) | PERSONA |
| GT-12 | Persona: executive (high-level overview) | PERSONA |
| GT-13 | Overview: self-serve analytics without SQL or the data team | OVERVIEW |
| GT-14 | Business goal: #1 Enterprise customer request | GOAL |

Note what is **not** ground truth: the three OPEN QUESTIONS. "Under 3 seconds" is a suggestion, not a decision — it belongs in `ambiguous_items`, and counting it here would penalise the extractor for behaving correctly.

### Extraction under test — the pre-fix run

This is the state you hit: the extractor had no `product_overview` or `goals` field, and collapsed the metrics list.

| GT | Verdict | Score | Note |
|---|---|---|---|
| GT-01 | PARTIAL | 0.5 | Captured as "display core metrics" — the five named metrics were lost |
| GT-02 | FULL | 1.0 | Ranges preserved |
| GT-03 | FULL | 1.0 | "every 15 minutes" exact |
| GT-04 | FULL | 1.0 | Both access levels captured |
| GT-05 | FULL | 1.0 | |
| GT-06 | FULL | 1.0 | PostgreSQL named |
| GT-07 | FULL | 1.0 | |
| GT-08 | FULL | 1.0 | |
| GT-09 | FULL | 1.0 | Q3 2026 exact |
| GT-10 | FULL | 1.0 | `current_workaround` populated |
| GT-11 | FULL | 1.0 | |
| GT-12 | FULL | 1.0 | |
| GT-13 | MISSING | 0.0 | No schema field existed for it |
| GT-14 | MISSING | 0.0 | No schema field existed for it |

**Sum = 12.5 / 14 → completeness = 0.893**

> **Suggested score: 0.893 — with hallucination rate at 0.0.**
>
> Read together, those two numbers say the extractor is trustworthy but lossy: nothing invented, but the product's purpose and one requirement's specifics never made it through. That is exactly the right diagnosis. A completeness figure reported alone would have looked like a decent pass; paired with a clean hallucination rate it pointed straight at a schema gap rather than a prompt failure.

### After the schema fix

Adding `product_overview` and `goals` recovers GT-13 and GT-14. Tightening rule 4 (collapsing is PARTIAL) should recover GT-01.

**Expected: 14 / 14 → 1.000, hallucination rate still 0.0.**

That pairing is the claim worth making in the writeup: completeness rose 0.893 → 1.000 **while** hallucination stayed at 0.0. Completeness rising alone would be unremarkable — you can always extract more by extracting loosely. Rising with hallucination flat is a real improvement.

---

## Why 0–1, and logging it to Langfuse

Scores on 0–1 are the convention across eval tooling, and Langfuse's score API takes a plain numeric value — so a 0–1 completeness score drops straight into the same trace your run already produced. That lets you filter traces by score, chart completeness over time, and put a real number next to each test in your baseline log.

To attach a score to an existing trace, add a `score-create` event to the Langfuse batch:

```json
{
  "id": "<uuid>",
  "type": "score-create",
  "timestamp": "<iso>",
  "body": {
    "traceId": "<the traceId from that run>",
    "name": "extraction_completeness",
    "value": 0.893,
    "dataType": "NUMERIC",
    "comment": "12.5 / 14 ground truth items · 2 missing (overview, business goal) · 1 collapsed"
  }
}
```

Push `hallucination_rate` as a second score on the same trace. Two scores side by side on one trace is what makes the pairing argument visible rather than asserted — and it is a stronger observability screenshot than a trace tree alone.

Keep both on the same polarity so they read consistently: completeness **higher is better**, hallucination rate **lower is better**. Say which is which in the writeup — a reader seeing 0.893 and 0.0 should not have to guess.

---

## Worked example 2 — the combined multi-source run (GT-01…25)

The single-PRD workflow concatenates every Drive file into one input, so the judge builds ground truth across **all** the sources at once: the product brief, the five meeting transcripts, and the stakeholder notes. Reading them together yields roughly **25** distinct stated items — which is why this run's denominator is 25, not the 14 of the single-brief example above.

> **Reconstruction, not the judge's verbatim list.** The `Agent 5` judge node isn't traced, so its exact ground truth wasn't captured. The table below is reconstructed from the source files and is consistent with the observed score (0.54). To capture the judge's actual list in future, log its raw JSON to a field on the trace.

| ID | Ground-truth item | Source | Captured? |
|---|---|---|---|
| GT-01 | Display 5 named core metrics (revenue, active users, churn, NPS, tickets) | brief | Yes |
| GT-02 | Filter by date range (7 / 30 / 90-day / custom) | brief | Yes |
| GT-03 | Data refreshes every 15 minutes | brief | Yes |
| GT-04 | Role-based access (executives all, team leads own team) | brief | Yes |
| GT-05 | Export to PDF for board reports | brief | Yes |
| GT-06 | Integrate with existing PostgreSQL warehouse | brief | Yes (constraint) |
| GT-07 | No third-party analytics tools | brief | Yes (constraint) |
| GT-08 | Mobile accessible / responsive | brief | Partial |
| GT-09 | Target launch Q3 2026 | brief | Yes (constraint) |
| GT-10 | Filter by date, category, **and status** | Transcript 1 | Missing (only date captured) |
| GT-11 | Results load under 2 seconds | Transcript 1 | Missing |
| GT-12 | New dedicated page, match existing design system | Transcript 1 | Missing |
| GT-13 | Designs due end of April | Transcript 1 | Missing |
| GT-14 | Export as PDF and CSV | Transcript 4 | Yes |
| GT-15 | PDF must include company logo on every page | Transcript 4 | Missing (criterion lost) |
| GT-16 | CSV must preserve formulas (resolved to XLSX) | Transcript 4 | Missing (criterion lost) |
| GT-17 | Analytics service as its own microservice | notes | Yes |
| GT-18 | Use existing events table + timestamp index | notes | Yes |
| GT-19 | Multi-tenant / per-account data filtering | notes | Missing |
| GT-20 | Search & filtering prioritised over new metrics | notes | Missing |
| GT-21 | Persona: business analyst (exports to Excel today) | brief | Yes |
| GT-22 | Persona: team lead (weekly summaries) | brief | Yes |
| GT-23 | Persona: executive (high-level overview) | brief | Yes |
| GT-24 | Overview: self-serve analytics without SQL or the data team | brief | Yes |
| GT-25 | Business goal: #1 enterprise customer request | brief | Yes |

**~14–15 of 25 captured (some partial) → completeness ≈ 0.54**, hallucination 0.07, on the gpt-4o extractor run.

Two patterns are visible and worth citing in the writeup:

- **The misses cluster in acceptance-criteria detail and the stakeholder notes** (GT-10, 15, 16, 19, 20). The extractor captured the headline requirement — "export PDF and CSV" — but dropped the stated specifics beneath it (logo, preserve-formulas). This is the grouping behaviour: the extractor folds detail into one line while the judge counts each stated detail separately.
- **0.54 is a fair, not a broken, number.** A PRD that says "export to PDF and CSV" but drops "must include the company logo" has genuinely lost a requirement an engineer needs. The strict judge is measuring something real.

This is also the run that produced the project's central evaluate-and-iterate story: an earlier `gpt-4o-mini` extractor scored **0.32** here (4 of ~25 items, zero contradictions); moving to `gpt-4o` plus an exhaustiveness instruction lifted it to **0.54** and turned contradiction detection on — at the cost of hallucination rising from 0.00 to 0.07, a recall/precision tradeoff the paired metrics made visible.

---

## Using this in your submission

**For Q3 §7 (Evaluation strategy).** Cite completeness with its measurement method — LLM-as-judge against a hand-built ground truth, partial credit for lost detail — rather than just naming the metric. Method is what makes a metric credible.

**For Q3 §8 (Testing observations).** The 0.893 → 1.000 pairing above is a clean before/after, and it came from a real defect you found by reading output against source. Keep the pre-fix number.

**For Q4 (Reflection).** The instructive part is *how* the gap surfaced. The pre-fix PRD said "Not discussed in this meeting." for Product Overview, which was a true statement about the extraction and a wrong statement about the brief. No error appeared in any trace. It was only visible by reading the output against the source and asking why something was absent — which is the argument for expected-keyword checks in a baseline dataset rather than merely confirming the pipeline ran.

**One caution on judging.** Have the judge build ground truth *before* seeing the extraction. Shown both at once, an LLM judge tends to anchor on what the extraction contains and under-report what's missing — the same completion pressure that makes your PRD Generator want to fill empty sections. If you can, hand-check ground truth on two or three tests to calibrate the judge, and say in the writeup that you did.
