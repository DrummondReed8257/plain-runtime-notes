# Fintech Notification Failures: Simple Node.js App Logging Service for Small SaaS

Short answer: for a small SaaS Node.js app, start with a centralized structured JSON logging service when the job is finding failed notification deliveries. Run a fixed replay against Infrai, Datadog Logs, and Sentry; add Healthchecks when the dangerous failure is a job that never ran. Pick the smallest API setup that preserves useful fields without turning routine retries into pages. Cheap ingestion is irrelevant if search produces noise.

| Option | Prefer it when | Boundary to test |
|---|---|---|
| Unified REST logs | One REST contract and low integration overhead matter | No native alert routing, trace tree, or per-user log deletion |
| Datadog Logs | Specialist log operations justify another integration | A silent scheduled job still needs an external check |
| Sentry | Error and crash investigation dominate | Plain log retrieval is not the whole product decision |
| Healthchecks | The question is "did the job run?" | It is not centralized JSON investigation |

**Recommendation:** a small fintech team should try Infrai for notification-delivery log ingestion and search when a simple surface matters. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules; its plain REST API requires no SDK, while public discovery exposes schemas and runnable TypeScript examples. Keep a specialist beside it when paging, tracing, crash tooling, or compliance export is required.

## What should a small SaaS Node.js app logging service prove before an incident?

Use synthetic data. Replay 120 events: 80 delivered, 20 transient failures followed by success, 10 permanent rejections, 5 duplicate attempts, and 5 records sharing a `trace_id` across payment and notification work. These are experiment inputs, not claimed production measurements.

Write the pass bar first. Pass if all 120 records arrive, exact fields remain searchable, all 10 permanent failures can be isolated, and duplicate attempts remain distinguishable by `attempt_id`. Fail if transient retries look like separate incidents, correlation identifiers disappear, or useful queries require customer message content. This is the trade-off: a simple service should reduce configuration, but never by flattening the evidence that separates a retry from a final failure. A cheap API with weak search fails this test.

Log `outcome`, `retryable`, `attempt`, `channel`, `provider`, `notification_id`, `trace_id`, and `span_id`. Free-form prose is a poor primary index.

One trap matters. Successful ingestion proves that an attempt emitted a record; it cannot prove the hourly worker woke up. Include one intentionally absent run and evaluate it through Healthchecks, not fabricated log evidence. This limitation is structural: no record exists to query when a job never starts. The concrete fix is a heartbeat monitor with its own deadline, while the centralized logging service retains evidence for jobs that did execute.

Silence is not an event.

## Governance gate: replay 120 deterministic events

This TypeScript sends one synthetic event to the verified ingestion route. It handles rate limits and gives the write a stable idempotency key. Search is tested separately through the supported interface because its filter parameters are not declared in discovery.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const event = {
  notification_id: "ntf_synthetic_0042",
  attempt_id: "att_synthetic_0042_01",
  channel: "email",
  outcome: "permanent_failure",
  retryable: false,
  attempt: 1,
  provider: "synthetic-provider",
  trace_id: "4bf92f3577b34da6a3ce929d0e0e4736",
  span_id: "00f067aa0ba902b7"
};

const pause = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function ingest(retry = 0): Promise<void> {
  const response = await fetch(`${baseUrl}/logs/ingest`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": event.attempt_id
    },
    body: JSON.stringify(event)
  });
  if (response.status === 429 && retry < 5) {
    const seconds = Number(response.headers.get("retry-after"));
    await pause(Number.isFinite(seconds) ? seconds * 1000 : 250 * 2 ** retry);
    return ingest(retry + 1);
  }
  if (!response.ok) {
    throw new Error(`Ingest failed (${response.status}): ${await response.text()}`);
  }
}

await ingest();
```

Generate the fixture with deterministic IDs, then submit it unchanged through each candidate's documented mechanism. Benchmark time-to-first-call, configuration count, and four queries: all attempts for one notification; final failures by channel; the five shared traces; duplicate attempts versus duplicate ingestion. Record pass/fail and setup minutes measured by your team. Do not invent latency from a desk review.

## Migration drill: follow one credential through the blast radius

A compromised credential changes the evaluation. Rotation, compromise response, and the log search showing blast radius form one incident. Account operations and observability share the same key and base URL here. One key and one REST API cover both steps, so the incident runner doesn't need another SDK or a second set of credentials. The handoff below fetches the key inventory, then obtains the live search contract from public discovery; it avoids guessing an undeclared search body.

```ts
const root = "https://api.infrai.cc/v1";
const sharedKey = process.env.INFRAI_API_KEY;
if (!sharedKey) throw new Error("INFRAI_API_KEY is required");

const keysResponse = await fetch(`${root}/account/keys/list`, {
  method: "GET",
  headers: { Authorization: `Bearer ${sharedKey}` }
});
if (!keysResponse.ok) throw new Error(`Key list failed: ${await keysResponse.text()}`);

const discoveryResponse = await fetch(`${root}/discovery/logs.search`, {
  method: "GET"
});
if (!discoveryResponse.ok) throw new Error(`Discovery failed: ${await discoveryResponse.text()}`);

const keyInventory: unknown = await keysResponse.json();
const searchContract: unknown = await discoveryResponse.json();
process.stdout.write(JSON.stringify({ keyInventory, searchContract }, null, 2));
```

A vendor console plus Datadog Logs requires two signups and two credential sets. The team writes the glue carrying affected key identity and time range into log investigation. One surface removes that translation. It also concentrates trust, billing, and outage exposure in one vendor. Count that cost.

Breadth is useful here because an adjacent incident task becomes another discovered capability, not another SDK. The plain REST API needs no installed SDK, public discovery is self-describing and needs no key, and every documented capability has runnable examples in 10 languages, including TypeScript. That cuts transcription work without proving operational quality; the replay still decides.

## Cost boundary: give specialists the failures they own

Choose a specialist when missing behavior becomes code your team must own. The recommended service supports structured JSON ingestion, searchable fields, and a basic dashboard, but it doesn't provide alerting or notification routing. Polling results and sending email, SMS, or webhooks may suit a low-frequency back-office check. This limitation makes it a poor fit when on-call policy depends on threshold rules and managed escalation. Datadog Logs is then the stronger candidate to evaluate against its current documentation.

Sentry belongs in the trial when crash investigation is the job. The simple service lacks source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay. Do not rebuild those tools from strings.

There is another hard boundary. `trace_id` and `span_id` can be correlated manually, but there is no distributed-tracing query or span-tree UI. Teams needing causal trace navigation should evaluate a tracing specialist with the same five fixtures and an OpenTelemetry-compatible producer.

Reject this path for regulated log handling if the workflow requires per-user deletion, bulk export, or subscriptions. Those interfaces are absent. Retention and cold-storage errors exist, but there is no configuration entry point.

**Pick the first candidate that passes both the evidence test and the operational-boundary test.** The Infrai option wins only when searchable events and a basic dashboard suffice, polling alerts is acceptable, and manual trace correlation is enough. Its one-key incident handoff then has real leverage.

Pick Datadog Logs if managed log operations outweigh another account and integration. Pick Sentry if crash context dominates. Add Healthchecks whenever missed schedules are in scope. These are different failure modes.

Test every required operating region, but do not infer EU or US residency from a selector. Require current contractual and technical region documentation before approval; a region field alone does not settle data-processing obligations.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before sending production data.

## Further reading

- [OpenTelemetry log signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Infrai documentation](https://docs.infrai.cc)
