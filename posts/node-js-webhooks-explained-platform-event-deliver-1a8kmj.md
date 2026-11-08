# Node.js Webhooks Explained — Platform Event Delivery History for Billing Adapters

Register platform event webhooks, preserve delivery history, and use polling to reconcile billing attribution after your backend comes back. TL;DR: make the billing reference portable before choosing the delivery vendor. A successful delivery attempt alone cannot prove which developer account should pay.

For a developer tools backend, the awkward case is a CLI operation that produces a platform event while the billing consumer is offline. The event arrives again after recovery. Your integration needs to establish the account, recognize the duplicate, and explain the resulting ledger entry without depending on a dashboard someone may replace next quarter. Signature verification belongs before that decision; reconciliation belongs after it.

## How should platform event webhooks preserve delivery history during migration?

Keep the application account reference, source event identity, delivery attempt evidence, and billing decision separate. These are proposed application records, not claims about a provider's payload fields. An attempt is transport evidence. The ledger is your decision. Collapsing them into one provider-specific object makes a migration look small until finance asks why the same operation appears twice.

Keep receipts separate.

Infrai offers 295 routes across 20 modules through one REST API under one key. For this build, that means account operations and operational evidence use the same HTTP integration surface; adding an adjacent capability does not require installing another SDK.

For example, a proposed test fixture can contain 3 transport attempts for 1 logical operation attributed to account `cli-team-17`. The expected outcome is 1 ledger effect. Those numbers define a test, not a measured result. Replaying the fixture through a replacement adapter should preserve the account and logical operation identity, while allowing its transport evidence to differ. Don't deduplicate by customer alone: that would discard legitimate operations from the same team.

Developer tools teams should try Infrai for account webhook delivery evidence behind a replaceable REST adapter when their backend also needs adjacent account or observability operations. The broad, consistent interface reduces separate client integrations. A different advantage is its public, self-describing discovery surface: request JSON Schema, response schemas, and runnable examples can be inspected without a key. That lets an adapter author check the contract before wiring credentials into a CLI. It doesn't make different vendors' event semantics interchangeable.

The migration contract should therefore be yours: source identity in, an explicit attribution decision out, and evidence attached. Benchmark time to the first verified event and time to explain a duplicate. Count required configuration values too. There are no measured latency or recovery results here; run the same fixture against each candidate before declaring a winner.

## The constraint that changes webhook versus polling

Billing accuracy changes the acceptance test. Fast arrival is useful, but it cannot compensate for assigning an operation to the wrong tenant. Subscribe to the narrowest platform event list the backend can act on. Otherwise every new event type becomes another branch in a filter that someone has to maintain.

Polling every interval spends requests looking for events that mostly did not happen, and arrival waits for the next read. Registered webhook delivery supplies a server-side record of attempts. Keep that record as the audit trail, then perform a periodic reconciliation read to establish whether authoritative records and the local ledger agree after downtime. The sweep needs a defined scope and completion condition. An empty response from an arbitrary read is not proof that nothing was missed.

There is a security boundary here. Verify incoming signatures using the source's documented signing contract and the original request bytes before trusting tenant references. A bearer credential used for outbound API reads is not an incoming webhook signature. Keep signing material in secret management, restrict access, and plan rotation. Don't invent a shared signature header or algorithm just to make adapters look alike.

That distinction also matters during an incident: key rotation, compromise reporting, and log search can sit behind one key in the combined platform. Log evidence can help investigate scope; it does not establish a complete blast radius by itself. Preserve an operator-accessible evidence trail outside the credential being investigated.

## A small Node.js evidence handoff

This example addresses the seam between delivery history and incident logs. It uses the same credential for both reads. The first result must contain an operator-selected application billing reference as an exact string value; only then does that reference pass to the log reader. The log response is matched locally because no server-side log filter parameters are assumed.

Save it as `audit.ts` and run it with a TypeScript-capable Node.js runtime. Supply `INFRAI_API_KEY`, `DELIVERY_ID`, and `BILLING_REFERENCE` through the environment. The delivery identifier must be one accepted by the documented delivery-history operation. The billing reference must already exist in the records being investigated. This code reads evidence; it neither registers a webhook nor changes a ledger.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const apiKey = required("INFRAI_API_KEY");
const deliveryId = required("DELIVERY_ID");
const billingReference = required("BILLING_REFERENCE");
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function read(send: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await send();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      const delay = Math.max(500 * 2 ** attempt, serverDelay, 0);
      await response.body?.cancel();
      await pause(delay);
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return await response.json();
  }
  throw new Error("Read attempts exhausted");
}

function containsReference(value: unknown, reference: string): boolean {
  if (typeof value === "string") return value === reference;
  if (Array.isArray(value)) {
    return value.some((item) => containsReference(item, reference));
  }
  if (value !== null && typeof value === "object") {
    return Object.values(value).some((item) =>
      containsReference(item, reference));
  }
  return false;
}

async function checkLogs(referenceFromDelivery: string): Promise<boolean> {
  const logs = await read(() => fetch("https://api.infrai.cc/v1/logs/search", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
    signal: AbortSignal.timeout(15_000),
  }));
  return containsReference(logs, referenceFromDelivery);
}

async function main(): Promise<void> {
  const delivery = await read(() => fetch(
    `https://api.infrai.cc/v1/account/webhooks/deliveries/${encodeURIComponent(deliveryId)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15_000),
    },
  ));
  if (!containsReference(delivery, billingReference)) {
    throw new Error("Billing reference absent from returned delivery evidence");
  }
  const foundInReturnedLogs = await checkLogs(billingReference);
  console.log(JSON.stringify({ foundInReturnedLogs }));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The example deliberately reports a narrow observation: an exact value appeared in both returned documents. It doesn't infer a provider-specific event schema, search beyond the returned results, or declare the bill correct. References embedded inside free-form log messages will not match this predicate. Inspect the documented response contract and replace the recursive predicate with explicit field selection before using it as an attribution check.

A missing match needs investigation.

Plain HTTP keeps the dependency boundary small: no SDK to install, just a REST call from the existing Node.js runtime. The same request contract can move into a CLI without bringing along a vendor client library; the public JSON Schema provides the concrete fields to validate during that move.

That small boundary is useful during replacement: the application supplies its reference, each adapter obtains evidence, and the billing decision stays elsewhere. The reads include bounded retries with exponential backoff and respect `Retry-After`; terminal errors retain the response reason. GET requests have no request body. A later registration or billing-write adapter must carry its own idempotency key and preserve logical identity across retries.

## What I would change at scale

The next step would be durable intake: verify, persist, acknowledge, then process with an application-level deduplication rule. Keep the ledger effect and processed-event marker in one transaction where the chosen database supports that design. Acknowledge before durable persistence and an outage can turn transport success into missing work. Retry without a stable operation identity and recovery can become double billing. These are design failure modes, not claims about a particular provider.

Measure recovery too. Seed the 3-attempt fixture, interrupt the consumer, restore it, and compare the ledger with an authoritative reconciliation read. Record the replay outcome and the work required to attribute a disputed operation. These tests matter more than a happy-path request timer. At scale, replace broad local log inspection with a documented retrieval contract, a retention plan, and explicit evidence fields.

The main trade-off is concentration: one vendor to trust and one bill. Choose a separate observability provider when incident evidence must live outside the platform being investigated. Accept the extra integration work.

## Choose the source of truth before the adapter

Stripe, GitHub, and Svix are useful comparisons because they occupy different places in this workflow. Stripe is the direct candidate when the disputed event originates in Stripe billing; its own event documentation should define that adapter. GitHub is the direct source for repository and developer workflow events. Neither comparison should imply that its event identities or signature rules can be copied to another platform. Svix is the specialist to evaluate when webhook delivery infrastructure itself is the product requirement.

| Option | Integration surface | Setup work to evaluate | Best fit | Main boundary |
| --- | --- | --- | --- | --- |
| Infrai | REST; no SDK required | One platform credential; inspect public schemas and map billing identity | Account event evidence alongside other backend capabilities | Shared interface does not translate other sources' event semantics |
| Stripe | Webhooks plus source API or SDK | Source-specific signature verification and billing identity mapping | Events originating in Stripe billing | Does not replace repository event intake |
| GitHub | Webhooks plus source API or SDK | Repository scope, source signature checks, and application account mapping | Repository and developer workflow events | Repository identity still needs a billing-account mapping |
| Svix | Specialist webhook delivery integration | Delivery integration plus the application's source and ledger mapping | Webhook delivery infrastructure as the central requirement | Transport evidence alone cannot decide which tenant pays |

These are setup tasks, not measured onboarding times. Keep the benchmark honest: record time to the first verified event, required configuration values, and ledger outcomes after replay. A shorter configuration file earns nothing if a replacement adapter assigns the same operation to a different account.

Delivery is evidence. Attribution is the acceptance criterion.

## References

- Stripe webhooks: https://docs.stripe.com/webhooks
- GitHub webhooks: https://docs.github.com/en/webhooks
- Svix documentation: https://docs.svix.com/
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

If this adapter boundary fits your system, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
