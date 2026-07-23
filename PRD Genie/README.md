# PRD Genie — AI-Powered Product Documentation Assistant

Capstone · Applied Agentic AI for PMs/TPMs · **Shailendra Parolkar**

PRD Genie turns a product request into a structured PRD, epics and user stories, and the clarification questions the request left open — and refuses to generate a PRD when the request doesn't support one. Built in n8n Cloud with a four-agent sequential pipeline, scored by three metrics, and traced in Langfuse.

## The problem

PMs at NeuronForge spend 60–90 minutes converting each meeting into a PRD, every author formats differently, and requirements get lost between the conversation and the document. The risk that shaped the build: a PRD containing a requirement nobody asked for is worse than no PRD, because engineering builds to it before anyone notices.

## Architecture

The **chat request is the primary input** and defines the PRD's scope. A **Google Drive folder** supplies **supporting documents** that may only add detail to a requirement the request already raised — never introduce one of their own. With no requirement in the chat, the run halts.

```
Chat request (PRIMARY)                     Google Drive (SUPPORTING)
   ▼                                            │
Has Chat Requirements? ── no ──► "describe your requirements" (no PRD)
   │ yes                                        ▼
   └────────► Combine (PRIMARY + SUPPORTING, labelled)
                 ▼
   Agent 1 · Requirement Extractor   gpt-4o-mini · scopes to chat, enriches from docs
                 ▼
   Parse + Validate  ── Guardrail A: quote-or-drop (in code)
                 ├──────────────► Agent 4 · Gap Analyzer  gpt-4o-mini (parallel)
                 ▼
   Sufficiency Gate  ── Guardrail B: halt if no requirements
                 ├── false ──► Halt ─┐
                 └── true ──► Agent 2 · PRD Generator  gpt-4o
                                 ▼
                            Agent 3 · Story Breakdown  gpt-4o
                                 ▼
             Merge → Judge + format score → Langfuse → Chat response
```

**Orchestration pattern: sequential pipeline** — every input passes through the same fixed, data-dependent stages. Full justification in [Q3-architecture-writeup.md](Q3-architecture-writeup.md).

### Three structural guardrails

- **A — quote-or-drop:** every extracted requirement must carry a verbatim quote; a code node deletes any that don't, before generation.
- **B — sufficiency gate:** zero requirements → the pipeline halts and returns questions. T9 ("Meeting happened. Notes: none.") passes by construction.
- **C — scope boundary:** the chat defines scope; supporting documents enrich only. Anything a document raises that the chat didn't is routed to `excluded_out_of_scope` and left out.

## Evaluation

Three metrics, scored 0–1, pushed to every Langfuse trace. Baseline means across the 12-test dataset:

| Metric | Mean | Method |
|---|---|---|
| Extraction completeness | **0.948** | LLM judge, ground truth from the chat request only |
| Hallucination rate | **0.068** | LLM judge, grounded against the full source |
| Format compliance | **0.927** | deterministic script (JS port runs in-workflow) |

The evaluation itself was debugged twice from traces — see Q3 §8 and [Q4-reflection.md](Q4-reflection.md).

## Cost

**$0.010 per completed run · ~$1.21 / PM / month** at 4 runs/day (measured from traces). A halted run costs $0.001 (~10× less). Mixed-model design (mini extraction, gpt-4o generation) saves ~63% against all-gpt-4o.

## Repository contents

| File | What it is |
|---|---|
| `PRD Genie n8n Worrkflow.json` | **The workflow** — import into n8n |
| `PRD Genie n8n Workflow.png` | canvas screenshot |
| `Q1-ideation.md` · `Q2-program-charter.md` · `Q3-architecture-writeup.md` · `Q4-reflection.md` | assignment answers |
| `agent-prompts.md` | all four agent system prompts + guardrail reasoning |
| `baseline-test-log.xlsx` | baseline dataset results, T1–T12 |
| `eval-extraction-completeness.md` · `eval-hallucination-rate.md` · `eval-format-compliance.md` | evaluation methodology + GT-01…25 and C1–C13 |
| `scripts/score_format_compliance.py` | deterministic format-compliance scorer |
| `Langfuse Trace.csv` | trace export from the baseline run |
| `Resources/` | inputs (transcripts, brief, notes) and config (template, eval inputs) |

## Setup

1. **Import** `PRD Genie n8n Worrkflow.json` into n8n.
2. **OpenAI credential** on the four model nodes (needs `gpt-4o-mini` and `gpt-4o`).
3. **Google Drive credential** on the two Drive nodes; set your supporting-docs folder ID.
4. **Langfuse Basic Auth** on `Send to Langfuse` (public key = user, secret = password; US region URL).
5. **Open the chat** and describe your requirements. Prefix with `T1:` … `T12:` to tag the run in traces.

> Credentials live in n8n's store and are **not** in the exported JSON. This repository is public — no keys are committed.

## Tech stack

n8n Cloud · OpenAI gpt-4o and gpt-4o-mini · Langfuse (ingestion API) · Google Drive · Markdown output
