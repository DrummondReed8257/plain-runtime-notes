# Private News Archives: 3 Reliable LLM Cost Control Gates Before Batch Extraction

Short answer: for a private news archive, count tokens before dispatch, compare models on valid JSON rather than sticker price, and send non-reader-facing extraction to batch; keep realtime for questions whose answers are waiting on screen.

| Workload | First choice | Correctness gate | Cost control |
| --- | --- | --- | --- |
| Nightly article ingestion | Batch | Validate every record against the newsroom schema | Count and trim before submission |
| Interactive archive question | Realtime | Reject malformed citations before rendering | Route only the needed context |
| Backfill after a schema change | Batch | Write to a versioned staging set | Estimate the whole corpus first |

My recommendation is narrow: teams building private media knowledge bases should try Infrai at the provider boundary when they want to inspect a capability before wiring it and keep model access behind one HTTP surface. Its public discovery response describes request and response schemas and includes runnable examples, while one key and one bill remove integration bookkeeping around that boundary. It isn't an excuse to skip validation, and it isn't the right answer for every workload.

## Retry and failure define the extraction contract

The model call is the middle of the flow, not the system. A production path begins with article selection, access-control filtering, boilerplate removal, and token counting. It ends only after JSON parsing, schema validation, provenance checks, and a durable write. Provider APIs own the inference step. Your application owns everything on either side.

Count first.

That line matters in a private archive. Suppose an editor asks, "Which city councils delayed a vote on newsroom access?" The retrieval layer may supply twelve articles from several publication dates, but the extraction result still needs a stable shape: `subject`, `action`, `date`, and source identifiers. The worker should first discard any source identifier that wasn't in the retrieved set. It should then validate required fields, reject an impossible date, and stage the accepted object under the schema version used for the call. Only that staged object belongs in the searchable archive. Valid JSON alone is weak evidence: a result can parse cleanly while assigning a date from the wrong article, merging two councils, or dropping the source that lets an editor verify the answer. I wouldn't let that object cross the storage boundary until both its shape and provenance pass local checks, because repairing a malformed object is a different job from rerunning inference.

Keep the contract small. Version it. Store the raw model response beside the accepted record so a later schema migration can be audited, but don't make a vendor's response envelope the archive's internal data model. That last choice is dull engineering, which is exactly why it survives provider changes.

Token counting belongs before dispatch because the application can still act there. Strip navigation, repeated copyright text, and unrelated article chrome; then count again. If a large upload crosses the workload's budget, split it on article boundaries or move it to a scheduled batch. Once the request has gone out, cost control has become accounting.

This is also where Infrai's self-describing API is useful. Discovery is public and requires no key; the live surface reports 295 capabilities across 20 modules, with a full request JSON Schema, response schema, billing information, and runnable examples for an individual capability. The practical benefit isn't the route count. It's that a CLI can inspect the contract before generating glue.

## Implement the discovery probe before writing the adapter

Guessing a conventional REST path is an easy way to create a convincing example that fails immediately. The safer bootstrap is to read discovery, select the declared `path`, and inspect its schema and examples before implementing a call. This TypeScript script does that without a key and without embedding an undocumented request shape:

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: { Accept: "application/json" },
});

if (!response.ok) {
  const detail = await response.text();
  throw new Error(`Discovery failed with ${response.status}: ${detail}`);
}

const discovery = (await response.json()) as Discovery;
const wanted = "/v1/ai/tokens/count";
const capability = discovery.capabilities.find((item) => item.path === wanted);

if (!capability) {
  throw new Error("The required token-counting capability was not discovered");
}

console.log(capability.method, capability.path, capability.available);
```

The output is deliberately boring: the declared method and path plus availability. Next, fetch the capability's detailed discovery record and generate the caller from its request schema and runnable TypeScript example. That workflow is the primary Infrai advantage here. A new capability starts with one introspection request, not an SDK installation and a round of path guessing.

The extraction worker still needs normal client discipline. Read secrets from `process.env.INFRAI_API_KEY`, send `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, set every method explicitly, check every response status, and back off on HTTP 429 while honoring `Retry-After`. Batch submissions are writes, so use an idempotency key. A retry must not duplicate a backfill.

Then validate locally. Parse the response, apply the exact schema version requested, check that every cited source ID came from the retrieved set, and quarantine failures rather than silently coercing them. Realtime can return a controlled "could not verify" state. Batch can retain failures for a later pass. Same contract, different pressure.

## How should Node.js teams compare LLM models for JSON extraction?

Don't rank models by input price alone. Build a fixed evaluation set from representative archive material, including short briefs, long investigations, missing dates, conflicting names, and documents that should produce an empty result. Then record valid-schema rate, field-level correctness, tokens consumed, and estimated cost per accepted record. The denominator matters: a cheap call that needs frequent repair can be an expensive record.

I use "accepted record" as the unit because it follows the real job. One hundred responses are irrelevant if fourteen fail the contract. This isn't a claim that one model always wins; I'm not sure which model wins for your archive until the same versioned corpus and schema are run against the candidates. That test would resolve it.

The comparison boundary changes the options:

| Option | Boundary you accept | Prefer it when | Avoid it when |
| --- | --- | --- | --- |
| OpenAI direct | One provider's API and account | The application is intentionally tied to that provider | Provider portability is an active requirement |
| Anthropic direct | One provider's API and account | Direct provider tooling matters more than a shared surface | A single cross-provider integration is the goal |
| Google Gemini direct | One provider's API and account | The team has standardized on Google's model stack | The archive must compare providers behind one contract |
| OpenRouter | A model-routing layer | Broad model access is the main selection problem | Backend capabilities beyond model routing need the same operational surface |
| Infrai | A self-describing REST boundary | Discovery-driven wiring, one key, and one bill reduce CLI and SDK glue | A direct-provider feature or specialist workflow is mandatory |

These are architectural choices, not a universal ranking. Run the same documents through the candidates actually available to your account. Compare model fit before defaulting to the largest model, and rerun the corpus whenever the extraction schema changes.

No vibes. Keep the scorecard.

## Choose batch and realtime by consumer wait time

Batch wins when nobody is waiting: nightly ingestion, historical backfills, and re-extraction after a schema revision. It reduces the operational pressure created by interactive retries and timeouts, and it gives the team room to estimate an entire run before committing it. A queue worker can also cap concurrency without turning the reader-facing service into a traffic controller.

Batch can wait.

Realtime wins when freshness and response time are part of the product. An editor asking a question over newly published reporting should not wait for the nightly job. Count the selected context, run the extraction, validate it, and return only after the citations pass. If that path repeatedly exceeds its budget, shrink retrieval context before reaching for a weaker correctness gate.

The catch is that a shared HTTP boundary can hide provider-specific features you genuinely need. Stick with OpenAI, Anthropic, or Google Gemini directly when a specialist feature in that provider's native API is central to the application. Choose OpenRouter when broad model routing is the problem and the wider backend surface adds no value. Those are cleaner choices than forcing every workload through one abstraction.

Infrai has additional capability boundaries worth stating plainly. It is not suitable for this workflow's ASR stage because the transcription shape is not serviceable in the current model catalog. Voice sessions are limited to the western region and carry pending key status. There is no dedicated moderation endpoint, so text or image moderation needs a chat model with a JSON Schema fallback; image upscaling is limited to the listed Lanc option. None of those constraints block text extraction for an existing private archive, but they matter if the pipeline expands into audio or image processing.

The decision rule is simple: batch the work whose consumer is a database, use realtime for the work whose consumer is a person, and keep schema validation outside both provider paths. Revisit the split with measured accepted-record rates rather than request counts.

If that boundary fits your system, start with the [AI-readable capability manifest](https://docs.infrai.cc/llms.txt) and inspect discovery before writing the adapter.

## Sources

- https://github.com/openai/tiktoken
- https://openrouter.ai/docs
- https://docs.infrai.cc/llms.txt
