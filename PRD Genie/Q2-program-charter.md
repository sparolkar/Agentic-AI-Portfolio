# Q2 — Program Charter (15 points)

**Program:** PRD Genie — AI-Powered Product Documentation Assistant
**Sponsor:** Product & Innovation, NeuronForge Technologies
**Program lead:** Shailendra Parolkar (AI PM/TPM) · **Status:** Pilot · v1.0

## Vision

Every requirement that reaches engineering can be traced to something a person actually asked for. PRD Genie turns a product request — enriched by supporting meeting documents — into a consistently structured PRD, epics, and user stories, and where the request leaves a gap, it returns the question to ask rather than a plausible guess.

## Objectives

| # | Objective | Measure | Baseline result |
|---|---|---|---|
| O1 | Cut request-to-PRD time | 60–90 min manual → under 10 min incl. review | met in pilot testing |
| O2 | Standardise PRD structure | 100% conform to `prd_template.md` | format compliance 0.927 mean |
| O3 | Keep generated content grounded | every requirement traceable to a source quote | hallucination rate 0.068 mean |
| O4 | Surface gaps at request time | clarification questions on every run, including halts | achieved (all 12 baseline tests) |
| O5 | Keep unit economics negligible | under $5 / PM / month | measured $1.21 / PM / month |

## Scope

**In scope.** Chat request as primary input; a Google Drive folder of supporting documents that add detail within the request's scope; requirement extraction; PRD generation against the template; epic and user-story breakdown; gap analysis; Langfuse observability; three evaluation metrics.

**Out of scope for v1.** Live Jira/Confluence/Notion integration; audio transcription (text only); multi-request synthesis; automatic estimation or sprint planning; writing back to any system of record. PRD Genie produces documents; humans decide what to do with them.

**Explicitly not automated.** Approval. A generated PRD is a draft until a PM signs it off.

## Success criteria

| Criterion | Target | Measured |
|---|---|---|
| Extraction completeness | ≥ 0.90 | **0.948** mean |
| Hallucination rate | ≤ 0.10 | **0.068** mean |
| Correct refusal on insufficient input | 100% | T2, T9 halted ✓ |
| Format compliance | ≥ 0.90 | **0.927** mean |
| Cost per PM per month | < $5 | **$1.21** |

## Timeline

| Phase | Duration | Exit criteria |
|---|---|---|
| Design | Week 1 | Agent specs, orchestration pattern justified, diagram |
| Build core | Weeks 2–3 | Extractor → PRD → stories end to end |
| Extended capability + scope design | Week 4 | Gap Analyzer + chat-primary scope + sufficiency gate |
| Evaluate | Week 5 | Langfuse connected; 12 baseline tests scored; two documented fix loops |
| Pilot | Weeks 6–8 | 5 PMs, 20+ real requests, time-saved + hallucination data |
| Decision | Week 9 | Go / no-go on wider rollout |

## Risks

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Fabricated requirements reach engineering | High | Medium | Quote-or-drop in code; generator isolated from raw source; Source column makes fabrication inspectable |
| Supporting documents widen scope beyond the request | High | Medium | Scope boundary (Guardrail C): docs enrich only; out-of-scope items routed to `excluded_out_of_scope` |
| Over-refusal blocks usable input | Medium | Medium | Gate keys on stated requirements only; ambiguity surfaced, not suppressed |
| PMs treat drafts as approved | High | Medium | Draft status in header; pilot training positions output as a starting point |
| Model/pricing change shifts cost or behaviour | Low | Medium | Per-agent model choice; baseline dataset re-runnable as regression suite |
| Credential leakage | Medium | Low | Credentials in n8n store, never in exported JSON |

## Stakeholders

| Stakeholder | Interest | Engagement |
|---|---|---|
| PMs / TPMs | Time saved, trustworthy drafts | Pilot cohort; weekly feedback |
| Engineering leads | Consistent, estimable PRDs | Review generated PRDs weeks 6–8 |
| Design leads | Personas reflect real user needs | Sample review |
| VP Product | Time-to-market, adoption | Phase-gate reviews |
| Data / Security | Content handling, credential hygiene | Review before pilot expands |

## Rollout plan

**Phase 1 — Pilot (5 PMs).** Chat interface only. Every generated PRD reviewed against its source by the PM who ran it; corrections logged. Goal: a hallucination number from real requests.

**Phase 2 — Team rollout.** Add the team's past PRDs as few-shot examples; introduce a human-in-the-loop confirmation of extracted requirements before generation.

**Phase 3 — Org-wide.** Integrate with the document system of record; revisit fine-tuning the extractor if trace data shows a persistent failure mode.

**Rollback.** Each phase is gated on the hallucination rate holding at or below target on real data. If it degrades, the program returns to the previous phase — the failure this system exists to prevent is exactly the one scaling would amplify.
