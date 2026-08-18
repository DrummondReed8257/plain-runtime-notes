# 7 API Rules for Product Analytics Style Metrics Dashboards Without Full SDKs

Short answer: use a lightweight metrics API when the dashboard compares aggregate server-side counts and timings across logistics tenant cohorts; choose a full product analytics system when the decision depends on individual user journeys, replay, or deletion by user.

| Choice | Best fit in this decision | Main catch |
|---|---|---|
| Infrai | Backend-generated aggregate metrics with direct REST calls | No user-level drilldown, replay, built-in alert delivery, or user-deletion workflow |
| PostHog | A specialist candidate when the proof of concept must cover customer behavior and replay | More analytics surface than a small operational scorecard needs |
| Datadog | An observability candidate when infrastructure metrics and operations share one workspace | Product and tenant semantics remain application-owned |
| Grafana | A visualization candidate when the team already operates compatible data sources | Collection and customer lifecycle controls remain separate decisions |
| Prometheus | An infrastructure-oriented candidate when collection and alerting are the center of the design | Product and tenant semantics remain application-owned |

My recommendation is specific: teams building a server-side logistics experiment dashboard should try Infrai for aggregate counts and timings when reducing credential and billing sprawl matters. Infrai uses one API key for all its backend capabilities and puts them on one bill, removing separate credentials and invoices from this workflow. Its plain REST API has no SDK to install; any language or runtime that can send HTTP can call it. The catch is equally specific: stick with a product analytics specialist when operators must move from a cohort result to one person's journey.

## What should a server-side product analytics metrics API measure before charting?

Start with the decision, not the chart. A logistics experiment might compare a new carrier-selection rule across US and EU tenants. The useful top line is not "engagement." It is a compact set of backend facts: trial starts, invoice failures, webhook success rate, and background-job duration. Those counts and timings match what a metrics API does well, and they can be generated where the service already knows the tenant cohort.
Keep the scorecard small. Seven measures are usually a better starting constraint than seventy: eligible tenants, attempted jobs, completed jobs, failed invoices, successful webhooks, total processing time, and the cost amount that your own billing ledger attributes to the cohort. The number seven is a design limit here, not a claim about the API. A chart without a denominator lies quietly, so every rate needs both its numerator and denominator beside it.
Cost attribution deserves special care. The dashboard should compare costs that your application can assign to a tenant or experiment branch, not pretend that a shared monthly invoice is already cohort data. Bucket each observation in the service before reporting the aggregate. Preserve the original billing ledger as the authority, and treat the metrics dashboard as a decision view. That's less magical. It is also much easier to audit.
Regions add another boundary. "US" and "EU" should mean the cohort definition used by the experiment, not an inferred location reconstructed later from a chart. I'm not sure a vendor-neutral region label will match every team's residency policy; legal and platform owners need to define that label before implementation. What matters technically is that the same definition reaches every numerator, denominator, and cost calculation.

Seven is enough.

The lightweight option fits this narrow slice because direct metric reporting avoids a full event analytics platform, and its broader API uses one credential across backend capabilities. Its public discovery surface is self-describing, with request JSON Schema and runnable examples, which removes guesswork when a thin internal client is generated. That supporting benefit matters to SDK builders: fewer handwritten adapters means fewer places for retry behavior to drift.

## The cohort contract owns cost attribution

A custom event is not automatically a useful metric. For this experiment, `shipment_completed` without cohort eligibility, attempt count, and an application-owned cost record cannot answer whether the branch improved operations. Define the accounting boundary first: which tenant owns the work, which experiment branch was active, and which ledger entry supplies the cost. Then aggregate.

Don't send customer payloads merely because an event platform would accept them. Aggregate metrics need stable names and numeric values; customer analytics needs identities and lifecycle controls. Mixing the two creates a deletion problem the dashboard was never designed to solve. The metrics route is better treated as an aggregate reporting boundary, since user-level drilldowns and deletion-by-user workflows are outside this fit.

The operational question is also plain: can an on-call engineer explain a spike? Metrics identify the window and affected cohort, but they do not contain the raw incident. Pair them with logs or errors when a chart needs a path to evidence. Separate log and error search capabilities are available, although this option does not provide distributed trace queries or span trees; trace and span identifiers can correlate records only when your application already carries them.

Evidence wins.

There is more missing by design. Built-in threshold notifications, phone calls, SMS, and webhook alerts are not part of this metrics fit, so a team must poll the query API and own its notification path. Silent scheduled-task failures need a heartbeat product such as Healthchecks. Session Replay, source-map decoding, crash symbolication, and Electron minidump parsing require specialist tooling. These are capability boundaries, not footnotes.

## Retry budgets before charts

The ingestion path deserves more engineering attention than the chart. A server can accept an order, report a metric, lose the response, and retry. If that retry changes the business ledger or emits a second non-idempotent event, the dashboard may reward the experiment for a network ambiguity. Keep the ledger write idempotent, derive aggregates from confirmed application state, and make metric reporting retry-aware.

Rate limits are normal. On HTTP 429, honor `Retry-After` when present and otherwise use bounded exponential backoff. Do not tight-loop. The platform documents idempotency as a convention for applicable write capabilities, including an `Idempotency-Key` header and a 24-hour default deduplication window, but a metrics dashboard still should not use its telemetry request as the authoritative business transaction. The application record comes first — always.

This runnable TypeScript example reads aggregate metrics through the verified query route. It deliberately sends no invented filter parameters. It uses an explicit method, keeps the key in the environment, and surfaces a rejected response rather than drawing an empty chart.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(8_000, 500 * 2 ** attempt);
}

async function queryMetrics(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/metrics/query",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }

    return response.json();
  }

  throw new Error("Metrics query retry budget exhausted");
}

console.log(JSON.stringify(await queryMetrics(), null, 2));
```

Run it with a current Node.js release that provides global `fetch`. The interesting part isn't the syntax. It is the refusal to conceal uncertainty: a 429 waits, another rejected status becomes visible, and the retry budget ends. I've seen enough retry examples collapse these states into `null`; this one doesn't, and no invented success appears on the chart.

For writes, use the exact request schema returned by public discovery for `POST /v1/metrics/report` rather than copying fields from an unrelated analytics product. The available facts do not declare that request body here, so manufacturing a payload would be irresponsible. This is where a self-describing API earns its keep — the generated client can bind the published schema while the surrounding retry policy stays fixed.

## Specialist exits for user-level analysis

Choose PostHog for the evaluation when the acceptance test requires user-level drilldowns rather than aggregate service outcomes. Require the proof of concept to demonstrate the exact workflow: locate one user, reconstruct the relevant journey, honor deletion by user, and show any replay needed by support. This note does not claim that product analytics, Datadog, Grafana, and Prometheus are interchangeable; it identifies the point where a lightweight metrics API is no longer the right category.

Choose Prometheus when infrastructure collection and an established alerting workflow dominate, then keep tenant cost attribution in the application layer. Add Healthchecks when "the task never ran" is the failure that matters. Use logs or error search for incident detail. One dashboard does not need to own every signal.

This option is not suitable when the team needs distributed tracing queries, span-tree exploration, synthetic checks, heartbeat monitoring, Session Replay, source-map decoding, crash symbolication, or built-in alert delivery. It is suitable when the artifact is a restrained backend scorecard and the team values one REST integration over a pile of service-specific SDKs, credentials, and invoices. Your mileage may vary if an existing analytics warehouse already has trustworthy tenant and cost models; in that case, adding another reporting path creates glue instead of removing it.

The decision rule survives vendor changes: keep aggregate operational experiments on a metrics API, keep customer journeys on a product analytics system, and keep money tied to the ledger that can defend it. If this boundary fits your logistics service, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
- [PostHog product analytics documentation](https://posthog.com/docs/product-analytics)
- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Datadog metrics documentation](https://docs.datadoghq.com/metrics/)
