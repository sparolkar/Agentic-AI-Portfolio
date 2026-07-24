# Q3 — PRD Genie: Architecture Writeup (45 points)

**Shailendra Parolkar** · Applied Agentic AI for PMs/TPMs · NeuronForge Technologies

## 1. Problem summary

PMs and TPMs at NeuronForge lose hours translating meeting transcripts and stakeholder notes into structured PRDs, and because every PM formats them differently, engineering estimation slows down and requirements get lost in documents nobody revisits. PRD Genie ingests a transcript, brief, or set of notes and produces a template-conformant PRD, epics and user stories, and the clarification questions needed to close what the meeting left open. The dominant risk is fabrication: a PRD that reads authoritatively but contains a requirement nobody stated is worse than no PRD at all, because engineering will build to it before anyone notices.

That risk shaped every subsequent decision. Six of the twelve baseline tests grade the system on correctly refusing — flagging ambiguity, surfacing contradictions, declining to generate — and nine carry an explicit "must NOT" clause. Groundedness is not one concern among several here; it is the design.

## 2. Architecture

The chat message is the **primary input** and defines the PRD's scope. Files in a Google Drive folder are **supporting documents** — they may only add detail to a requirement the chat already raised, never introduce one of their own. With no requirements in the chat, the run halts: a PRD is never built from documents alone.

```
Chat request (PRIMARY — defines scope)          Google Drive folder (SUPPORTING)
   │                                                 │
   ▼                                                 │
Has Chat Requirements?  ── no ──► "describe your requirements" (no PRD)
   │ yes                                             │
   ▼                                                 ▼
   └──────────────►  Combine  ◄──── read + keep text files (detail only)
                        │   PRIMARY block + SUPPORTING blocks, labelled
                        ▼
   Agent 1 · Requirement Extractor          gpt-4o-mini, temp 0, JSON mode
        │  scopes to the PRIMARY block; enriches from SUPPORTING docs;
        │  records out-of-scope doc items in excluded_out_of_scope
        ▼
   Parse + Validate  ──────── Guardrail A: quote-or-drop, enforced in code
        │
        ├───────────────────────────────► Agent 4 · Gap Analyzer     gpt-4o-mini
        ▼                                       │   (parallel branch)
   Sufficiency Gate  ──────── Guardrail B: stated_count > 0 ?
        │                                       │
        ├── false ──► Halt: "no requirements extractable"  (Agents 2 and 3 never invoked)
        │                                       │
        └── true ───► Agent 2 · PRD Generator   gpt-4o   (extraction JSON only)
                           │                    │
                           ▼                    │
                      Agent 3 · Story Breakdown gpt-4o
                           │                    │
                           └──────► Merge ◄─────┘
                                      │
                                      ▼
                Judge + format score → Langfuse ingestion → Chat response
                (extraction_completeness · hallucination_rate · format_compliance)
```

**Why scope by the chat.** An earlier design concatenated every document with equal weight and extracted from the pile. On the combined corpus (brief + five transcripts + stakeholder notes) that produced a bloated, unfocused PRD: the extractor pulled in requirements from stakeholder notes that no one had actually requested, and the completeness judge — building ground truth across all documents — measured the system against ~25 items, many irrelevant to any single request. Making the chat the scope boundary fixes both: the PRD contains only what was asked for, enriched by whatever detail the documents add about it, and the evaluation is measured against the right target.

*Canvas screenshot: 【canvas screenshot】*

## 3. Tool selection

| Category | Choice | Why |
|---|---|---|
| Workflow platform | **n8n Cloud** | Used in class, so build effort went into prompt design rather than tooling. PRD Genie has no external API surface — text in, markdown out — so no platform held an integration advantage. What n8n contributed was the IF node, which renders the sufficiency gate as visible workflow structure rather than logic buried in a prompt. |
| LLM — extraction, gap analysis | **gpt-4o-mini** | Both are classification against a fixed schema. With the chat-primary scope, the extractor works on a focused request plus targeted enrichment, so the cheap model is sufficient and runs where input volume is highest. |
| LLM — PRD, stories | **gpt-4o** | These agents hold a 10-section template and a requirements set simultaneously while resisting the pull to fill gaps. Their output is the graded artifact. |
| Ingestion | Chat Trigger (primary) + Google Drive (supporting) | The chat message defines scope; a Drive folder supplies supporting documents that add detail within that scope. An optional `T1:` prefix tags runs so the baseline dataset is runnable from the same interface. |
| Observability | **Langfuse** (US region) | Set up in class. n8n has no native Langfuse node, so traces are posted to Langfuse's ingestion API from a Code + HTTP node pair — which also let me emit named spans for both guardrails. |
| Output | Chat response, markdown | Three sections in one reply. Keys live in the n8n credential store, never in the exported workflow JSON, which is committed publicly. |
| Auth | None for inputs | File/text input requires no OAuth. Stated explicitly rather than left blank. |

**Mixed-model rationale.** The extractor consumes the most input tokens but emits compact JSON; the generator consumes compact JSON but emits the long graded document. Putting the cheap model where volume is and the strong model where quality is graded saves ~63% against an all-gpt-4o pipeline (§6, measured) without touching the output that gets marked.

## 4. Orchestration pattern — sequential pipeline

Every input to PRD Genie is the same kind of object — one body of meeting text — and each passes through the same stages in the same fixed order: extract, generate, decompose. The stages are strictly data-dependent: the PRD Generator cannot run before requirements exist, and Story Breakdown cannot run before there is a feature list.

A **router/dispatcher** would spend a classification call deciding a route already known at design time, since transcripts, briefs, and notes all need identical treatment. A **hierarchical** planner would decompose a workflow whose decomposition is fixed before any input arrives. When the dependency graph is a straight line, the pattern that matches it is a pipeline.

The one deliberate deviation: the Gap Analyzer branches off the *extractor's* output in parallel with PRD generation rather than sitting downstream of it. Clarification questions derive from what the extraction found missing, not from what the PRD contains — which is also why a halted run still returns useful questions.

Ahead of the pipeline sits a deterministic **scoping front end** — a chat-requirements gate and a combine step that labels the chat as primary and the Drive files as supporting. It is not a fourth orchestration pattern; it is input preparation that fixes the pipeline's scope before extraction begins. Keeping it deterministic (an IF node and a Code node, no LLM) means the scope boundary is auditable rather than a model's judgement call.

## 5. Agents

| Agent | Responsibility | Input | Output | Model | Key prompt rule (guardrail) |
|---|---|---|---|---|---|
| 1 · Requirement Extractor | Scope to the chat, enrich from supporting docs, classify | PRIMARY chat block + SUPPORTING doc blocks | JSON: stated / ambiguous / contradictions / personas / constraints / dependencies / excluded_out_of_scope / not_discussed | gpt-4o-mini | A requirement may enter only if the chat raises it; supporting docs add detail but never scope. Every item carries a verbatim quote. Discussion without decision routes to `ambiguous`, never `stated`. Numbers and versions preserved exactly. |
| 2 · PRD Generator | Populate the 10-section template | Extraction JSON **only** | Markdown PRD, Source column populated | gpt-4o | Sections without supporting entries read exactly "Not discussed in this meeting." No placeholders, no "TBD — recommend...". Success Metrics named as the highest-risk section. |
| 3 · Story Breakdown | Convert features to epics and stories | PRD FRs + acceptance criteria | Epics → Features → `As a...` stories, MoSCoW | gpt-4o | One story per requirement unless the requirement names multiple actors. Every story traces to an FR ID. Priority absent → "Not stated". |
| 4 · Gap Analyzer | Turn gaps into stakeholder questions | Ambiguity, contradictions, not-discussed | Blocking / clarifying questions, unresolved conflicts | gpt-4o-mini | Ask, never answer. A suggested answer becomes a requirement the moment a busy stakeholder agrees to it. |

### Two structural guardrails

Prompt rules alone do not survive a template that demands ten sections, so two guardrails sit outside the prompts:

**Guardrail A — quote-or-drop, enforced in code.** The Parse + Validate node deletes any extracted requirement or persona lacking a verbatim quote before the payload moves on, recording the count in `unsupported_items_dropped`. A guardrail the model cannot talk its way past is stronger than one it merely agreed to. The count doubles as a hallucination signal requiring no hand-grading.

**Guardrail B — sufficiency gate.** An IF node halts the pipeline when zero stated requirements are extracted, returning a "no PRD generated" response plus clarification questions. T9 therefore passes by construction: the PRD Generator is not asked to decline, it is never invoked.

**Guardrail C — scope boundary.** The chat request defines scope; supporting documents may only enrich requirements the chat already raised. Any document item the chat never raised is routed to `excluded_out_of_scope` and kept out of the PRD, and a chat with no requirements halts before any document is read. This makes the anti-fabrication design explicit at the input boundary: the PRD cannot contain a requirement the user did not ask for, even when a document mentions one.

**Traceability by construction.** The template's Source column carries the requirement ID for every functional requirement, and every ID resolves to a verbatim quote. Hallucination becomes inspectable — any row with a missing or unresolvable Source is unsupported — which is how the hallucination metric is computed without subjective grading.

## 6. Cost analysis

**Measured from Langfuse traces across the 12-test baseline run** (not estimated), against list pricing of $0.15/$0.60 per 1M tokens for gpt-4o-mini and $2.50/$10.00 per 1M for gpt-4o. Averages are per agent call:

| Agent | Model | Avg tokens (in / out) | Avg cost / run |
|---|---|---|---|
| Requirement Extractor | gpt-4o-mini | 2,135 / 617 | $0.00069 |
| PRD Generator | gpt-4o | 656 / 446 | $0.00610 |
| Story Breakdown | gpt-4o | 446 / 175 | $0.00286 |
| Gap Analyzer | gpt-4o-mini | 189 / 590 | $0.00038 |
| **Completed run** | | | **$0.01005** (range $0.0056–$0.0206) |

| Item | Value |
|---|---|
| Runs per PM per day | 4 (roughly one per planning meeting) |
| **Cost per PM per day** | **$0.040** ($0.01005 × 4) |
| **Cost per PM per month** | **$1.21** (30 days) · **$0.88** (22 working days) |
| Halted run (T2, T9) | **$0.00102** — ~10× cheaper than a completed run |
| Ingestion / auth APIs | $0 — text and Drive read, no paid API calls |
| Observability | $0 — Langfuse free tier covers evaluation volume |
| **Total per PM per month** | **≈ $1.21** |

The evaluation judge (Agent 5, gpt-4o) adds roughly $0.03–0.04 per run but is an *evaluation* cost, not a production one — it does not ship, so it is excluded from the per-PM figures above.

Three observations worth more than the headline number:

**The two gpt-4o generators are 89% of run cost.** The extractor reads the most tokens yet costs under 7% of the run, because it runs on gpt-4o-mini and emits compact JSON. Cost is driven by the *quality-graded* output (PRD + stories), not by input volume.

**The mixed-model choice saves ~63%.** An all-gpt-4o pipeline would cost about $0.027/run; the measured mixed-model run is $0.01005. Putting the cheap model on extraction and gap analysis — where the output is structured, not customer-facing — and the strong model only on the two graded artifacts is the whole saving.

**A halted run costs $0.00102 — 10× less than a completed one.** The sufficiency gate declines to spend gpt-4o tokens on inputs (T2, T9) where generation would produce nothing trustworthy. A guardrail that improves both correctness and unit economics is the strongest kind.

## 7. Evaluation strategy

Three metrics, all scored 0–1 and pushed to the same Langfuse trace on every run:

| Metric | What it measures | How scored | Baseline mean |
|---|---|---|---|
| **Extraction completeness** | fraction of the chat request's requirements captured | LLM judge (gpt-4o), ground truth built from the PRIMARY chat request only | **0.948** |
| **Hallucination rate** | fraction of extracted items unsupported by any source | LLM judge, grounded against the full source (chat + supporting docs) | **0.068** |
| **Format compliance** | template structure conformance | deterministic script (JS port in-workflow), 13 weighted checks | **0.927** |

Completeness and hallucination are reported **together, always**. Guardrails suppress output, so the cheapest way to drive hallucination to zero is an agent that refuses everything. Hallucination falling while completeness holds flat is a real improvement; both falling is a regression wearing a success costume. Format compliance is scored deterministically rather than by an LLM — a structural check has a correct answer, and using a model to look for fabricated headings would risk the grader hallucinating one.

In production I would add a correct-refusal rate on deliberately thin inputs, and a PM-facing acceptance rate — what fraction of generated PRDs are approved without substantive edits — as the honest measure of whether the system saves time rather than relocating it.

## 8. Testing observations

All 12 baseline inputs were run and traced. Every test passed its expected-output criteria (12/12 in `baseline-test-log.xlsx`), and every run's three metric scores were captured in Langfuse.

| Test | Completeness | Hallucination | Format | Behaviour |
|---|---|---|---|---|
| T1 | 1.000 | 0.143 | 0.867 | generated |
| T2 | 1.000 | 0.000 | — | halted (no requirement stated) ✓ |
| T3 | 1.000 | 0.000 | 0.933 | both sides of the refresh/API-load conflict surfaced, not resolved |
| T4 | 1.000 | 0.000 | 0.867 | generated |
| T5 | 1.000 | 0.667 | 0.933 | thin input; small denominator (few extracted items) |
| T6 | 1.000 | 0.000 | 0.933 | three viewpoints captured |
| T7 | 1.000 | 0.000 | 1.000 | "10,000 concurrent users", "< 200ms at p95", "Salesforce REST API v52" preserved exactly |
| T8 | 1.000 | 0.000 | 0.933 | three personas, three stories |
| T9 | 1.000 | 0.000 | — | halted at sufficiency gate ✓ |
| T10 | 1.000 | 0.000 | 0.933 | SSO + dependency captured |
| T11 | 0.750 | 0.000 | 1.000 | full-PRD generation; missed only "PM: Sarah" |
| T12 | 0.625 | 0.000 | 0.867 | story breakdown; missed only "PM: Sarah" |
| **Mean** | **0.948** | **0.068** | **0.927** | |

**What worked.** Completeness is near-perfect — ten of twelve tests at 1.0, the other two (T11, T12) at 0.75 and 0.625 missing a single item each. Hallucination is near-zero: ten of twelve tests at 0.000, mean 0.068. Correct refusal held on both thin inputs — T2 and T9 halted with no PRD despite four supporting documents being available, the exact over-extraction failure the scope design was built to prevent. Exact-value preservation (T7) and contradiction surfacing without resolution (T3) both behaved.

**The evaluate-and-iterate loop — the central result.** This project's strongest evidence is two consecutive fixes driven by traces, each verified by re-measurement.

*Fix 1 — a metric measuring the wrong scope.* An early trace showed extraction completeness at **0.033 on T10** (the SSO request), which looked catastrophic. The trace revealed the cause was not the extractor but the *metric*: the completeness judge built ground truth from the entire combined corpus — brief, five transcripts, notes — while the extractor correctly scoped to the one-line chat request, so the judge counted every unrelated dashboard requirement as "missed." The fix was architectural: make the chat the explicit scope boundary (Guardrail C), and re-scope the judge to build ground truth from the primary request alone while grounding hallucination against the full source. After the fix, T10 rose from **0.033 to 1.000**.

*Fix 2 — an evaluator that wasn't itself observable.* A subsequent run showed hallucination at 0.700 on T11 and T12, which was hard to credit because those outputs plainly covered their input and nothing more. The reason it could not be diagnosed was that the judge — the one agent whose reasoning was not logged — reported a bare score with no verdicts. Adding the judge's flagged items to the trace metadata resolved it in one look: the next run showed `hallucinated_items: []` on both, judge summary *"all extracted items were supported."* The 0.700 had been a judge artifact on a three-item denominator, not real fabrication. The same metadata pinpointed the remaining completeness gap to a single missed item — `GT-03: "PM: Sarah"` — the extractor dropping the PM assignment. This is the observability principle applied to the evaluator itself, not just the pipeline.

**The remaining weakness, stated plainly.** Two things are genuinely imperfect. The extractor drops the "PM: Sarah" assignment on T11/T12 (completeness 0.75 / 0.625) — a real, specific, minor miss worth a prompt tweak. And T5, a deliberately thin input, shows hallucination 0.667 on a very small denominator, where a single flagged item dominates the ratio; a fuller evaluation would run it several times to separate signal from small-sample noise.

**Measurement honesty.** These are single-run scores. The extractor and the LLM judge are both stochastic, so individual numbers — especially on small denominators like T5, T11, T12 — carry run-to-run variance; the means are the trustworthy figures, and a production evaluation would run each test several times and report a range. Scores from before the judge fix measured against a scope the system had deliberately abandoned and are not comparable to these.

**Observability.** Langfuse traces every run under a per-test name (`PRD Genie — T1` … `T12`), with named spans for both guardrails — the sufficiency-gate span reads HALTED or PASSED explicitly, so a T9 trace shows the guardrail working rather than leaving it implied by absent generations. All three metric scores attach to each trace, so completeness, hallucination, and format are read together per test. Trace export: `Langfuse Trace.csv`.
