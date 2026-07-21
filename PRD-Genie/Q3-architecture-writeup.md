# Q3 — PRD Genie: Architecture Writeup (45 points)

**Shailendra Parolkar** · Applied Agentic AI for PMs/TPMs · NeuronForge Technologies

> **Fill-in markers.** Anything in `【 】` needs a number or screenshot from your own runs. Everything else is complete.

## 1. Problem summary

PMs and TPMs at NeuronForge lose hours translating meeting transcripts and stakeholder notes into structured PRDs, and because every PM formats them differently, engineering estimation slows down and requirements get lost in documents nobody revisits. PRD Genie ingests a transcript, brief, or set of notes and produces a template-conformant PRD, epics and user stories, and the clarification questions needed to close what the meeting left open. The dominant risk is fabrication: a PRD that reads authoritatively but contains a requirement nobody stated is worse than no PRD at all, because engineering will build to it before anyone notices.

That risk shaped every subsequent decision. Six of the twelve baseline tests grade the system on correctly refusing — flagging ambiguity, surfacing contradictions, declining to generate — and nine carry an explicit "must NOT" clause. Groundedness is not one concern among several here; it is the design.

## 2. Architecture

```
Chat input (transcript / brief / notes)
   │
   ▼
Agent 1 · Requirement Extractor            gpt-4o-mini, temp 0, JSON mode
   │  stated / ambiguous / contradictions / personas / dependencies
   │  every item carries a verbatim quote
   ▼
Parse + Validate  ──────── Guardrail A: quote-or-drop, enforced in code
   │
   ├─────────────────────────────────► Agent 4 · Gap Analyzer      gpt-4o-mini
   │                                        │   (parallel branch)
   ▼                                        │
Sufficiency Gate  ──────── Guardrail B: stated_count > 0 ?
   │                                        │
   ├── false ──► Halt: "no requirements extractable"  (Agents 2 and 3 never invoked)
   │                                        │
   └── true ───► Agent 2 · PRD Generator    gpt-4o   (receives extraction JSON only)
                      │                     │
                      ▼                     │
                 Agent 3 · Story Breakdown  gpt-4o
                      │                     │
                      └──────► Merge ◄──────┘
                                 │
                                 ▼
                    Langfuse ingestion  ──►  Chat response
                    (traces + guardrail spans)   (PRD · Stories · Questions)
```

*Rendered diagram: `01-design-steps-1-to-3.md` (Mermaid). Canvas screenshot: 【canvas screenshot】*

## 3. Tool selection

| Category | Choice | Why |
|---|---|---|
| Workflow platform | **n8n Cloud** | Used in class, so build effort went into prompt design rather than tooling. PRD Genie has no external API surface — text in, markdown out — so no platform held an integration advantage. What n8n contributed was the IF node, which renders the sufficiency gate as visible workflow structure rather than logic buried in a prompt. |
| LLM — extraction, gap analysis | **gpt-4o-mini** | Both are classification and span-finding against a fixed schema. The extractor reads every token of every transcript, so the cheap model runs where input volume is highest. |
| LLM — PRD, stories | **gpt-4o** | These agents hold a 10-section template and a requirements set simultaneously while resisting the pull to fill gaps. Their output is the graded artifact. |
| Ingestion | Chat Trigger + Code node | Text input; no parsing complexity. An optional `T1:` prefix tags runs so the baseline dataset is runnable from the same interface. |
| Observability | **Langfuse** (US region) | Set up in class. n8n has no native Langfuse node, so traces are posted to Langfuse's ingestion API from a Code + HTTP node pair — which also let me emit named spans for both guardrails. |
| Output | Chat response, markdown | Three sections in one reply. Keys live in the n8n credential store, never in the exported workflow JSON, which is committed publicly. |
| Auth | None for inputs | File/text input requires no OAuth. Stated explicitly rather than left blank. |

**Mixed-model rationale.** The extractor consumes the most input tokens but emits compact JSON; the generator consumes compact JSON but emits the long graded document. Putting the cheap model where volume is and the strong model where quality is graded saves 33% against an all-gpt-4o pipeline (§6) without touching the output that gets marked.

## 4. Orchestration pattern — sequential pipeline

Every input to PRD Genie is the same kind of object — one body of meeting text — and each passes through the same stages in the same fixed order: extract, generate, decompose. The stages are strictly data-dependent: the PRD Generator cannot run before requirements exist, and Story Breakdown cannot run before there is a feature list.

A **router/dispatcher** would spend a classification call deciding a route already known at design time, since transcripts, briefs, and notes all need identical treatment. A **hierarchical** planner would decompose a workflow whose decomposition is fixed before any input arrives. When the dependency graph is a straight line, the pattern that matches it is a pipeline.

The one deliberate deviation: the Gap Analyzer branches off the *extractor's* output in parallel with PRD generation rather than sitting downstream of it. Clarification questions derive from what the extraction found missing, not from what the PRD contains — which is also why a halted run still returns useful questions.

## 5. Agents

| Agent | Responsibility | Input | Output | Model | Key prompt rule (guardrail) |
|---|---|---|---|---|---|
| 1 · Requirement Extractor | Classify requirement-bearing statements | Raw text | JSON: stated / ambiguous / contradictions / personas / constraints / dependencies / not_discussed | gpt-4o-mini | Every item must carry a verbatim quote; no quote, no item. Discussion without decision routes to `ambiguous`, never `stated`. Numbers and versions preserved exactly. |
| 2 · PRD Generator | Populate the 10-section template | Extraction JSON **only** | Markdown PRD, Source column populated | gpt-4o | Sections without supporting entries read exactly "Not discussed in this meeting." No placeholders, no "TBD — recommend...". Success Metrics named as the highest-risk section. |
| 3 · Story Breakdown | Convert features to epics and stories | PRD FRs + acceptance criteria | Epics → Features → `As a...` stories, MoSCoW | gpt-4o | One story per requirement unless the requirement names multiple actors. Every story traces to an FR ID. Priority absent → "Not stated". |
| 4 · Gap Analyzer | Turn gaps into stakeholder questions | Ambiguity, contradictions, not-discussed | Blocking / clarifying questions, unresolved conflicts | gpt-4o-mini | Ask, never answer. A suggested answer becomes a requirement the moment a busy stakeholder agrees to it. |

### Two structural guardrails

Prompt rules alone do not survive a template that demands ten sections, so two guardrails sit outside the prompts:

**Guardrail A — quote-or-drop, enforced in code.** The Parse + Validate node deletes any extracted requirement or persona lacking a verbatim quote before the payload moves on, recording the count in `unsupported_items_dropped`. A guardrail the model cannot talk its way past is stronger than one it merely agreed to. The count doubles as a hallucination signal requiring no hand-grading.

**Guardrail B — sufficiency gate.** An IF node halts the pipeline when zero stated requirements are extracted, returning a "no PRD generated" response plus clarification questions. T9 therefore passes by construction: the PRD Generator is not asked to decline, it is never invoked.

**Traceability by construction.** The template's Source column carries the requirement ID for every functional requirement, and every ID resolves to a verbatim quote. Hallucination becomes inspectable — any row with a missing or unresolvable Source is unsupported — which is how the hallucination metric is computed without subjective grading.

## 6. Cost analysis

Per completed run (PRD + stories + questions), measured against list pricing of $0.15/$0.60 per 1M tokens for gpt-4o-mini and $2.50/$10.00 for gpt-4o:

| Agent | Model | Tokens (in / out) | Cost |
|---|---|---|---|
| Requirement Extractor | gpt-4o-mini | 1,150 / 620 | $0.00054 |
| PRD Generator | gpt-4o | 1,400 / 1,250 | $0.01600 |
| Story Breakdown | gpt-4o | 1,500 / 900 | $0.01275 |
| Gap Analyzer | gpt-4o-mini | 700 / 500 | $0.00041 |
| **Total per run** | | **4,750 / 3,270 = 8,020** | **$0.0297** |

| Item | Estimate |
|---|---|
| Runs per PM per day | 4 (roughly one per planning meeting) |
| **Cost per PM per day** | **$0.119** ($0.0297 × 4) |
| **Cost per PM per month** | **$3.56** (30 days) · **$2.61** (22 working days) |
| Ingestion / auth APIs | $0 — text input, no external API calls |
| Observability | $0 — Langfuse free tier covers evaluation volume |
| **Total per PM per month** | **≈ $3.56** |

Three observations worth more than the headline number:

**Output tokens on gpt-4o are 72% of run cost.** Verbosity drives spend, not context size — which is why trimming the guardrail language to save tokens would be a false economy.

**The mixed-model choice saves 33%.** An all-gpt-4o pipeline costs $0.0446/run ($5.35/PM/month). The saving is real, but the better argument is shape: the extractor handles 100% of input volume at 1.8% of run cost.

**A halted run costs $0.0005 — 58× less than a completed one.** The sufficiency gate declines to spend gpt-4o tokens on inputs where generation would produce nothing trustworthy. A guardrail that improves both correctness and unit economics is the strongest kind.

*Replace these with measured token counts from your traces if `tokenUsage` is exposed: 【measured cost per run】*

## 7. Evaluation strategy

| Metric | What it measures | How |
|---|---|---|
| **Extraction completeness** | % of stated requirements captured | Baseline dataset outputs vs. expected-output criteria |
| **Hallucination rate** | % of PRD items not traceable to source | Source column audit + `unsupported_items_dropped` from traces |
| **Format compliance** | % of PRDs with all 10 sections, including "Not discussed" | Automated section check |

Completeness and hallucination are reported **together, always**. Guardrails suppress output, so the cheapest way to drive hallucination to zero is an agent that refuses everything. Hallucination falling while completeness holds flat is a real improvement; both falling is a regression wearing a success costume.

In production I would add a correct-refusal rate on deliberately thin inputs, and a PM-facing acceptance rate — what fraction of generated PRDs are approved without substantive edits — as the honest measure of whether the system saves time rather than relocating it.

## 8. Testing observations

All 12 baseline inputs were run through the pipeline and documented in `03-baseline-test-log.xlsx`. Results: 【X passed / 12】.

**What worked.** 【e.g. T7 preserved "10,000 concurrent users", "< 200ms at p95" and "Salesforce REST API v52" exactly; T9 halted at the sufficiency gate with no PRD produced; T3 recorded both sides of the refresh-versus-API-load conflict without resolving it.】

**Where it hallucinated.** 【The specific failure: which agent, which test input, what was invented, and the trace that shows it. Keep the original ugly output — do not clean it up.】

**What I fixed.** 【Which prompt changed, what rule was added, and the before/after on the same input. Change one variable at a time so the improvement is attributable.】

**How I know the fix holds.** Re-running the input the guardrail was written against is the weakest evidence available — passing it was close to guaranteed. 【State what else you checked: fresh edge-case inputs the guardrail had not seen, a full 12-test re-run confirming no regression, and an extraction-completeness figure showing the agent did not simply start refusing. If you only have the original re-run, say so and name it as a limitation.】

**Still open.** 【Known limitations. Naming them reads as understanding, not weakness.】

**Observability.** Langfuse traces every run, including named spans for both guardrails — the sufficiency gate span reads HALTED or PASSED explicitly, so a T9 trace shows the guardrail working rather than merely implied by absent generations. Screenshots: 【trace screenshots】.
