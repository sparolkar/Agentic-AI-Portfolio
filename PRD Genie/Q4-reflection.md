# Q4 — Reflection (5 points)

**Shailendra Parolkar · PRD Genie**

> Drafted from the actual trace findings. Personalise the voice before submitting; the facts are all from the run data.

## 1. What the traces revealed

Observability did not just confirm the system worked — it twice showed me that something I *believed* was wrong, and once that my *measurement* was wrong.

- **A metric measuring the wrong thing.** Extraction completeness came back at **0.033** on the SSO test (T10), which looked like a broken extractor. The trace showed the opposite: the extractor had correctly captured the one-line request. The fault was the completeness judge, which was building its ground truth from the entire combined corpus — brief, five transcripts, notes — while the system had deliberately scoped the PRD to the chat request. The judge was penalising the system for correctly ignoring documents it was never asked about.
- **An evaluator I couldn't see into.** A later run reported **0.700 hallucination** on T11 and T12, outputs that plainly covered their input and nothing more. I couldn't explain it because the judge logged a bare score and no reasoning. After I added the judge's flagged items to the trace metadata, the next run showed `hallucinated_items: []` — the 0.700 had been a judge artifact on a three-item denominator, not real fabrication.
- **A precise, minor real defect.** The same metadata pinpointed the only genuine completeness gap: the extractor drops `PM: Sarah` (ground-truth item GT-03) on T11/T12. One specific missed item across two tests — not a systemic problem.

## 2. Improvement plan

In priority order, each grounded in a trace observation:

1. **Fix the "PM: Sarah" miss.** Add an explicit instruction to the extractor to capture named role assignments (PM, owner). It's a one-line change and the only real completeness gap in the suite.
2. **Reduce judge variance on small denominators.** T5's 0.667 hallucination came from a two-item base where one flagged item dominates. Run each baseline test 3× and report the mean and range rather than a single number.
3. **Make the judge auditable by default** (done) and keep it — the metadata logging turned two unexplained scores into one-look diagnoses.
4. **Human-in-the-loop** confirmation of extracted requirements before generation, as the cheapest defence against the fabrication failure mode.

## 3. Risks of AI-generated PRDs

- **Fabrication that reads as competence.** The dangerous output is the plausible one — a confidently stated requirement nobody asked for, forwarded to stakeholders and built against. This is why the design withholds the raw source from the PRD Generator rather than merely instructing it to be careful.
- **False confidence from format.** A complete 10-section template signals thoroughness regardless of whether the request supported it; an automated system makes that claim on every run.
- **Scope creep from supporting documents.** Left unchecked, a document that mentions a feature nobody requested would bloat the PRD. The scope boundary (Guardrail C) exists precisely to prevent this — and the traces confirmed it holds (T2, T9 halted despite four documents being available).
- **Traceability decay.** Source IDs are only meaningful while someone checks them. A team that stops auditing the Source column has a system that looks grounded and isn't.

## 4. Connection to the course's evaluation skills

The core skill this course teaches is the loop: **evaluate → find a failure → fix → re-evaluate → confirm.** I ran it twice, and the second time is the one I'm proudest of, because the thing I fixed was the *evaluator*, not the pipeline.

The baseline dataset caught what ad-hoc testing would have missed. The vague and contradictory inputs — T2, T3, T5, T9 — are where a system that looks fine on happy-path testing falls over, and I would not have written those inputs myself. The least intuitive idea I absorbed is that a confident, complete answer to a vague input is a *failure* even when it reads well — which is why correct refusal (T2, T9 halting) counts as a pass, and why hallucination and completeness must be reported together: driving either to a perfect number alone is trivially gamed.

If this ran in production, I would measure two things the baseline can't: a PM-facing acceptance rate — what fraction of PRDs are approved without substantive edits — and the hallucination rate on real requests rather than 12 curated ones. The curated suite proves the mechanism; only real use proves the value.
