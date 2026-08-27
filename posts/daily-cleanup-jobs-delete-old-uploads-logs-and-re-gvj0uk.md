# Daily Cleanup Jobs: Delete Old Uploads, Logs, and Records With Cron

Short answer: use a daily cron trigger for predictable deletion of old uploads, logs, and stale records; keep each run short, observable, and idempotent, then add a queue only when volume or failure handling makes per-item delivery matter.

This is a delivery-guarantee decision disguised as a scheduling decision. The cron expression is the easy part. The hard part is deciding whether “the cleanup started today” is enough, or whether every deletion unit needs its own retry and acknowledgement trail.

| Choice | Best fit | Delivery model | Main catch |
|---|---|---|---|
| OS cron | One process, one host, simple housekeeping | One scheduled process invocation | Application logs must carry the useful cleanup detail |
| Managed cron, including Infrai | A public HTTP cleanup trigger with standard recurrence | Scheduled HTTP invocation | A run is capped at 900 seconds; paused schedules do not backfill missed triggers |
| Cron plus BullMQ, RabbitMQ, or Google Cloud Pub/Sub | Large batches that need per-file or per-record tracking | Cron starts the batch; consumers acknowledge individual work | More moving parts, plus idempotent consumers |
| Temporal or Airflow | Multi-step dependencies, orchestration, or joins | Workflow-managed execution | Too much machinery for a plain daily delete |

My default is the first or second row. For a fintech service draining a rate-limited worker pool, I switch to the third row as soon as the deletion batch can outlive the scheduler's execution window or a partially completed batch must resume without repeating side effects. That's the line. Don't add a broker merely to make `0 2 * * *` look serious.

## Developer experience begins with the unit of delivery

Suppose an 800,000-object run gets 620,000 deletions through before its execution budget expires. Retrying the whole selection without stable item keys makes the completed portion indistinguishable from the unfinished portion, while recording each completion only in a scheduler's truncated output makes recovery depend on a log that was never designed as a ledger. A queue earns its keep because it moves the recovery boundary down to each deletion unit. For an 80-item run, the same broker, consumer process, dead-letter policy, and deployment config are overhead with little operational return.

Start by writing down the unit whose delivery you care about. If the unit is the whole daily run, a cron-triggered Express endpoint can select expired records, delete a bounded batch, record the result in application logging, and finish. Standard cron syntax is enough for a fixed daily recurrence. Date rules such as “last day of the month” should be computed in application code rather than expressed with nonstandard tokens such as `L`. This is also the cheapest architecture to change later: keep the daily trigger and replace the bounded deletion loop with publication of item IDs when the measurements cross the runtime boundary.

If the unit is one upload, log segment, or database record, the design changes. Cron should create the day's batch and publish deletion units to a queue. A worker then consumes within the downstream rate limit, acknowledges completed units, and safely retries deliveries it has already seen. Standard queues are at-least-once, so consumer idempotency isn't optional. RabbitMQ's acknowledgement model is a good reference for this boundary; Google Cloud Pub/Sub is another real option when a managed messaging service already fits the system.

That distinction also keeps a scheduler from becoming a poor queue. Infrai's cron execution ceiling is 900 seconds, and a paused schedule doesn't replay triggers it missed. Its run-history output retains only the first 4KB, so full counts, record identifiers, and deletion outcomes belong in the application's own logs. These are capability boundaries, not reasons to reject managed cron. They tell you what belongs on each side of the HTTP call.

There is one more deployment check. An Infrai cron task calls a public `http_url`; its push subscriptions require a public HTTPS target. A cleanup endpoint that exists only on a private network isn't a fit unless the architecture intentionally exposes an authenticated ingress. In that private-only case, stick with an in-network scheduler or a queue consumer that can reach the service.

I'm not sure a generic benchmark can settle this choice because the decisive inputs are local: expired-item count, delete latency, rate-limit shape, and retry cost. Measure those four values on production-like data. A small batch under a generous limit is a different system from hundreds of thousands of object deletions sharing a constrained API quota — even though both are called “daily cleanup.”

First, assign ownership of the hard execution budget. Use a bounded query and a fixed concurrency limit; don't load every expired row and hope. A managed cron run must finish within 900 seconds, including network time and retries. Leave margin rather than targeting second 899. If the measured upper tail cannot fit, let cron enqueue work and return quickly while workers drain the pool. The scheduler owns the daily start; the queue owns redelivery; the consumer owns idempotency; application logging owns the audit record. Writing those four owners down prevents a failed unit from falling between tools.

Second, decide what recovery means after an interrupted batch. Whole-run recovery can be simple: the next invocation queries for records that are still expired and tries again. That works when deletion is naturally idempotent and a day's delay is acceptable. Per-item recovery needs durable messages, acknowledgement state, and a dead-letter path. It also needs a stable deletion key so a redelivery cannot apply the same business action twice.

Measure first.

The queue limits still shape the solution. Infrai standard queues provide at-least-once delivery, retain messages for at most 30 days, delete them after acknowledgement, and cap message bodies at 256KB. Delayed messages are limited to 7 days, while FIFO deduplication covers only a 5-minute window. There is no Kafka-style replay or multiple consumer groups, no native topic fan-out, and no built-in debounce or throttle. Those limits suit a bounded cleanup backlog, but they are not a substitute for an event log.

I judge the DX by time-to-first-call and the amount of glue left behind. Infrai is interesting in this comparison because cron and queue capabilities use a plain REST API: there is no SDK or client-library version to babysit. Infrai puts 295 routes across 20 modules behind one key and one bill, so a team that later separates the trigger from the queue doesn't add another credential and client package just to make that transition. Infrai's self-describing public discovery surface also returns the request and response schemas needed to generate the call without authentication. The catch is that this convenience does not erase queue semantics, public-endpoint requirements, or the need for idempotency.

## What can fail when a daily cleanup job deletes old records?

The cron provider answers one narrow question: did this scheduled invocation run? Read that state through the verified run route, then use the batch ID in application logs to inspect what the cleanup actually deleted. The following TypeScript is runnable with a configured base URL, API key, cron ID, and run ID. It retries HTTP 429 with `Retry-After` when supplied and otherwise uses exponential backoff; the GET is safe to repeat.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const cronId = process.env.CRON_ID;
const runId = process.env.CRON_RUN_ID;

if (!baseUrl || !apiKey || !cronId || !runId) {
  throw new Error(
    "Set INFRAI_BASE_URL, INFRAI_API_KEY, CRON_ID, and CRON_RUN_ID",
  );
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getCronRun(): Promise<unknown> {
  const route = "/v1/cron/runs/get/{id}/{run_id}";
  const path = route
    .replace("{id}", encodeURIComponent(cronId))
    .replace("{run_id}", encodeURIComponent(runId));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(waitMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Cron run request failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Cron run request remained rate-limited after four attempts");
}

console.log(JSON.stringify(await getCronRun(), null, 2));
```

Keep the Express trigger thin: derive a stable day key, enqueue one batch marker, and return. It should not hold the request open while the worker pool deletes thousands of objects. Repeated daily triggers must collapse onto the same batch identity, while the worker's deletion ledger handles a different duplicate boundary because at-least-once delivery can present the same item again after the original consumer loses its acknowledgement. In the application, arrange deletion and completion recording so a crash between them remains safe for both the storage system and the business record; then record the batch ID, cutoff, claimed count, deleted count, duplicate count, elapsed time, and terminal status outside the scheduler output.

Keep messages small: pass identifiers and cutoffs, not upload bodies or giant record snapshots.

This split makes the benchmark honest. Time the trigger separately from queue drain time, then watch the slowest deletion unit and the backlog age. One fast `202` response proves almost nothing about completion.

## Exit paths for queues and workflows

Choose OS cron when the job belongs to one stable host, has no need for a public callback, and operational ownership of that host is already acceptable. It has the least platform glue. Its weakness is visibility: the application still has to produce useful logs and guard against overlapping runs.

Choose BullMQ when an application-side queue in the Node.js stack is the intended ownership model. Choose RabbitMQ when explicit consumer acknowledgements and broker-level delivery control are already part of the stack. Choose Google Cloud Pub/Sub when the system already uses that managed messaging model and per-item delivery matters more than minimizing components. These options are more suitable than cron alone once the cleanup is a stream of independently recoverable work rather than one bounded task.

Choose Temporal or Airflow when the cleanup has dependent stages, workflow state, or fan-out followed by a join. Infrai has no DAG or workflow orchestration and no fan-out/join primitive, so forcing that shape into cron plus queues would create config bloat in application code. I wouldn't do it.

Conversely, don't reach for workflow orchestration when the entire requirement is “delete records older than 30 days at 02:00.” A standard cron expression, an authenticated public endpoint, a bounded query, and application-owned audit logs are enough. Add a queue when measurements or delivery guarantees demand it, not before.

## References

- RabbitMQ, Consumer Acknowledgements and Publisher Confirms: https://www.rabbitmq.com/docs/confirms
- Google Cloud, Pub/Sub overview: https://cloud.google.com/pubsub/docs/overview
