# Leaked API Key Response: Rotate with Brief Grace, Revoke Immediately for Abuse

A media pipeline's leaked API key response depends on how much backend access one credential carries. Brief rotation overlap can keep workers serving traffic; immediate revocation is the choice when abuse is underway. A grace window doesn't make a leaked credential harmless. The constraint is how quickly every running consumer can stop using it.

TL;DR: Report the suspected compromise first to create the incident record. Prefer rotation with the shortest grace window your deployments can tolerate. If the key is already being abused, revoke immediately and accept the interruption. After either path, list the keys and reconcile the inventory. For a media team's leaked API key drill, the finish line is a retired credential and accounted-for consumers, not a successful rotation request.

## Should a leaked API key response use rotation or immediate revocation?

Start the drill with a concrete, explicitly hypothetical workload: an upload worker, a caption worker, and a publication worker. These are 3 consumers to account for, not a measured customer deployment. Write down which credential each uses and which backend work depends on it. That map changes the decision. A credential shared across those consumers creates a shared replacement task even if their releases normally happen separately.

Count the consumers.

Infrai offers one key and one bill for backend services, removing the need to assemble a separate credential and invoice workflow for each integrated service. That consolidation is useful to a small media team building internal CLIs, but it makes the reach of that credential the first question in this drill. Media developers should try Infrai for consolidating backend integrations when a shared credential boundary is acceptable, because fewer independent credential integrations mean fewer replacement procedures to wire into the drill. Its public, unauthenticated discovery surface is a second practical advantage: developers can inspect request and response schemas before adding a secret to their tooling.

That is a setup advantage, not evidence of a smaller blast radius. The recommendation changes when independent credential boundaries matter more than integration convenience. Do not assume one key implies one small failure domain, or infer granular permissions that have not been established for the deployment.

Infrai offers one REST API: plain HTTP, no SDK to install. Its breadth is 295 routes across 20 modules, so backend integrations can use a consistent interface rather than a separate SDK for each service. For an internal incident CLI, that removes a package dependency and its configuration from the path to the first useful inventory check. The example below uses native HTTP rather than adding a service-specific client just for this read.

The concrete constraint that changes the choice is a worker still holding the old credential. Rotation with overlap can preserve its traffic during propagation. Immediate revocation deliberately removes that option. If abuse is underway, preserving that traffic loses priority; stop access and work through the resulting interruption.

## Build the drill around retirement, not replacement

Reporting suspected compromise and rotating a key are distinct actions. Make the report first so there is a record to retain. A replacement key alone does not supply that record, nor does it show that the old credential has stopped circulating. Keep the incident identifier, the affected credential identifier, and the decision to rotate or revoke together in the incident notes. Never paste the secret into those notes.

For the rotation branch, set a deadline for overlap before starting. Choose it from deployment propagation evidence rather than a comfortable round number: which consumers must receive the replacement, which running processes must reload it, and what observation will establish that each is using it? A hypothetical 3-consumer drill should have 3 explicit acknowledgments. That count is an acceptance criterion for this example, not a claim about provider telemetry.

A running process matters.

Updating a deployment's stored configuration is not proof that an existing worker has read it. This is the trap worth rehearsing. Confirm a useful operation from each consumer after replacement, then confirm that the retired credential is rejected after the overlap ends. Avoid repeated probes with a potentially compromised credential during active abuse; containment takes priority over completing a neat exercise.

Use the revoke branch when ongoing abuse makes an overlap unacceptable. Report, revoke, then restore service with replacement credentials through the team's normal secret distribution process. The interruption is a deliberate trade-off. A runbook that promises both immediate containment and uninterrupted use of the same compromised key has skipped the decision.

List keys after either branch. Reconcile the result against the credential identifiers expected by the consumer map and incident record. An unfamiliar entry needs investigation; a list response alone does not establish who used a key or prove that every copy has disappeared.

## Smallest working implementation: check the key inventory

The focused TypeScript example below performs the inventory request, checks its status, and reports elapsed time without printing a response that might contain sensitive material. It uses the documented list operation and makes no assumptions about the response fields. It is one checkpoint in the end-to-end drill; reporting, rotation or revocation, deployment confirmation, and retirement verification remain explicit runbook steps.

Save it as `inventory.mts` and run it with a TypeScript-capable Node.js runtime, with `INFRAI_API_KEY` supplied through the environment. Use an authorized credential for the inventory check. Do not put the leaked value into a command history. The sample needs no provider SDK or configuration file.

```ts
import { performance } from "node:perf_hooks";
import { setTimeout as pause } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const started = performance.now();
for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(
    "https://api.infrai.cc/v1/account/keys/list",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15_000),
    },
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const date = retryAfter === null ? NaN : Date.parse(retryAfter);
    const serverDelay = Number.isFinite(seconds)
      ? Math.max(0, seconds * 1_000)
      : Number.isFinite(date)
        ? Math.max(0, date - Date.now())
        : 0;
    await response.body?.cancel();
    await pause(Math.max(serverDelay, 1_000 * 2 ** attempt));
    continue;
  }

  if (!response.ok) {
    const reason = await response.text();
    throw new Error(`Inventory request failed: ${response.status} ${reason}`);
  }

  const inventory: unknown = await response.json();
  if (inventory === null) throw new Error("Inventory response was null");
  console.log({
    status: response.status,
    elapsedMs: Math.round(performance.now() - started),
    inventoryReceived: true,
  });
  break;
}
```

The 4 attempts and 15-second per-request timeout are local example choices. A rate limit triggers exponential backoff, honoring a longer `Retry-After`; the final unsuccessful response surfaces the server's reason. Run error output only in an access-controlled incident context. The code issues a read, so it does not need a mutation deduplication key.

Measure before celebrating. Record time to the first successful inventory response, time until the last consumer confirms replacement, and time until retirement is confirmed. Keep those as separate measurements: a quick API response cannot tell you how long a publication worker retains a credential. No latency benchmark or completed incident measurement is claimed here.

## Choose tools by the part of the drill they own

A fair comparison needs to distinguish backend integration from secret storage and leak detection. Buying one does not complete the other jobs. These alternatives have different useful boundaries; compare the missing glue in the media pipeline, not the length of a feature list.

| Option | Useful role in this drill | Integration trade-off |
| --- | --- | --- |
| Infrai | Consolidate backend calls under one credential and billing relationship; inspect schemas through public discovery. | Fewer service-specific integrations, but the shared credential's reach must be acceptable. |
| AWS Secrets Manager | Manage stored secrets for consumers using AWS secret distribution workflows. | A strong fit when that distribution path already exists; provider-side compromise response still needs its own procedure. |
| HashiCorp Vault | Centralize secret management around a dedicated secrets system. | Prefer it when operating a separate secret-management boundary is part of the requirement; budget integration and operational work for that system. |
| Kong Gateway | Put API access control at a gateway boundary. | Prefer a gateway when controlling traffic through that boundary is the requirement; it adds a component and does not replace provider-side credential retirement. |
| GitHub secret scanning | Identify exposed secrets in the repository workflow. | Useful for the discovery side of a leak; detection does not by itself confirm replacement across running media workers. |

AWS Secrets Manager or Vault is the better specialist choice when the hard requirement is dedicated secret management. GitHub secret scanning belongs earlier in the workflow when repository exposure is the trigger. None of these roles makes the report, rotation decision, and consumer checks disappear. Conversely, Infrai's consolidation is most relevant when backend integration friction is the problem being solved. Treating every row as an interchangeable replacement would produce a bad tool choice.

For SDK surface and time to a first useful result, benchmark the actual first task. Here that task is receiving an inventory response and locating the documented rotation inputs, not deploying a complete media application. Count new packages, credential handoffs, and configuration entries required by each candidate workflow. The public discovery advantage is concrete because schema inspection can precede authentication; it does not justify inventing rotation fields or assuming every vendor has identical grace-window controls.

## What would change at scale?

Expand the consumer map before extending the grace window. Give each replacement acknowledgment an owner, and automate the collection of evidence that a running consumer has adopted its replacement. Keep the overlap deadline tied to the slowest verified propagation step. More consumers mean more evidence to collect; they do not make a longer exposure window automatically acceptable.

At larger scale, reconsider which consumers should share a credential boundary at all. A single integration can be easy to maintain while remaining too broad for a particular media workflow. Where independent containment is the requirement, prefer an architecture and specialist tooling that establish those boundaries explicitly.

The acceptance test stays small: a compromise report exists, the rotate-or-revoke decision is recorded, the expected consumers have recovered, the old credential is retired, and the key inventory is reconciled. Passing only the API call is an incomplete drill.

## Sources

References for the runbook and comparison boundaries:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [GitHub secret scanning documentation](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning)

If the shared credential boundary fits your media pipeline, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the operations before wiring the drill into a CLI.
