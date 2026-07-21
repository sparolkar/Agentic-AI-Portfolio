# Q1 — Ideation (15 points)

**PRD Genie · NeuronForge Technologies**
Shailendra Parolkar · Applied Agentic AI for PMs/TPMs

## Pain points in NeuronForge's PM workflow

### 1. Requirements are re-typed out of transcripts by hand

**Manual step today.** After each planning meeting, the PM re-reads the transcript and retypes requirements into a document, deciding as they go what counts as a requirement versus discussion. A 45-minute meeting takes 60–90 minutes to convert.

**Agent solution.** The Requirement Extractor reads the raw text and returns a structured object separating stated requirements from ambiguous discussion, with a verbatim quote attached to every item.

**Inputs → outputs.** Transcript, brief, or notes (`.txt`/`.md`) → JSON containing `stated_requirements`, `ambiguous_items`, `contradictions`, `personas`, `constraints`, `dependencies`.

**Risks.** The agent classifies undecided discussion as a settled requirement. Mitigated by the quote requirement — an item with no verbatim source is dropped in code before it can travel downstream — and by an explicit rule that "TBD", "let's discuss offline", and similar phrases route to `ambiguous_items`.

### 2. PRD format varies by author, so engineering can't estimate consistently

**Manual step today.** Each PM writes PRDs in their own structure. Engineering re-reads each one to locate acceptance criteria and NFRs before estimating, and the inconsistency itself creates rework.

**Agent solution.** The PRD Generator populates one fixed 10-section template for every input, so structure stops varying by author.

**Inputs → outputs.** Extracted requirements JSON → markdown PRD conforming to `prd_template.md`, with a Source column tracing every functional requirement to its requirement ID.

**Risks.** A fixed template invites the model to fill sections the meeting never covered — the single most likely failure in this system. Mitigated structurally: the generator receives only the extraction JSON and never the raw transcript, so it has no raw material to infer from, and every unsupported section must read "Not discussed in this meeting."

### 3. Breaking a PRD into epics and stories is manual and inconsistent

**Manual step today.** The PM decomposes features into epics and user stories by hand, applying priority informally. Two PMs decompose the same PRD differently.

**Agent solution.** The Story Breakdown agent converts the PRD's functional requirements into epics, features, and `As a [persona]` user stories with MoSCoW priority, each traced to its FR ID.

**Inputs → outputs.** PRD functional requirements and acceptance criteria → epics → features → user stories with priority and criteria.

**Risks.** Story inflation — five requirements becoming eleven stories — which reads as thoroughness rather than fabrication and therefore survives review. Mitigated by a one-story-per-requirement default, split only when the requirement explicitly names multiple actors, and by requiring an FR ID on every story.

### 4. Gaps in a discussion are noticed late, usually mid-sprint

**Manual step today.** Missing information surfaces when a developer is blocked. The PM then reconstructs who to ask and what exactly was left open, often weeks after the meeting.

**Agent solution.** The Gap Analyzer converts ambiguous items, contradictions, and untouched PRD topics into specific clarification questions, each addressed to a named role.

**Inputs → outputs.** Ambiguity, contradiction, and not-discussed arrays from extraction → blocking questions, clarifying questions, and unresolved conflicts.

**Risks.** The analyzer proposes answers rather than asking questions — a suggested answer becomes a requirement the moment a busy stakeholder says "sure." Mitigated by an explicit rule that it may only ask, never answer, and that every question must quote the gap it came from.

### 5. Thin or contradictory meetings produce confidently wrong documents

**Manual step today.** A PM under deadline pressure writes a plausible PRD from a thin meeting, filling gaps from experience. The assumptions are invisible to everyone downstream.

**Agent solution.** A sufficiency gate between extraction and generation. When zero stated requirements are extracted, the pipeline halts and returns clarification questions instead of a PRD.

**Inputs → outputs.** Extraction result → either the full pipeline, or a "no requirements extractable" response plus questions.

**Risks.** The gate is too strict and blocks thin-but-usable inputs. Mitigated by keying the gate on stated requirements only — an input with one solid requirement and much ambiguity still proceeds, with the ambiguity surfaced in Open Questions rather than suppressed.

## Why these five

The first three attack time cost directly and are the required core capabilities. The last two attack a different and more expensive failure: documents that look finished but aren't. In this domain a missing PRD costs an afternoon, while a confidently fabricated requirement costs a sprint — engineering builds to it before anyone notices nobody asked for it. That asymmetry is why the design spends its complexity budget on refusing to guess rather than on generating more.
