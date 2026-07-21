# Q4 — Reflection (5 points)

**Shailendra Parolkar** · PRD Genie

> This one has to be yours. The marks here are for what *your* traces showed, and I don't have your run data. What follows is the structure the question asks for, with the specific prompts that turn your observations into answers. Replace every `【 】`.

## 1. What the traces revealed

The question asks what observability told you that you would not otherwise have known. Strong answers name something surprising.

Candidates worth checking in your Langfuse data before you write:

- **Which agent actually produced a bad output.** Open the PRD Generator's input span and search it for the suspect content. Absent there → Agent 2 fabricated. Present → Agent 1 fabricated and Agent 2 rendered it faithfully. Naming the agent is what separates an observation from a guess.
- **Token cost on vague versus detailed inputs.** Hallucination is verbose. If T2 or T5 produced *more* output tokens than T1, that's a finding.
- **`unsupported_items_dropped` across the 12 runs.** Non-zero means the extractor generated items without quotes and the code guardrail caught them — evidence the structural guardrail was doing work the prompt alone didn't.
- **The gap between "looks fine" and "is grounded."** Any output you'd have accepted on a read-through that the Source column showed was unsupported.

【Write 3–5 sentences on what you actually found, naming the agent, the test ID, and the number.】

## 2. Improvement plan

【What you would change next, in priority order, and why that order. Ground each item in a trace observation rather than general good practice.】

Reasonable candidates, if your data supports them:

- Fine-tune the Requirement Extractor on stated-versus-ambiguous discrimination — the playbook flags this as the natural candidate, and it's the agent whose errors propagate furthest.
- Add held-out edge cases to the baseline dataset so future guardrails are validated on inputs they weren't written against.
- Add a human-in-the-loop confirmation of extracted requirements before PRD generation, which is the cheapest possible defence against the failure mode that matters most.
- Replace estimated token counts with measured ones for a cost figure you can defend.

## 3. Risks of AI-generated PRDs

The strongest version of this section is specific to what you observed rather than generic AI-risk commentary.

**Fabrication that reads as competence.** The dangerous output is not the obviously wrong one — it's the plausible one. A fabricated Success Metrics section looks like diligence, gets forwarded to stakeholders, and is built against. This is why the system's central design choice was withholding the transcript from the PRD Generator rather than instructing it to be careful.

**False confidence from format.** A completed 10-section template signals thoroughness regardless of whether the meeting supported it. Structure is a claim about completeness, and an automated system makes that claim on every run.

**Erosion of the PM's own judgment.** If reviewing a generated PRD becomes a formality, the system has relocated the work rather than removed it — and moved it somewhere with less scrutiny.

**Traceability decay over time.** Source IDs are only meaningful while someone checks them. A team that stops auditing the Source column has a system that looks grounded and isn't.

【Add whichever of these your testing actually surfaced, plus anything you saw that isn't listed.】

## 4. Connection to evaluation skills from the course

【Your own reflection. The question is asking whether you internalised the evaluate → find failure → fix → re-evaluate → confirm loop, not whether you can name it.】

Worth addressing directly:

- What the baseline dataset caught that ad-hoc testing would have missed. The vague and contradictory inputs — T2, T3, T5, T9 — are where a system that looks fine on happy-path testing falls over, and you would not have written those inputs yourself.
- Why the pass condition inverts on thin inputs: a confident, complete answer to a vague input is a failure even when it reads well. This is the least intuitive idea in the course and the one most worth saying you absorbed.
- Why reporting hallucination rate alone is insufficient, and what pairing it with completeness protects against.
- What you would measure differently if this ran in production against real meetings rather than 12 curated inputs.

---

**One suggestion on tone.** This section rewards honesty about failure more than a clean narrative. "The extractor invented an acceptance criterion on T4 and I found it because the Source column had no matching quote" is worth more than "the system performed well across all tests." If your run was messy, say so and say what you learned — that's the answer the question is actually asking for.
