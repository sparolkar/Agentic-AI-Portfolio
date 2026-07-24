# Evaluation Prompt — Format Compliance

The third of PRD Genie's production metrics. It answers: **does the generated PRD follow the required template structure?** — the right sections, in the right order, with the tables their columns call for.

This is the only one of the three metrics that is purely *structural*. Completeness and hallucination require judgement about meaning; format compliance is a checklist against `prd_template.md`. Because it's mechanical, it should be scored **deterministically by a script**, not by an LLM. A model asked to check whether ten headings are present will occasionally miscount or hallucinate a heading that isn't there — the exact failure this project exists to catch, reintroduced in the grader. A script cannot.

There is an LLM version below for when you can't run code, but the script is the reference. A JavaScript port of this exact script runs live in the n8n workflow (node **Score Format Compliance**), verified to produce identical scores — so format compliance is pushed to Langfuse on every run alongside the other two metrics.

---

## What compliance means here

`prd_template.md` defines ten sections:

1. Product Overview 2. Goals and Objectives 3. User Personas 4. Feature Requirements (4.1 Functional table, 4.2 Non-Functional table) 5. Acceptance Criteria 6. Out of Scope 7. Dependencies (table) 8. Assumptions 9. Open Questions 10. Timeline (table)

**Crucial distinction:** compliance is about the section being *present and correctly formed*, not *filled*. A section reading `Not discussed in this meeting.` is fully compliant — the guardrail put it there deliberately, and penalising it would pit this metric against the anti-hallucination design. Format compliance asks "is the skeleton right?", not "is it complete?" That's what the other two metrics are for.

### The checklist (13 weighted checks)

| # | Check | Weight |
|---|---|---|
| C1 | All 10 top-level sections present | 3 |
| C2 | Sections in template order | 1 |
| C3 | §4 split into 4.1 Functional and 4.2 Non-Functional | 1 |
| C4 | §4.1 is a table with columns ID / Requirement / Priority / Source | 1 |
| C5 | §4.2 is a table with columns ID / Requirement / Category / Target | 1 |
| C6 | §7 Dependencies is a table (Dependency / Owner / Status / Risk) | 1 |
| C7 | §10 Timeline is a table (Milestone / Target Date) | 1 |
| C8 | Functional requirement IDs match `FR-\d+` | 1 |
| C9 | Priority values ∈ {Must, Should, Nice, Not stated} | 1 |
| C10 | Every §4.1 row has a non-empty Source cell | 1 |
| C11 | No leftover template placeholders (`[ ]`, `< >`, `FR-001` empty rows) | 1 |
| C12 | Empty sections use the exact abstention string, not blanks or invented filler | 1 |
| C13 | No sections outside the template (no invented headings) | 1 |

`compliance = points earned / 15`, reported 0.000–1.000.

C1 carries weight 3 because a missing section is the failure a downstream reader most immediately hits. C10 and C12 connect format to the anti-hallucination design: a populated Source column and a correct abstention string are formatting rules that *enforce* groundedness, so they earn their own checks.

---

## The scoring script

`scripts/score_format_compliance.py` — deterministic, run it over the generated PRD markdown.

```python

#!/usr/bin/env python3
"""Format-compliance score (0-1) for a generated PRD against prd_template.md.
Structural only: presence, order, tables, IDs. Does NOT judge whether sections
are filled — an abstention is compliant. Usage: python score_format_compliance.py PRD.md"""
import re, sys, json

ABSTAIN = "Not discussed in this meeting."
SECTIONS = ["Product Overview","Goals and Objectives","User Personas",
            "Feature Requirements","Acceptance Criteria","Out of Scope",
            "Dependencies","Assumptions","Open Questions","Timeline"]

def headings(md):
    return [re.sub(r"^#+\s*\d*\.?\s*","",h).strip()
            for h in re.findall(r"^#{1,3}\s+.*$", md, re.M)]

def has_cols(md, section, cols):
    # find the section body, then look for a table header row containing every column
    m = re.search(rf"{re.escape(section)}(.*?)(?=\n#{{1,2}}\s|\Z)", md, re.S)
    body = m.group(1) if m else ""
    for line in body.splitlines():
        if "|" in line and all(c.lower() in line.lower() for c in cols):
            return True
    return False

def score(md):
    h = headings(md)
    hl = " \n ".join(h).lower()
    checks = {}

    checks["C1_all_sections"]   = (3, sum(s.lower() in hl for s in SECTIONS) == 10)
    # order: sections appear in template sequence
    idx = [hl.find(s.lower()) for s in SECTIONS if s.lower() in hl]
    checks["C2_order"]          = (1, idx == sorted(idx) and len(idx) == 10)
    checks["C3_fr_split"]       = (1, "functional" in hl and "non-functional" in hl)
    checks["C4_fr_table"]       = (1, has_cols(md,"Feature Requirements",["Requirement","Priority","Source"]))
    checks["C5_nfr_table"]      = (1, has_cols(md,"Feature Requirements",["Requirement","Category","Target"]))
    checks["C6_dep_table"]      = (1, has_cols(md,"Dependencies",["Owner","Status","Risk"]))
    checks["C7_timeline_table"] = (1, has_cols(md,"Timeline",["Milestone","Target"]) or ABSTAIN in md)
    checks["C8_fr_ids"]         = (1, bool(re.search(r"\bFR-\d+\b", md)) or ABSTAIN in md)
    checks["C9_priority"]       = (1, not re.search(r"Priority[:\|]\s*(High|Low|Critical|P\d)", md, re.I))
    # source column populated on every FR row (rows that carry an FR- id)
    fr_rows = [l for l in md.splitlines() if re.search(r"\bFR-\d+\b", l)]
    checks["C10_source_filled"] = (1, all(len([c for c in r.split("|") if c.strip()])>=4 for r in fr_rows) if fr_rows else True)
    checks["C11_no_placeholder"]= (1, not re.search(r"\[\s*\]|<[^>]*>|FR-001\s*\|\s*\|", md))
    checks["C12_abstain_string"]= (1, "not discussed" not in hl)   # abstention lives in body, never as a heading
    ALLOWED = [s.lower() for s in SECTIONS] + ["functional", "product requirements document"]
    checks["C13_no_extra_secs"] = (1, all(any(a in x.lower() for a in ALLOWED) or len(x) < 3 for x in h))

    earned = sum(w for (w,ok) in checks.values() if ok)
    total  = sum(w for (w,_) in checks.values())
    return {
        "format_compliance": round(earned/total, 3),
        "points": earned, "max": total,
        "failed": [k for k,(w,ok) in checks.items() if not ok],
        "checks": {k: ok for k,(w,ok) in checks.items()},
    }

if __name__ == "__main__":
    md = open(sys.argv[1], encoding="utf-8").read()
    print(json.dumps(score(md), indent=2))
```

---

## Worked example — the `sample_product_brief.txt` PRD

The PRD generated from the brief, after the schema fix. Overview, Goals, Personas and the requirement tables are populated; Success Metrics, Acceptance Criteria, Assumptions and Timeline abstain because the brief stated nothing measurable or dated.

| Check | Result | Note |
|---|---|---|
| C1 all 10 sections | ✅ (3) | every heading present |
| C2 order | ✅ (1) | template sequence |
| C3 FR split | ✅ (1) | 4.1 and 4.2 both there |
| C4 FR table cols | ✅ (1) | ID / Requirement / Priority / Source |
| C5 NFR table cols | ✅ (1) | ID / Requirement / Category / Target |
| C6 Dependencies table | ✅ (1) | present (abstains, but structurally a section) |
| C7 Timeline table | ✅ (1) | abstains — allowed |
| C8 FR ids | ✅ (1) | FR-001 … FR-005 |
| C9 priority values | ✅ (1) | Must / Should / Not stated only |
| C10 Source filled | ✅ (1) | every FR row cites a REQ id |
| C11 no placeholders | ✅ (1) | template `FR-001 | |` rows all replaced |
| C12 abstain string | ✅ (1) | empty sections use the exact literal |
| **C13 no extra sections** | ❌ (0) | generator added a **"Success Criteria"** heading beside the template's "Goals and Objectives" — a duplicate not in the template |

**14 / 15 → format_compliance = 0.933**

> ### Suggested score: **0.933**
>
> One structural defect: the generator emitted an extra "Success Criteria" section that the template does not define, most likely bleeding in from the brief's own headings. Everything else — all ten sections, both requirement tables with correct columns, populated Source cells, exact abstention strings — conforms. The fix is a one-line addition to the PRD Generator prompt ("Use ONLY the ten template sections; do not add headings the template does not define"), after which this returns 1.000.

Contrast with the **unguarded** run from the hallucination eval: that PRD would also score high on *format* — it had all ten sections filled — because format compliance is blind to truth. A PRD can be perfectly formatted and full of fabrication. That's precisely why this metric is reported alongside the other two and never alone: it certifies the skeleton, not the content.

---

## LLM fallback (only if you can't run the script)

For scoring inside the n8n judge chain, where running Python isn't available:

```
ROLE: You are a format compliance checker. You verify a PRD's STRUCTURE against
a fixed template. You do not judge whether sections are filled, truthful, or
well written — only whether the required skeleton is present and correctly formed.

TEMPLATE SECTIONS (exact, in order):
1 Product Overview · 2 Goals and Objectives · 3 User Personas ·
4 Feature Requirements (4.1 Functional table: ID|Requirement|Priority|Source;
4.2 Non-Functional table: ID|Requirement|Category|Target) · 5 Acceptance Criteria ·
6 Out of Scope · 7 Dependencies (table) · 8 Assumptions · 9 Open Questions ·
10 Timeline (table)

CHECK, and mark each pass/fail:
  - all 10 sections present (weight 3)
  - sections in template order (1)
  - 4.1 and 4.2 present as tables with the specified columns (2)
  - Dependencies and Timeline are tables (2)
  - functional requirements use FR-<n> ids and every row has a Source (2)
  - priority values are Must/Should/Nice/Not stated only (1)
  - empty sections use EXACTLY "Not discussed in this meeting." — not blanks,
    not "TBD", not invented filler (1)
  - NO sections beyond the ten template sections (1)

RULES:
1) A section reading "Not discussed in this meeting." is COMPLIANT. Do not
   penalise an abstention — it is a correctly formed empty section.
2) You are blind to content truth. A fabricated but well-structured PRD scores
   high here; that is correct, because a different metric judges truth.
3) An extra section the template does not define is a FAILURE, even if useful.

format_compliance = points earned / 15, as a decimal 0.000-1.000.

OUTPUT JSON only:
{ "format_compliance": 0.0, "points": 0, "max": 15,
  "failed_checks": ["..."], "extra_sections": ["..."], "summary": "<one line>" }
```

The script and this prompt use the same 15-point scale, so their scores are directly comparable — useful for confirming the LLM version tracks the deterministic one before you trust it.

---

## Using this in your submission

**Q3 §7.** Note that format compliance is scored **deterministically**, unlike the other two. That's a deliberate choice worth one sentence: a structural check has a correct answer, so a script is both cheaper and more trustworthy than an LLM, and using a model to check for fabricated headings would risk the grader hallucinating. Reviewers reward knowing when *not* to use an LLM.

**Q3 §8.** The 0.933 → 1.000 fix (removing the stray "Success Criteria" heading) is another clean one-variable before/after for the observations section.

**The three metrics together.** Completeness (recall, in), hallucination (precision, out), format (structure). A run can score 1.0 on any one while failing the others — a fabricated PRD is well-formatted, an abstaining PRD is clean of hallucination, a thorough extraction can still be mis-structured. Reporting all three, on the same 0–1 scale and the same trace, is what makes the evaluation credible rather than cherry-picked.
