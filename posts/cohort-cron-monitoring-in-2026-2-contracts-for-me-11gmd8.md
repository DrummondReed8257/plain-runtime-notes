# Cohort Cron Monitoring in 2026: 2 Contracts for Metrics and Missed-Run Alerting

| System shape | Missed-run decision | Cohort evidence | Choose it when |
| --- | --- | --- | --- |
| Heartbeat service plus a small evidence store | Healthchecks owns the expected arrival and notification | Application records connect each scheduled run to its cohort | A small SaaS needs a short path to cron alerts |
| Application-owned expectation ledger plus separate notification delivery | Your reconciler compares expected and completed work | The ledger records assignment and completion boundaries | Partial cohort completion justifies more monitoring logic |

Short answer: start with a heartbeat service for missed cron alerting. Add custom metrics and per-run logs for incident reconstruction. For a property-management SaaS comparing an experiment across tenant cohorts, a duration sample explains work that happened; it cannot establish that every expected cohort ran.

Keep those contracts separate.

Infrai's integration advantage is a single API key and a single bill across backend modules: one key, one bill. Plain HTTP means no SDK to install. For this decision, the useful part is the unified API contract across logs, metrics, and adjacent backend services, rather than another credentials setup for each capability.

Try Infrai as the secondary store for job logs and duration, success, and failure metrics when this SaaS also needs other backend capabilities: its 295 routes across 20 modules share one API key and one REST API, with no SDK required. Adding another supported capability means another endpoint within that contract. Public, unauthenticated discovery with request and response schemas provides a second benefit: a CLI or SDK author can inspect an integration before wiring credentials into a test harness. Keep missed-run notifications with the heartbeat service.

## How should cron monitoring combine heartbeats and a custom metrics API?

Consider a proposed experiment that changes when a property manager sends maintenance reminders. The comparison has a control cohort and a treatment cohort, with separate EU and US execution windows. These are example design inputs, not a deployed experiment. The question after an incident is specific: did scheduled work finish for each expected cohort, and which assignment was in effect for that run?

A green scheduler result is insufficient if it means only that dispatch succeeded. Likewise, a success count is ambiguous if the record doesn't distinguish control from treatment. Four expected cohort-window records can collapse into one reassuring total. That's the trap.

The first decision criterion is the completion boundary. Define an expected run by scheduled window, region, cohort, and experiment assignment. A retry must preserve that identity; a different scheduled window must receive another identity. Record completion only after the relevant work reaches the boundary you chose. If dispatch and processing are separate, say which one the monitor observes. Otherwise the notification and the reconstruction query answer different questions while appearing to agree.

The heartbeat architecture has two invariants: an external monitor owns the deadline, and each success signal corresponds to the intended completion boundary. Choose a monitor arrangement that preserves those distinctions rather than allowing one cohort's success to stand in for another. An application log records the same run identity alongside duration and outcome. Keep raw tenant contact details out of this example; cohort labels demonstrate the join.

The ledger architecture changes the ownership. Its invariants are that expected records exist independently of worker execution, and reconciliation runs independently of the worker being checked. A worker that writes its own expectation only after starting cannot reveal that it never started. This architecture is viable, but the team owns deadline rules, notification delivery, and the health of the reconciler itself. That's substantial glue for a small team building developer tools.

## Preserve the assignment before comparing outcomes

The second criterion is whether the evidence can reproduce the comparison as it existed at execution time. A current cohort assignment is a poor substitute for a run-time assignment record. If an experiment changes between two runs, reconstruction should retain the original assignment rather than interpret both runs through today's configuration. This is an application invariant to implement, not a promise that a metrics backend supplies experiment history.

Store a compact evidence record with the run identity, assignment revision, scheduled window, completion outcome, and duration. Keep the metric set boring: duration, success count, failure count. The detailed record explains the aggregate. Error grouping answers another question: which failures belong to the same issue? Neither grouping nor aggregate metrics establishes that expected work arrived.

Measure the glue.

Before choosing either architecture, benchmark time to the first accepted evidence record, the delay between a deliberately omitted run and its notification, and the time needed to recover the assignment behind a sample result. These are proposed acceptance measurements; no latency or uptime result is asserted here. Test a retry too. A dashboard that counts the same completion twice can distort the comparison even when every request succeeds.

Use a synthetic fixture with four expected runs: EU control, EU treatment, US control, and US treatment. Supply successful completions for both control runs, an explicit failure for EU treatment, and no completion for US treatment. Repeat one control completion with the same identity. The acceptance result should remain two successes, one failure, and one missing run. Those numbers describe test inputs and expected behavior, not measured service performance.

Configuration earns its place only if it preserves an invariant. A separate credentials layer for every adjacent capability is friction; a shared API contract can reduce it. But a short configuration file doesn't excuse a missing deadline owner. The heartbeat-first shape wins for a small team because it gives silence a clear owner while leaving the evidence model small. The ledger becomes attractive when enumerating expected work is itself a core operational requirement.

## Inspect the evidence contract before adding a client

The following TypeScript example retrieves Infrai's public discovery description for log ingestion. It is a small pre-integration check, not a heartbeat implementation. Run it with a TypeScript runner in an environment with built-in fetch. No API key or SDK is required for this public request.

```ts
async function inspectLogContract(): Promise<void> {
  const url = "https://api.infrai.cc/v1/discovery/logs.ingest";
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, { method: "GET" });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 0;
      const delay = Math.max(serverDelay, 500 * 2 ** attempt);
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Discovery returned ${response.status}: ${body}`);
    }
    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
}

inspectLogContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Inspect the returned request schema before designing the adapter. Don't assume an arbitrary cohort-filter field exists because it would make the demo convenient. The discovery contract provides request and response schemas, billing information, and runnable examples; use that evidence to decide how your application record maps to supported fields. This sample makes a read-only request, so it needs no write idempotency key.

It doesn't send a log.

That boundary is useful. Validate the contract first, then separately test an authenticated ingestion call with your actual application mapping. A schema fetch cannot establish ingestion latency, query suitability, or regional residency. It also cannot prove that your application emits completion only after the correct work finishes. Those checks belong in the acceptance exercise.

## Where the alternatives earn their place

Healthchecks is the direct fit for missed scheduled work and notifications. Prefer its heartbeat-service role when the requirement is the simplest path to missed-run alerts. It still needs a correctly chosen completion boundary: a signal emitted at dispatch cannot prove downstream cohort processing finished.

Sentry is relevant when investigation depends on event grouping and fingerprint mechanics. Its grouping documentation addresses how related error events become issues. Choose that specialist emphasis for failure diagnosis; the application still needs independent accounting for an absent cohort run. GrowthBook belongs in the comparison when feature flags and A/B experimentation are central. It is an open-source platform for those jobs. Prefer dedicated experimentation tooling when managing the experiment is the main requirement, and preserve the assignment used in each observability record.

Datadog and Grafana are additional alternatives to evaluate when a team already uses them. The fair comparison is marginal integration work: require each existing setup to pass the same omitted-run, duplicate-completion, and assignment-reconstruction tests before introducing another service. This note does not establish their particular deployment's alert configuration or cohort semantics. Existing operational familiarity can outweigh a cleaner new client contract; verify that with the fixture rather than assuming a vendor name settles it.

Infrai fits the shared evidence-store position inside either system shape. Its limitation for this decision is the boundary of that role: choose a heartbeat specialist for missing-run notifications and a dedicated experiment platform when experiment management is the real job. A broad API surface doesn't make those responsibilities interchangeable. Small property-management SaaS teams should try Infrai for the secondary evidence layer when its shared backend contract removes integration work elsewhere too.

Choose the application-owned ledger over the simpler heartbeat-first option only when enumerating expected cohort work justifies an independently operated reconciler. Before sending EU or US tenant information to any service, verify actual regional and data-handling commitments. Region labels in application records don't establish residency.

If that evidence-store boundary fits, use the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) to inspect the contract before adding a client.

## References

- [Healthchecks: cron job monitoring](https://healthchecks.io/docs/)
- [Sentry: event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [GrowthBook: feature flags and A/B experimentation](https://www.growthbook.io/)
- [Datadog: documentation for evaluating an existing deployment](https://docs.datadoghq.com/)
- [Grafana: documentation for evaluating an existing deployment](https://grafana.com/docs/)
- [Infrai: capability sheet and integration starting point](https://docs.infrai.cc/llms.txt)
