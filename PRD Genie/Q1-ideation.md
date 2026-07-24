# Q1 — Ideation (15 points)

**PRD Genie · NeuronForge Technologies** — Shailendra Parolkar

## Pain points in NeuronForge's PM workflow

### 1. Requirements are re-typed out of meetings by hand

**Manual step.** After each planning meeting a PM re-reads the transcript and retypes requirements into a document, deciding as they go what is a requirement versus discussion. A 45-minute meeting takes 60–90 minutes to convert.

**Agent solution.** The Requirement Extractor reads the input and returns a structured object separating stated requirements from ambiguous discussion, each item carrying a verbatim quote.

**Inputs → outputs.** Chat request (primary) + optional supporting documents → JSON of `stated_requirements`, `ambiguous_items`, `contradictions`, `personas`, `constraints`, `dependencies`.

**Risks.** Classifying undecided discussion as a settled requirement. Mitigated by the quote-or-drop rule (no verbatim quote → item dropped in code) and an explicit rule routing "TBD" / "let's discuss offline" to `ambiguous_items`.

### 2. PRD format varies by author, so engineering cannot estimate consistently

**Manual step.** Every PM writes PRDs differently; engineering re-reads each to locate acceptance criteria and NFRs before estimating.

**Agent solution.** The PRD Generator populates one fixed 10-section template for every input.

**Inputs → outputs.** Extraction JSON → markdown PRD with a Source column tracing each functional requirement to its requirement ID.

**Risks.** A fixed template invites the model to fill sections the meeting never covered. Mitigated structurally: the generator receives only the extraction JSON, never the raw source, and every unsupported section reads "Not discussed in this meeting."

### 3. Breaking a PRD into epics and stories is manual and inconsistent

**Manual step.** The PM decomposes features into epics and stories by hand, applying priority informally.

**Agent solution.** The Story Breakdown agent converts the PRD's functional requirements into epics, features, and `As a [persona]` stories with MoSCoW priority, each traced to an FR ID.

**Inputs → outputs.** PRD functional requirements → epics → features → user stories.

**Risks.** Story inflation — five requirements becoming eleven stories, which reads as thoroughness. Mitigated by a one-story-per-requirement default, split only when multiple actors are named.

### 4. Gaps in a discussion surface late, usually mid-sprint

**Manual step.** Missing information appears when a developer is blocked, weeks after the meeting.

**Agent solution.** The Gap Analyzer turns ambiguous items, contradictions, and untouched topics into specific clarification questions addressed to named roles.

**Inputs → outputs.** Extraction ambiguity/contradiction arrays → blocking questions, clarifying questions, unresolved conflicts.

**Risks.** The analyzer proposes answers rather than asking. Mitigated by an explicit ask-never-answer rule.

### 5. Thin or off-topic input produces confidently wrong documents

**Manual step.** A PM under deadline pressure writes a plausible PRD from a thin meeting, filling gaps from experience.

**Agent solution.** A scope boundary plus a sufficiency gate. The chat request defines scope; supporting documents can only enrich it. With no requirements in the chat, the pipeline halts and returns clarification questions instead of a PRD.

**Inputs → outputs.** Chat request → either the full pipeline, or a "no requirements extractable" response plus questions.

**Risks.** The gate being too strict and blocking usable input. Mitigated by keying the gate on stated requirements only — one solid requirement plus ambiguity still proceeds, with the ambiguity surfaced in Open Questions.

## Why these five

The first three attack time cost directly and are the required core capabilities. The last two attack a more expensive failure: documents that look finished but aren't. In this domain a missing PRD costs an afternoon; a confidently fabricated requirement costs a sprint, because engineering builds to it before anyone notices nobody asked for it. That asymmetry is why the design spends its complexity budget on refusing to guess rather than on generating more — and the measured hallucination rate of 0.068 across the baseline suite reflects that.
