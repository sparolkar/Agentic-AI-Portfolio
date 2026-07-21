# Q2 — Program Charter (15 points)

**Program:** PRD Genie — AI-Powered Product Documentation Assistant
**Sponsor:** Product & Innovation, NeuronForge Technologies
**Program lead:** Shailendra Parolkar (AI PM/TPM)
**Status:** Pilot · Version 1.0

## Vision

Every requirement that reaches engineering can be traced to something a person actually said. PRD Genie turns meeting transcripts, briefs, and stakeholder notes into consistently structured PRDs, epics, and user stories — and where the conversation left a gap, it returns the question to ask rather than a plausible guess.

## Objectives

| # | Objective | Measure |
|---|---|---|
| O1 | Cut transcript-to-PRD time | 60–90 min manual → under 10 min including PM review |
| O2 | Standardise PRD structure across authors | 100% of generated PRDs conform to `prd_template.md` |
| O3 | Keep generated content traceable | Every functional requirement carries a Source ID resolving to a verbatim quote |
| O4 | Surface gaps at meeting time, not mid-sprint | Clarification questions produced on every run, including runs that produce no PRD |
| O5 | Keep unit economics negligible against PM time | Under $5 per PM per month |

## Scope

**In scope.** Ingestion of transcripts, product briefs, and stakeholder notes as text; requirement extraction with stated/ambiguous separation; PRD generation against the standard template; epic and user story breakdown; gap analysis producing stakeholder questions; observability via Langfuse; chat interface for interactive use.

**Out of scope for v1.** Live integrations with Jira, Confluence, or Notion; audio transcription (text input only); multi-document synthesis across several meetings; automatic estimation or sprint planning; writing back to any system of record. PRD Genie produces documents; humans decide what to do with them.

**Explicitly not automated.** Approval. A generated PRD is a draft until a PM signs it off. The system is designed to make review fast and targeted, not to remove it.

## Success criteria

| Criterion | Target | How measured |
|---|---|---|
| Extraction completeness | ≥ 90% of stated requirements captured | Baseline dataset T1–T12 against expected-output criteria |
| Hallucination rate | ≤ 5% of PRD items not traceable to source | Source column audit + `unsupported_items_dropped` from traces |
| Correct refusal | 100% on insufficient inputs | T9 and T5 must not produce a PRD |
| Format compliance | 100% | All 10 template sections present, including "Not discussed" sections |
| Time saved | ≥ 45 min per meeting | PM time log, pilot cohort |
| Cost per PM per month | < $5 | Token accounting from traces |

## Timeline

| Phase | Duration | Exit criteria |
|---|---|---|
| Design | Week 1 | Agent specs, orchestration pattern justified, architecture diagram |
| Build core | Weeks 2–3 | Extractor → PRD → stories running end to end on T1 |
| Extended capability | Week 4 | Gap Analyzer integrated; sufficiency gate passing T9 |
| Evaluate | Week 5 | Langfuse connected; all 12 baseline tests run and documented; one fix loop completed with before/after evidence |
| Pilot | Weeks 6–8 | 5 PMs, 20+ real meetings, time-saved and hallucination data collected |
| Decision | Week 9 | Go / no-go on wider rollout |

## Risks

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Fabricated requirements reach engineering | High — a sprint built on something nobody asked for | Medium | Quote-or-drop validation in code; PRD Generator isolated from raw transcript; Source column makes fabrication inspectable |
| Template-completion pressure fills empty sections | High | High without mitigation | Explicit "Not discussed in this meeting." literal; empty sections declared correct output |
| Over-refusal — system too conservative to be useful | Medium — adoption stalls | Medium | Track extraction completeness alongside hallucination rate; a fix that lowers both is a regression, not a win |
| PMs treat drafts as approved documents | High | Medium | Draft status in the PRD header; pilot training positions output as a starting point |
| Model or pricing changes shift cost or behaviour | Low | Medium | Model choice isolated per agent; baseline dataset re-runnable as a regression suite |
| Key or credential leakage | Medium | Low | Credentials in n8n credential store, never in exported workflow JSON |

## Stakeholders

| Stakeholder | Interest | Engagement |
|---|---|---|
| PMs / TPMs | Primary users; time saved, trustworthy drafts | Pilot cohort; weekly feedback |
| Engineering leads | Consistent, estimable PRDs | Review of generated PRDs in weeks 6–8 |
| Design leads | Personas and requirements reflect user needs accurately | Sample review |
| VP Product | Time-to-market, adoption | Phase-gate reviews |
| Data / Security | Meeting content handling, credential hygiene | Review before pilot expands beyond 5 users |

## Rollout plan

**Phase 1 — Pilot (5 PMs, weeks 6–8).** Chat interface only. Every generated PRD is reviewed against its source transcript by the PM who ran it, and corrections are logged. The goal of this phase is a hallucination number from real meetings, not adoption.

**Phase 2 — Team rollout (one product team, weeks 9–12).** Add the team's own past PRDs as few-shot examples. Introduce a human-in-the-loop review step where extracted requirements are confirmed before PRD generation runs.

**Phase 3 — Org-wide (quarter 2).** Integrate with the document system of record. Revisit fine-tuning the extractor if the trace data shows a persistent, characterisable failure mode worth training against.

**Rollback.** Each phase is gated on the hallucination rate holding at or below target on real meeting data. If it degrades, the program returns to the previous phase rather than proceeding — the failure mode this system exists to prevent is precisely the one that scaling would amplify.
