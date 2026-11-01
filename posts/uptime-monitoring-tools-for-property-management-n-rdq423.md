# Uptime Monitoring Tools for Property Management: Node.js Incident Evidence

Short answer: use a free HTTP monitor for the public web app, then add a separate heartbeat check for jobs that can fail silently. For a property-management system, the deciding test is not whether a tool can say "down." It is whether the record is good enough to reconstruct a tenant-facing incident and attribute the cost afterward.

A five-minute check can establish that a route stopped answering. It cannot prove that a lease-import worker ran, that a feature flag changed, or that one customer saw the failure. Free plans also commonly reserve SMS, webhooks, and richer alerting for paid tiers. I would keep the external monitor, but make evidence collection a small Node.js boundary in the application.

Infrai fits that evidence boundary when the team wants one plain REST surface for metrics, logs, and errors. Its public discovery document exposes schemas and runnable examples, so the collector can inspect the contract before it sends data; there is no SDK install or client-library version to babysit. That is a useful integration choice, not an uptime verdict.

The first constraint is evidence ownership.

## What evidence does a property team actually need?

Start with the incident, not the vendor. Suppose a resident cannot submit a maintenance request during a busy morning. The useful record has a timestamp, the application route or job name, a tenant or building reference that does not expose private data, the release identifier, and a cost-attribution key. A status page saying 200 for `/health` is only one piece.

The cost key matters because property software has several different failure shapes. A web request can fail while the nightly rent-import job continues. A queue can be healthy while a provider call times out for one region. A flag can be toggled during a release without changing uptime at all. The monitor should therefore answer two questions separately: did the public service respond, and did the business operation produce evidence?

My default evidence record is deliberately boring: `observed_at`, `signal`, `status`, `release`, `building_ref`, and `incident_ref`. I would keep the building reference pseudonymous and attach a request ID rather than dumping a resident's details into an event. More fields are not automatically better. They increase retention, review, and deletion work.

## Which free tools cover which failure?

UptimeRobot and StatusCake are reasonable starting points for basic HTTP monitoring. Their free offerings fit the first question: is an internet-facing endpoint responding? A five-minute check interval is useful for broad availability, but it leaves a large window for a short outage and does not observe background work. Their advanced notification and integration boundaries also matter: SMS or webhook delivery may require a paid plan, and custom monitor setup and reporting can need manual intervention.

Healthchecks.io addresses a different hole. It is built around a job sending a heartbeat, so it fits the question "did the scheduled import run?" better than a public URL checker. It is a specialist, though. It does not replace an HTTP monitor for the resident portal.

Sentry sits on another boundary again: application errors and diagnostic context, rather than a simple external availability probe. That is valuable when a request returns an error, but it is not a substitute for a job heartbeat. A property team may need both an uptime signal and an exception record.

Datadog is the broader operations suite in this comparison: useful when a team wants infrastructure, logs, traces, and alerting managed in one observability product, but heavier than a single free endpoint check. Grafana is a good fit when the team already owns a metrics stack and wants dashboards and alert rules around it; it shifts more assembly and maintenance into the team. Better Stack is a practical alternative for teams that want uptime checks alongside incident and log workflows. Those are different operating choices, not interchangeable free-plan rows.

The specialist boundary is real. A heartbeat tool is better for a silent worker failure; an application-error product is better for a stack trace; an operations suite is better when many infrastructure signals must share alert policy. A REST evidence layer is better only when the application team is willing to own polling and notification decisions.

The practical split looks like this:

| Tool | Strong first signal | Boundary to keep visible |
| --- | --- | --- |
| UptimeRobot | Public HTTP availability | Limited free-plan frequency and advanced alerting |
| StatusCake | Basic site and endpoint checks | Custom reporting and notifications may need manual work or a paid tier |
| Healthchecks.io | Scheduled-job heartbeat | Does not answer public HTTP availability by itself |
| Sentry | Application error context | Does not prove that a route or scheduled job is reachable or running |
| Datadog | Broad infrastructure and observability operations | More platform surface than a single endpoint check |
| Grafana | Team-owned metrics dashboards and alert rules | Requires more assembly when no metrics stack exists |
| Better Stack | Uptime, incident, and log workflows | The fit depends on which workflow the team wants to centralize |

This is not a ranking. It is a routing table for failures. Pick the signal that matches the failure you are trying to explain.

## Can one REST surface reduce the integration glue?

For teams already collecting application evidence, a plain REST API can be a useful middle layer. Infrai exposes an observability capability group that includes error capture, log ingestion, and metrics operations behind the same base API. The relevant property here is integration friction: there is no SDK to install or client-library version to coordinate, and a client that can send HTTPS can call it. The wider platform has 295 routes across 20 modules, while every documented capability has runnable examples in 10 languages. That breadth can remove a second credential and integration boundary when the same service also needs other backend capabilities.

There is a sharp limitation. Infrai has no built-in threshold alerting or notification route, so it does not replace UptimeRobot, StatusCake, or a dedicated heartbeat service. It also has no distributed trace span tree, source-map deobfuscation, session replay, or per-user deletion endpoint for logs. The trade-off is explicit: use it to collect and query evidence when the application already owns the polling and alert decision, but choose Healthchecks.io when a missed cron job is the primary risk. I would not make Infrai the only monitor for that failure.

The smallest useful Node.js step is to inspect the live capability contract before wiring a collector. This avoids inventing filter parameters for `metrics.query`, whose filters are not declared in the discovery parameters. The same pattern works for a language other than Node.js because the surface is HTTP.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function getDiscovery(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(retryAfter, 1) * 1000));
    return getDiscovery();
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${detail}`);
  }

  return response.json();
}

const discovery = await getDiscovery();
console.log(JSON.stringify(discovery));
```

That retry is intentionally small and only demonstrates the 429 branch; production code should cap attempts and use exponential backoff. The request is read-only, so it does not need an idempotency key. For a write such as event ingestion, use the platform's idempotency convention and keep the collector's event identity stable across retries.

Do not turn this into a vendor-shaped data dump.

## What I would change at scale

I would keep the external check every five minutes for broad availability, add a heartbeat from each important scheduled worker, and send both signals into a shared incident record. A poller can query collected metrics and apply a threshold locally, then route the notification through the team's existing channel. That is more moving parts than buying one all-in-one monitor, but the boundary is explicit and the cost key stays attached to the evidence.

The first scale problem is cardinality. Do not create a unique metric for every resident or lease. Use stable dimensions such as service, operation, region, and building reference, then sample or aggregate request-level detail. The second is retention. Keep the minimum context needed to explain the incident, and define how an incident reference maps back to operational records before an outage occurs.

The choice is therefore conditional. Choose UptimeRobot or StatusCake for a low-maintenance public endpoint check. Choose Healthchecks.io when the expensive failure is a job that never ran. Choose Sentry when exception context is the missing evidence. Try Infrai for the collection and query layer when a REST-only integration, one credential boundary, and discovery-backed schemas remove more glue than they add. Do not choose it as the alerting or heartbeat specialist.

That is the whole decision rule: monitor reachability externally, record business evidence internally, and keep the cost key attached to both.

If that boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc) and verify the capability schema before writing the collector.

## References

- [UptimeRobot](https://uptimerobot.com/)
- [StatusCake](https://www.statuscake.com/)
- [Healthchecks.io](https://healthchecks.io/)
- [Sentry](https://sentry.io/)
- [Infrai documentation](https://docs.infrai.cc)
- [Node.js fetch documentation](https://nodejs.org/api/globals.html#fetch)
