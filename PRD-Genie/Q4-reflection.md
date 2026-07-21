# Q4 — Reflection (5 points)

**Shailendra Parolkar** · PRD Genie

## 1. What the traces revealed

- **All agents actually good output.**
- **Token volume and cost were visible for all agents / models.** Hallucination is verbose. If T2 or T5 produced *more* output tokens than T1, that's a finding.
- **Traces revealed correctness, halucination, relevane for each agent run.**

## 2. Improvement plan

- Fine-tune the Requirement Extractor on stated-versus-ambiguous discrimination — it's the agent whose errors propagate furthest.
- Add held-out edge cases to the baseline dataset so future guardrails are validated on inputs they weren't written against.
- Add a human-in-the-loop confirmation of extracted requirements before PRD generation, which is the cheapest possible defence against the failure mode that matters most.

## 3. Risks of AI-generated PRDs

**Fabrication that reads as competence.** The dangerous output is not the obviously wrong one — it's the plausible one. A fabricated Success Metrics section looks like diligence, gets forwarded to stakeholders, and is built against. This is why the system's central design choice was withholding the transcript from the PRD Generator rather than instructing it to be careful.

**False confidence from format.** A completed 10-section template signals thoroughness regardless of whether the meeting supported it. Structure is a claim about completeness, and an automated system makes that claim on every run.

**Erosion of the PM's own judgment.** If reviewing a generated PRD becomes a formality, the system has relocated the work rather than removed it — and moved it somewhere with less scrutiny.

**Traceability decay over time.** Source IDs are only meaningful while someone checks them. A team that stops auditing the Source column has a system that looks grounded and isn't.

## 4. Connection to evaluation skills from the course

- What the baseline dataset caught that ad-hoc testing would have missed. The vague and contradictory inputs — T2, T3, T5, T9 — are where a system that looks fine on happy-path testing falls over, and you would not have written those inputs yourself.
- Why the pass condition inverts on thin inputs: a confident, complete answer to a vague input is a failure even when it reads well. This is the least intuitive idea in the course and the one most worth saying you absorbed.
- Why reporting hallucination rate alone is insufficient, and what pairing it with completeness protects against.
- What you would measure differently if this ran in production against real meetings rather than 12 curated inputs.
