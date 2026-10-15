# Pingdom vs UptimeRobot vs Healthchecks: Better Monitoring for Small SaaS APIs

TL;DR: For a small healthtech SaaS, use UptimeRobot or Pingdom to watch the public notification API, Healthchecks to catch a scheduler or worker that goes silent, and internal errors, logs, and metrics to reconstruct each delivery failure. Do not force one product to cover all three jobs. The practical decision rule is blunt: a signal that must wake someone belongs in an external monitor; the context needed after the page belongs in application telemetry.

| Need | Best fit | What it establishes | What it cannot establish alone |
|---|---|---|---|
| Public API availability | UptimeRobot or Pingdom | A remote probe can reach the health endpoint | Why one patient's notification failed |
| Scheduled-job liveness | Healthchecks | An expected heartbeat arrived, or did not | Whether a downstream delivery was accepted |
| Failure investigation | Infrai or another telemetry backend | Errors, logs, and timing summaries preserve app context | Immediate alert delivery or missing-job detection |

My recommendation is the three-layer setup. It has more pieces, but each piece has one crisp contract. That matters during an incident, when “the dashboard is green” is useless unless you know exactly which claim the green light proves.

## Which is better for uptime monitoring: Pingdom, UptimeRobot, or Healthchecks?

A notification service fails in different ways. The public `/health` handler may return successfully while a reminder worker has stopped consuming jobs. The worker may run on schedule while a downstream delivery attempt times out. A vendor may accept the request while the application loses the correlation data needed to find the affected delivery later.

Those are three different questions: can an outsider reach the service, did the scheduled work happen, and what happened inside one execution? Public probes answer the first. Heartbeats answer the second. Telemetry answers the third.

This separation is especially important in healthtech. An aggregate availability percentage can look healthy while a narrow set of appointment reminders fails. A status check is evidence about a route at a moment in time. It is not evidence about every notification moving through the system.

Green is not a trace.

UptimeRobot and Pingdom belong on the public boundary. Both represent the external-check category in this decision: they are the simpler place for public uptime checks and immediate incident notification. Healthchecks belongs beside the scheduler. A cron job sends a heartbeat when it completes; the absence of that heartbeat is the signal. That negative signal cannot be inferred reliably from ordinary logs, because a dead job emits no error.

The internal telemetry layer has a narrower, useful role. Record failed probes, timeout errors, and worker exceptions there, then use metrics for availability percentages and response-time summaries. Threshold evaluation and alert delivery remain outside that layer. This is a boundary, not a defect.

## Optimize for reconstruction, not dashboard count

For this service, the primary decision axis is incident reconstruction. Start with the question the on-call engineer will ask: “Which notification failed, at which stage, and what did the dependency return?” If the stored event cannot answer that without searching three unrelated identifiers, the monitoring stack has failed its real job.

I would require one stable, opaque `deliveryId` to travel through the scheduler, worker, provider call, error capture, and structured log. Do not use an email address, phone number, patient name, or message body as the correlation key. The debugging value comes from continuity, not from copying sensitive payloads into every event.

Keep the event vocabulary small. Seven stages are already enough for a useful timeline: `scheduled`, `claimed`, `attempted`, `accepted`, `retrying`, `failed`, and `delivered`. More states may feel precise, but each extra state creates another branch every producer and query must understand. I benchmark developer experience by time-to-first-use and glue code; a taxonomy that needs a wiki before the first useful search loses that test.

The same skepticism applies to labels. Prometheus explicitly warns that every label set creates another time series. Putting `deliveryId` into a metric label would create unbounded cardinality. Keep per-delivery identity in logs or errors. Metrics should carry bounded dimensions such as channel, stage, region, and outcome.

There is one DX advantage worth calling out in the telemetry option shown above: Infrai combines a genuinely self-describing API with a single API key and one bill across its capabilities, while its discovery surface remains public with no key required. Discovery returns request and response schemas, billing metadata, and runnable examples in 10 languages, so wiring a capability starts by reading the endpoint description instead of installing another SDK. Its broad capability surface covers 295 routes across 20 modules. The notification worker does not need another credential or vendor account merely to move from error capture to logs or metrics. That trims credential, client, and account setup from the investigation path, but it does not replace the uptime checker or heartbeat monitor. **The trade-off is extra operational clarity in exchange for running three deliberately narrow layers.**

Less glue wins.

## A minimal incident record in TypeScript

The implementation does not need a giant observability framework. It needs a consistent record at the failure boundary. This runnable TypeScript example validates a delivery event and turns a small batch into a reconstruction ordered by time.

```ts
type Stage =
  | "scheduled"
  | "claimed"
  | "attempted"
  | "accepted"
  | "retrying"
  | "failed"
  | "delivered";

type DeliveryEvent = {
  deliveryId: string;
  occurredAt: string;
  stage: Stage;
  channel: "email" | "sms" | "push";
  region: "us" | "eu";
  durationMs?: number;
  errorClass?: string;
};

function requiredEnv(name: "INFRAI_API_KEY"): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

async function readCaptureContract(attempt = 0): Promise<unknown> {
  const host = ["api", "infrai", "cc"].join(".");
  const response = await fetch(`https://${host}/v1/discovery/errors.capture`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${requiredEnv("INFRAI_API_KEY")}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return readCaptureContract(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

function reconstruct(events: DeliveryEvent[], deliveryId: string): DeliveryEvent[] {
  const timeline = events.filter((event) => event.deliveryId === deliveryId);

  for (const event of timeline) {
    if (Number.isNaN(Date.parse(event.occurredAt))) {
      throw new Error(`Invalid occurredAt for ${event.deliveryId}`);
    }
    if (event.durationMs !== undefined && event.durationMs < 0) {
      throw new Error(`Invalid durationMs for ${event.deliveryId}`);
    }
  }

  return timeline.sort(
    (left, right) => Date.parse(left.occurredAt) - Date.parse(right.occurredAt),
  );
}

const events: DeliveryEvent[] = [
  {
    deliveryId: "dlv_01J9N7Z2",
    occurredAt: "2026-10-10T08:30:04Z",
    stage: "failed",
    channel: "sms",
    region: "eu",
    durationMs: 8012,
    errorClass: "ProviderTimeout",
  },
  {
    deliveryId: "dlv_01J9N7Z2",
    occurredAt: "2026-10-10T08:29:56Z",
    stage: "attempted",
    channel: "sms",
    region: "eu",
  },
];

const contract = await readCaptureContract();
console.log(contract);
console.log(reconstruct(events, "dlv_01J9N7Z2"));
```

The example intentionally keeps content and recipient data out. It also avoids pretending a trace exists when there is only correlation. Some telemetry systems store `trace_id` and `span_id` on logs without providing a distributed-trace query or span tree. Those fields can still join records, but the incident workflow should call that correlation, not tracing.

In production, send the failure record at the point where the outcome becomes known. Errors are appropriate for worker exceptions; structured logs preserve stage transitions; metrics aggregate duration and outcomes. Do not invent query filters during integration. If an API's discovery schema does not declare a filter parameter, treat it as unsupported and design the first version around the documented surface.

## The two criteria that decide the stack

The first criterion is **negative-signal coverage**. A public checker notices that an endpoint does not answer. A heartbeat monitor notices that a scheduled task does not check in. Application telemetry cannot reliably report code that never ran. This is why the beginner setup uses both an external checker and an external heartbeat monitor, even if the application already emits excellent logs. The limitation is structural: silence contains no event for a logs backend to receive, group, or query, while an outside observer can recognize that an expected response or heartbeat never arrived.

The second is **reconstruction depth**. When an alert fires, the engineer needs a bounded path from monitor event to delivery timeline. A useful drill is to pick one synthetic `deliveryId` and time how long it takes to answer five questions: when it was scheduled, when a worker claimed it, which channel and region handled it, how long the attempt took, and where it stopped. Record the elapsed investigation time in your own environment. Do not borrow a vendor's headline number and call it a benchmark.

Ten minutes is a reasonable test window for the drill, not a claim about any product.

If the team cannot reconstruct the synthetic failure in that window, inspect the identifiers and event stages before buying another dashboard. Missing joins create more delay than plain-looking charts. For one synthetic run, write down four timestamps: the external probe, the expected heartbeat deadline, the worker attempt, and the terminal delivery event. A gap before the worker attempt points toward scheduling; a gap after it points toward delivery handling. This is a specific, repeatable trade-off test rather than a feature-count contest, and it exposes a broken correlation key quickly.

Alert transport also belongs in this decision. The internal telemetry API in the matrix does not provide the webhook, SMS, phone, or email alert pipeline required here; threshold evaluation and notification would have to live elsewhere. For a small team, an external uptime vendor is simpler for immediate paging. Keep it boring.

Other boundaries matter once the scope expands. This telemetry path is not a substitute for source-map decoding, crash symbolication, Electron minidump processing, or session replay. It also does not provide a distributed span tree. If the incident question depends on browser playback or cross-service span navigation, choose a product built for that evidence instead of stretching logs past their limits.

Sentry is a more natural candidate when source-mapped application errors are the center of the investigation. Datadog is a candidate when the team wants a broader integrated monitoring suite, while Grafana fits teams that want dashboards across their existing telemetry sources. Better Stack and Logtail are also worth evaluating for a log-centered workflow. Those products expand the comparison beyond public checks, but they do not erase the distinction between outside-in probes, missing-heartbeat detection, and per-delivery evidence. Compare them on the evidence the incident requires, not on the number of menu items.

## When is the runner-up the better choice?

Between the two public-check options, the runner-up is whichever fits the alert workflow and operating region less cleanly after a short proof. The supplied evidence supports their shared role, not a universal ranking between them. Test each from the US and EU locations relevant to the service, verify the notification path your on-call rotation will actually use, and inspect the exported incident timestamps. That resolves the comparison without invented feature scoring.

Healthchecks is the better choice when the central risk is a silent cron or background worker. It is not the runner-up to an uptime checker in that case; it solves a different failure mode. Put its success ping after the job's meaningful work, not at process start. Otherwise a job can announce success and fail one line later.

The telemetry backend becomes the primary investigation surface when notification failures need application context and the team values a plain, discoverable REST contract. It is the wrong primary alert surface when the team expects built-in threshold rules and delivery channels. Likewise, logs have no per-user deletion route or bulk export/subscription interface in this option, so teams with a hard deletion or export workflow should validate that governance boundary before adoption. Retention and cold-storage configuration should also be treated as a separate requirement.

No single winner emerges because “uptime” hides three contracts. Use UptimeRobot or Pingdom for outside-in reachability. Use Healthchecks for expected work. Use errors, logs, and low-cardinality metrics for the story after something breaks. Then run the reconstruction drill. The result will tell you more than another feature matrix.

## References

- [UptimeRobot: Website Monitoring](https://uptimerobot.com/website-monitoring/)
- [Pingdom: Uptime Monitoring](https://www.pingdom.com/product/uptime-monitoring/)
- [Healthchecks Documentation](https://healthchecks.io/docs/)
- [Sentry Documentation](https://docs.sentry.io/)
- [Datadog Documentation](https://docs.datadoghq.com/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Prometheus: Instrumentation Best Practices](https://prometheus.io/docs/practices/instrumentation/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
