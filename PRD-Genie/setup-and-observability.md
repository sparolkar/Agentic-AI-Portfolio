# PRD Genie — Workflow Setup & Observability

## Importing the workflow

1. n8n → **Workflows** → **Import from File** → `04-prd-genie-workflow.json`
2. Open each of the four `OpenAI ...` nodes and select your OpenAI credential (they're wired to the right agents already; only the credential is missing).
3. Confirm the model names are available on your key: `gpt-4o-mini` (extractor, gap analyzer) and `gpt-4o` (PRD, stories).
4. Run once with the default input — `Set Input Text` is pre-loaded with test T1.

To run a different test, edit `test_id` and `input_text` in the **Set Input Text** node. Output files are named `{test_id}-PRD.md`, `{test_id}-stories.md`, `{test_id}-clarification-questions.md`, so nothing overwrites as you work through T1–T12.

## What's on the canvas

24 nodes. The shape a grader will look for:

```
Set Input Text
   └─ Agent 1: Requirement Extractor (gpt-4o-mini, JSON mode, temp 0)
        └─ Parse + Validate Extraction  ← code node, drops unsupported items
             ├─ Sufficiency Gate (IF: stated_count > 0)
             │     ├─ true  → Agent 2: PRD Generator (gpt-4o)
             │     │            └─ Agent 3: Story Breakdown (gpt-4o)
             │     └─ false → Halt: Insufficient Input   ← PRD Generator never runs
             └─ Agent 4: Gap Analyzer (gpt-4o-mini)      ← parallel branch
                        ↓
                  Merge → Assemble → 3 × (Convert to File → Write)
```

Three things in there are worth pointing at explicitly in your writeup, because they're design decisions rather than plumbing:

**The Parse + Validate code node enforces the groundedness contract in code, not in the prompt.** Any extracted requirement or persona lacking a verbatim `quote` field is dropped from the payload and recorded in `dropped_items`, with a running `unsupported_items_dropped` count. So even if the model ignores the quote rule, the unsupported item never reaches the PRD Generator. A guardrail the model cannot talk its way past is stronger than one it merely agreed to.

**The Sufficiency Gate is why T9 passes by construction.** With `stated_count == 0` the false branch emits an explicit "no PRD generated" document and the PRD Generator is never invoked. You aren't hoping the model declines — it never gets the chance.

**The PRD Generator's input is `$json.extraction`, never the raw text.** Check this if you edit the node: the moment you pass the transcript alongside the JSON, you reopen the fabrication path the whole design closes.

`unsupported_items_dropped` also doubles as a free metric. If it's consistently above zero, your extractor is generating unsupported items and the drop count is your hallucination signal — before any hand-grading.

## Observability — read this before you wire it

I checked rather than assumed, and the situation is more awkward than the course materials imply: **n8n has no native Langfuse integration.** There are four working paths, none of them one-click:

| Path | What it gives you | Effort |
|---|---|---|
| **OpenTelemetry → Langfuse** | Full trace tree: per-agent input/output, tokens, latency, cost. This is the one that produces the screenshots the rubric wants. | ~30 min, needs Docker for self-hosted Langfuse |
| **Community node** `@langfuse/n8n-nodes-langfuse` | Prompt management only — does *not* send traces. Not sufficient on its own. | Low |
| **OpenRouter Broadcast** | Route LLM calls through OpenRouter, which forwards traces to Langfuse. Avoids OTel setup entirely. | Low-medium; changes your LLM provider |
| **Community node** `n8n-nodes-openai-langfuse` | Traces OpenAI and OpenAI-compatible models directly. | Low; third-party node |

For a capstone on a deadline, the OpenRouter path is the fastest route to real traces, and OpenTelemetry is the one that looks most like the class material. Either is defensible — and *saying which you chose and why* is itself the kind of tool rationale the Design sub-criterion rewards.

Whichever you pick, do it before you build agent 2. Per-agent traces are how you attribute a fabrication to the extractor versus the generator, and Q4 (5 pts) asks specifically for trace findings.

**Sources:**

- [n8n and Langfuse — Langfuse docs](https://langfuse.com/integrations/no-code/n8n)
- [Langfuse n8n node (prompt management)](https://langfuse.com/docs/prompt-management/features/n8n-node)
- [Deep n8n Observability with OpenTelemetry + Langfuse — n8n Community](https://community.n8n.io/t/deep-n8n-observability-with-opentelemetry-langfuse/190685)
- [Monitor AI chat interactions with Langfuse tracing — n8n workflow template](https://n8n.io/workflows/4972-monitor-ai-chat-interactions-with-gemini-25-and-langfuse-tracing/)
- [n8n + Langfuse integration discussion](https://github.com/orgs/langfuse/discussions/12598)

## What I could not verify

I built and structurally validated this JSON — unique node IDs, no dangling connections, no orphans, all four agent prompts correctly formed as expressions, model sub-connections wired to the right agents. **I could not run it**, because there's no n8n instance here.

So expect to fix small things on import, most likely:

- **Node typeVersions.** I targeted recent versions (`chainLlm` 1.5, `set` 3.4, `if` 2.2, `merge` 3.1). If your n8n is older, n8n will flag the node and you re-pick the operation from the dropdown — the parameters survive.
- **The `Merge` node's 3 inputs.** Only two of the three branches carry data on any given run (PRD+stories *or* the halt path, plus gaps). Confirm your merge mode passes through rather than waiting on all three; switch to "Append" if it stalls.
- **`readWriteFile` paths.** Files write to n8n's working directory. On n8n Cloud, filesystem writes are restricted — if you're on Cloud, delete the six file nodes and read the markdown from the `Assemble Submission Outputs` node output instead. Costs you nothing for grading.

Import it, run T1, and tell me what breaks — fixing it is faster than predicting it.

## Suggested test order

Not T1 through T12. Run these four first — they're the ones that reveal design problems rather than prompt problems:

1. **T9** ("Meeting happened. Notes: none.") — proves the sufficiency gate. If a PRD appears, stop and fix before anything else.
2. **T1** — proves the happy path, and shows how many of the ten PRD sections correctly read "Not discussed in this meeting." Expect seven or so. That's success, not failure.
3. **T3** (contradiction) — proves the extractor records both positions without resolving.
4. **T8** (three personas) — proves the personas array works and stories don't collapse into a generic "As a user."

If those four behave, the remaining eight are mostly confirmation. Then log everything in `03-baseline-test-log.xlsx`.
