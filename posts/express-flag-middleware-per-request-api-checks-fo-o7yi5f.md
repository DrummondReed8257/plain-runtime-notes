# Express Flag Middleware: Per-Request API Checks for Import Silence

A per-request flag check buys precise control, but a remote lookup on every call turns the control plane into part of the request path. For a B2B SaaS import API, use a locally evaluated snapshot, fail closed on the guarded operation, and emit separate signals for flag denial, import acceptance, and missing results. **TL;DR:** keep authorization and flag evaluation distinct; attach one stable decision to the request; then alert on expected results that never appear, not on every rejected call.

| Choice | Signal quality | Request-path risk | Best fit |
|---|---|---|---|
| Local snapshot, checked per request | High with bounded fields | No evaluation network call | Default for tenant gates |
| Remote check per request | Current central decision | Adds a request dependency | Rules that cannot run locally |
| Process-wide boolean | Poor tenant diagnosis | Small runtime surface | Emergency global stop |

**Recommendation:** evaluate each request against an asynchronously refreshed local snapshot. Record the boolean and a non-secret rule revision once. Do not count guard rejection as import failure. Remote evaluation is the runner-up when central rules depend on context that cannot be distributed safely.

## Which event should wake someone up?

The tempting alert is `feature disabled` or `request rejected`. That is usually noise. A disabled import can be deliberate, and a rejected manual retry says little about the scheduled job. The operational question is narrower: did an import that was expected to run stop producing results?

Model the lifecycle. A scheduler creates an expectation. The guarded API accepts or denies the start. An accepted run later produces a result, or its deadline expires. Mixing those events into one error counter makes a planned flag change resemble an outage.

Use bounded dimensions such as `source_type`, `schedule_class`, and `decision`. Avoid tenant IDs, job IDs, URLs, and exception messages in metric labels. Prometheus naming guidance recommends a common prefix, base units, and names whose aggregate still makes sense. Counters named `imports_guard_decisions_total` and `imports_results_total` retain meaning when their dimensions are summed.

Logs carry identifiers for investigation. Metrics carry dimensions for detection. A warning can describe an expected run whose deadline elapsed; an informational record can describe flag denial. RFC 5424 defines severity levels independently from application actions, so severity should communicate operational meaning rather than substitute for alert policy.

One event deserves a page: an enabled schedule produced no result inside its declared window. A disabled schedule deserves an audit trail. Evaluator failure deserves its own health signal.

Three paths. Three meanings.

## Two criteria decide the guard

First, demand **decision consistency within one request**. Evaluate once near the route boundary and put the result on `res.locals`. If handlers ask repeatedly, a snapshot refresh can make one request observe two revisions. It also adds glue to every service call. Bad DX, muddy telemetry.

The context should contain only inputs the rule needs. Here that is a tenant key and operation. Authentication must already have established the tenant; a feature flag is not authorization. If a caller can choose the tenant context, the guard is decorative.

Second, classify failure explicitly. A missing snapshot, an explicit false result, and an evaluator exception are different. Starting an import changes state, so this example denies the operation when no trustworthy decision exists. A read-only status route could reasonably serve stale data. Make fallback visible in code and telemetry.

I benchmark middleware at the boundary because averages conceal the tail, but there is no universal latency number worth inventing. Measure the complete guarded route in your environment: warm local evaluation, refresh overlap, malformed context, and evaluator failure. Keep duration distributions outside flag labels. Durations, revisions, and request IDs create dimensions that grow without helping detection.

Config bloat is another warning. If each route declares a key, fallback mode, log template, metric prefix, and error shape, the abstraction has failed. Put policy in a small registry and let middleware accept a typed operation.

Less surface.

## How should Express middleware check a feature flag per request?

This example uses an interface rather than a vendor SDK. The evaluator reads an in-memory snapshot refreshed outside the request path. `begin_import` is the internal policy key; the response does not reveal it.

```ts
import type { NextFunction, Request, Response } from "express";

type Operation = "begin_import";
type Decision = { enabled: boolean; revision: string };

interface FlagEvaluator {
  evaluate(operation: Operation, context: { tenantKey: string }): Decision;
}

interface Metrics {
  increment(
    name: string,
    labels: Record<string, "enabled" | "disabled" | "error">,
  ): void;
}

type GuardLocals = { flagDecision?: Decision };

export function requireOperation(
  operation: Operation,
  evaluator: FlagEvaluator,
  metrics: Metrics,
) {
  return (
    req: Request,
    res: Response<unknown, GuardLocals>,
    next: NextFunction,
  ): void => {
    const tenantKey = req.authenticatedTenantKey;

    try {
      const decision = evaluator.evaluate(operation, { tenantKey });
      res.locals.flagDecision = decision;
      metrics.increment("imports_guard_decisions_total", {
        decision: decision.enabled ? "enabled" : "disabled",
      });

      if (!decision.enabled) {
        res.status(404).json({ error: "Not found" });
        return;
      }
      next();
    } catch (error: unknown) {
      metrics.increment("imports_guard_decisions_total", { decision: "error" });
      req.log.error({ error, operation }, "flag evaluation failed");
      res.status(503).json({ error: "Temporarily unavailable" });
    }
  };
}
```

Application-specific type augmentation supplies `authenticatedTenantKey` and `log`. That dependency is intentional: identity and structured logging should exist before the flag layer. Never recover a tenant key from an arbitrary query parameter.

Wire the guard only to the state-changing route. Monitor results downstream.

```ts
app.post(
  "/imports",
  requireOperation("begin_import", flagEvaluator, metrics),
  async (req, res, next) => {
    try {
      const run = await importQueue.enqueue({
        tenantKey: req.authenticatedTenantKey,
        sourceType: req.body.sourceType,
      });
      res.status(202).json({ runId: run.id });
    } catch (error: unknown) {
      next(error);
    }
  },
);
```

The `202` means accepted, not completed. Result monitoring cannot infer success from HTTP traffic. It needs an expectation record with a deadline and a completion record written by the worker. Alert evaluation compares them. This catches the silent case: the route worked, the queue accepted the job, and no result arrived.

A decision log can include operation, boolean result, rule revision, authenticated tenant key, correlation ID, and severity. Do not log the full evaluation context by default; rule attributes can contain customer data. Redact at the logging boundary.

## Test the silence, not only the middleware

A route test expecting `404` for false is necessary and insufficient. The useful suite crosses the boundary between control and observation.

Test four states: enabled and accepted; disabled and denied; evaluator failure and denied; accepted but no result before the deadline. Then test non-alerts: an intentionally disabled schedule, a completed run, and a run still inside its window. Signal quality is won here.

Use a fake clock for deadlines and a deterministic evaluator for route tests. Avoid waiting real minutes or calling a shared service from CI. Assert label values too. One accidental `tenantKey` label can pass functional tests while damaging the metrics path.

Deployment needs two controls with different owners. The feature decision controls whether a tenant may start imports. Alert policy controls how long an expected result may be absent and which schedules are actionable. Coupling them lets a rollout silently rewrite incident policy. Keep their schemas and change histories separate.

Watch the enablement boundary. Existing schedules may lack expectation records. Define whether the first window begins at activation or the next scheduler tick, then test it. Ambiguity produces false missing-result alerts immediately after rollout.

## When does remote evaluation win?

The main limitation of local evaluation is distribution: it is not suitable when rules require authoritative, rapidly changing context that cannot be copied safely into each process. That trade-off makes remote evaluation the better fit when the request can afford the dependency. Bound the call with a deadline. Specify fallback for this operation. Instrument evaluator errors separately from disabled results.

It also fits when consistent central decisions matter more than request-path independence. The cost is architectural: retries, connection limits, regional failure, and cache semantics join route behavior. Benchmark those states, including failure, before adopting it. A happy-path median answers the least interesting question.

A process-wide boolean remains useful as an emergency stop. It is a poor substitute for tenant rollout because it cannot explain eligibility and collapses planned denial with system failure.

Preserve one decision per request, but alert on the business result after acceptance. That separation gives operators a quiet signal and developers a guard they can reason about without dragging configuration through every handler.

## Sources

- https://prometheus.io/docs/practices/naming/
- https://datatracker.ietf.org/doc/html/rfc5424
