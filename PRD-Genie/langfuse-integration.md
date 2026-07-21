# Langfuse Integration — Two Paths

n8n has no native Langfuse node for tracing. There are two workable routes, and they differ in effort and in what the trace looks like.

| | Path A — OTel env vars | Path B — ingestion API in-workflow |
|---|---|---|
| Code to write | **None** | ~60 lines in a Code node + one HTTP node |
| Works on n8n Cloud | No (needs env var access) | **Yes** |
| Trace granularity | Every n8n node, automatically | Exactly the spans you choose |
| Token counts | Real, from n8n's instrumentation | Whatever you can extract; may need estimating |
| Langfuse's own recommendation | **This one** | Legacy endpoint, still supported |

If you self-host n8n, do Path A. It's ten minutes and gives you real token numbers. Use Path B if you're on n8n Cloud, or if you want the guardrails to appear as named spans.

---

## Path A — OpenTelemetry (no code)

n8n exports OTLP traces natively; Langfuse accepts OTLP on `/api/public/otel`. You're just pointing one at the other.

**1. Build the auth header.** Langfuse uses HTTP Basic auth with your project keys:

```bash
echo -n "pk-lf-YOUR-PUBLIC-KEY:sk-lf-YOUR-SECRET-KEY" | base64
```

**2. Set these environment variables on your n8n instance:**

```bash
N8N_OTEL_EXPORTER_OTLP_ENDPOINT=https://cloud.langfuse.com/api/public/otel
N8N_OTEL_EXPORTER_OTLP_TRACING_PATH=/v1/traces
N8N_OTEL_EXPORTER_OTLP_HEADERS=authorization=Basic <the base64 string from step 1>
```

Use `https://us.cloud.langfuse.com/api/public/otel` for the US region, or your own host if self-hosting Langfuse. n8n appends `TRACING_PATH` to `ENDPOINT`, so the full URL becomes `.../api/public/otel/v1/traces`.

**3. Restart n8n and run the workflow.** Traces appear in Langfuse within a few seconds.

That's the whole integration. Every node becomes a span automatically, including your Parse + Validate and Sufficiency Gate nodes — which is exactly the trace tree we talked about earlier.

---

## Path B — Ingestion API from inside the workflow

More work, but you control the span structure, and it runs anywhere n8n can make an HTTP request.

### Where it goes

Insert two nodes **between** `Assemble Submission Outputs` and `Format Chat Response`:

```
Assemble Submission Outputs
   └─ Build Langfuse Batch   (Code)
        └─ Send to Langfuse  (HTTP Request)
             └─ Format Chat Response
```

Two required adjustments, or the chat reply breaks:

1. On **Send to Langfuse**, enable **Settings → On Error → Continue** (using error output). A Langfuse outage should degrade your observability, not your product.
2. In **Format Chat Response**, change the first line from `const j = $input.first().json;` to:

   ```javascript
   const j = $('Assemble Submission Outputs').first().json;
   ```

   The HTTP node replaces the item with Langfuse's response, so the formatter has to reach back for its data — the same `$('Node Name')` pattern used elsewhere in this workflow.

### Node 1 — "Build Langfuse Batch" (Code)

```javascript
// Builds a Langfuse ingestion batch: one trace, four generations for the agents,
// two spans for the guardrails. The guardrail spans are the point — they make the
// quote-validation and the sufficiency gate visible in the trace rather than
// implied by which generations are missing.

const asm  = $input.first().json;
const src  = $('Prepare Chat Input').first().json;
const meta = $('Parse + Validate Extraction').first().json;

const uid = () => crypto.randomUUID();
const now = new Date();
const iso = (offsetMs = 0) => new Date(now.getTime() + offsetMs).toISOString();

const traceId = uid();
const batch = [];

// Rough token estimate. Replace with real counts if your n8n version exposes
// tokenUsage on the chain nodes — check one node's output before trusting these.
const est = (s) => Math.ceil(String(s ?? '').length / 4);

const push = (type, body) =>
  batch.push({ id: uid(), type, timestamp: iso(), body });

// --- the trace ---------------------------------------------------------------
push('trace-create', {
  id: traceId,
  name: 'PRD Genie',
  sessionId: src.sessionId ?? undefined,
  input: src.input_text,
  output: asm.prd_markdown,
  tags: [asm.test_id, asm.input_assessment,
         asm.halted_at_sufficiency_gate ? 'halted' : 'completed'],
  metadata: {
    test_id: asm.test_id,
    stated_count: asm.stated_count,
    ambiguous_count: asm.ambiguous_count,
    contradiction_count: asm.contradiction_count,
    unsupported_items_dropped: asm.unsupported_items_dropped,
    halted_at_sufficiency_gate: asm.halted_at_sufficiency_gate,
  },
});

// --- generations -------------------------------------------------------------
const generation = (name, model, input, output, tOffset) => {
  const id = uid();
  push('generation-create', {
    id, traceId, name, model,
    startTime: iso(tOffset), endTime: iso(tOffset + 1000),
    modelParameters: { temperature: 0 },
    input, output,
    usageDetails: { input: est(input), output: est(output),
                    total: est(input) + est(output) },
  });
  return id;
};

generation('Agent 1 · Requirement Extractor', 'gpt-4o-mini',
           src.input_text, meta.extraction, 0);

if (!asm.halted_at_sufficiency_gate) {
  generation('Agent 2 · PRD Generator', 'gpt-4o',
             meta.extraction, asm.prd_markdown, 2000);
  generation('Agent 3 · Story Breakdown', 'gpt-4o',
             asm.prd_markdown, asm.stories_markdown, 4000);
}
generation('Agent 4 · Gap Analyzer', 'gpt-4o-mini',
           meta.gaps, asm.gap_markdown, 2000);

// --- guardrail spans ---------------------------------------------------------
push('span-create', {
  id: uid(), traceId, name: 'Guardrail · quote validation',
  startTime: iso(1200), endTime: iso(1250),
  level: asm.unsupported_items_dropped > 0 ? 'WARNING' : 'DEFAULT',
  statusMessage: `${asm.unsupported_items_dropped} unsupported item(s) dropped`,
  input: { items_received: meta.extraction?.stated_requirements?.length ?? 0 },
  output: { dropped: meta.dropped_items ?? [] },
});

push('span-create', {
  id: uid(), traceId, name: 'Guardrail · sufficiency gate',
  startTime: iso(1300), endTime: iso(1350),
  statusMessage: asm.halted_at_sufficiency_gate
    ? 'HALTED — no stated requirements; PRD Generator not invoked'
    : `PASSED — ${asm.stated_count} stated requirement(s)`,
  input: { stated_count: asm.stated_count },
  output: { proceeded: !asm.halted_at_sufficiency_gate },
});

return [{ json: { batch } }];
```

### Node 2 — "Send to Langfuse" (HTTP Request)

| Field | Value |
|---|---|
| Method | `POST` |
| URL | `https://cloud.langfuse.com/api/public/ingestion` |
| Authentication | Generic Credential Type → **Basic Auth** (user = public key, password = secret key) |
| Send Body | on, **JSON** |
| Specify Body | **Using Fields Below** → name `batch`, value `={{ $json.batch }}` |
| On Error | **Continue (using error output)** |

Two API behaviours that will confuse you otherwise: the ingestion endpoint returns **207**, not 200, and it reports per-event errors in the response body rather than as a 4xx. So a "successful" call can still contain rejected events — open the response once during setup and read it. Batches are capped at 3.5 MB, which a long transcript could approach; if you hit it, drop the raw `input` from the trace body and keep it on the extractor generation only.

---

## What I could not verify

I checked the endpoints, the auth scheme, and the env var names against current docs, but two things you should confirm on first run rather than trust from me:

**Token counts.** The estimate above is `characters / 4`, which is a convention, not a measurement. Before using these numbers in your cost analysis, open one `chainLlm` node's output in n8n and look for a `tokenUsage` field — if it's there, read from it instead. If it isn't, get your cost figures from the OpenAI dashboard rather than from Langfuse, and say in the writeup that Langfuse gave you structure and latency while costs came from the provider. That's an honest and defensible split.

**The `usageDetails` field name.** Langfuse has moved between `usage` and `usageDetails` across versions. If token numbers don't render in the UI, check the ingestion schema for your version and swap the key.

This is also why Path A is the better default where you can use it: n8n's OTel instrumentation reports real usage, so nothing has to be estimated.

**Sources:**

- [OpenTelemetry (OTEL) for LLM Observability — Langfuse](https://langfuse.com/integrations/native/opentelemetry)
- [OpenTelemetry configuration — n8n Docs](https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/opentelemetry)
- [Public API — Langfuse](https://langfuse.com/docs/api-and-data-platform/features/public-api)
- [Ingestion API reference — langfuse/langfuse](https://github.com/langfuse/langfuse/blob/main/fern/apis/server/definition/ingestion.yml)
- [Sending traces with nested spans via the public API](https://github.com/orgs/langfuse/discussions/9116)
- [n8n and Langfuse — Langfuse](https://langfuse.com/integrations/no-code/n8n)
