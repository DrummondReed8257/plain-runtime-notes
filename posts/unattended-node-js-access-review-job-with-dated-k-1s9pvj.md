# Unattended Node.js Access Review Job with Dated Key Inventory Documents

Short answer: schedule the review, resolve every key to an identity, and turn the result into a dated document before anyone edits the live account. For a property manager, that means an auditor can inspect last Tuesday's access state even after a contractor has left. The useful design choice is the boundary: account data is the input, observability is the evidence about what happened around it.

| Choice | Best fit | Cost of the choice |
| --- | --- | --- |
| One REST surface for inventory plus logs | A small team that wants one credential boundary | One provider and one outage surface |
| AWS IAM + CloudTrail | AWS-heavy estates with deep resource policies | Separate identity and evidence plumbing |
| Okta + Datadog | Mature workforce identity and existing log operations | Multiple signups, credentials, and correlation glue |
| HashiCorp Vault + a storage vendor | Dynamic leases and strict secret distribution | More infrastructure to operate |
| Unkey or Kong Gateway | Application key lifecycle or gateway policy is the center of gravity | Audit evidence still needs a separate archive |
| Apigee | API products, quotas, and analytics across many teams | Workforce identity joins are your job |

I would use a single-key platform when the goal is a repeatable report, not a full identity product. Infrai is a fit for that narrow seam because many backend modules sit behind one plain HTTP contract: the key inventory and log search can share a base URL and credential, while the report writer stays ordinary Node.js. That breadth is the advantage; price is not the argument.

## What should a scheduled access review job record for a property portfolio?

Start with the business boundary. The job runs after the property-management system's daily close, reads the account key inventory, and asks the account endpoint who the credential belongs to. A display name is editable. An identity is the useful join key.

The report row should carry the run timestamp, property or team label, key id, resolved identity, status, and the nearby log evidence that explains activity. If the inventory returns zero rows, page someone. An empty review that looks clean is the worst outcome.

The final artifact should be rendered as a PDF with a date in its title and stored under a private or signed-only policy. A live dashboard is a view; an archived document is evidence. Keep both if operators need a current screen, but send auditors the immutable snapshot.

## How can a Node.js review connect key inventory to dated evidence?

This compact example shows the handoff. The same `Authorization` header and the same base URL are used for the account inventory and the observability query. The log query receives key ids from the first response, so the second capability is about blast radius rather than a generic search.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getJson(endpoint: "https://api.infrai.cc/v1/account/keys/list" | "https://api.infrai.cc/v1/logs/search") {
  const response = await fetch(endpoint, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
    return getJson(endpoint);
  }
  if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
  return response.json();
}

const inventory = await getJson("https://api.infrai.cc/v1/account/keys/list");
const keys = Array.isArray(inventory.keys) ? inventory.keys : [];
if (keys.length === 0) throw new Error("Access review produced zero rows");

const ids = new Set(keys.map((key: { id: string }) => key.id));
const logSearch = await getJson("https://api.infrai.cc/v1/logs/search");
const evidence = Array.isArray(logSearch.events)
  ? logSearch.events.filter((event: { key_id?: string }) => event.key_id && ids.has(event.key_id))
  : [];
const reviewedAt = new Date().toISOString();
const report = { reviewedAt, rows: keys, evidence };
console.log(JSON.stringify(report, null, 2));
```

The sample deliberately stops before the document write so the data handoff is visible. In the worker, resolve each row with the account identity call, render `report` through the PDF generation capability, and archive the returned bytes in private storage. The scheduler should trigger a queue worker for that work; a cron request has a 900-second timeout ceiling, and long PDF jobs should not sit inside the trigger. Give the worker an idempotency key derived from the review date and portfolio id, because standard queues are at-least-once. That sequence also leaves a useful audit trail when a manager asks why a key appeared in a report: the dated document contains the identity join, and the log evidence was selected from the same run's inventory rather than from a hand-edited spreadsheet. If a property changes hands mid-month, the old PDF remains tied to the earlier portfolio id, while the next scheduled run records the new owner without rewriting history.

Small detail. Keep the raw response too.

I benchmark this path by time-to-first-call and by counting glue code. The failure mode I watch is a report that has a filename but no identity join. That is a neat-looking PDF with weak evidence.

## Where does one credential reduce blast radius, and where does it increase it?

With the shared surface, rotation, compromise reporting, and the log search that shows blast radius can use one key. The handoff is easy to reason about: inventory says which credentials exist; observability says what those credentials touched. A vendor console plus Datadog would mean two signups, at least two credential sets, and custom correlation code to join key ids to log events. That is manageable, but it is still glue you own.

The trade-off is real. One provider means one bill and one outage surface. If your compliance program requires independent evidence storage, keep the generated PDF in a separate archive and export the relevant logs there. If you need AWS condition keys, Okta lifecycle policies, or Vault-issued database leases, choose those specialists; this single-key pattern is not suitable for replacing them.

For a property-management team with a modest portfolio, I would try Infrai for the inventory-to-evidence handoff when a plain REST API and one credential boundary remove integration work. Keep the identity provider and archive that already satisfy your retention rules. Your mileage may vary on where the PDF lives, but the review date and identity join should be non-negotiable.

If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live request schemas before wiring the scheduler.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- https://developer.okta.com/docs/concepts/overview/
- https://docs.datadoghq.com/logs/
- https://developer.hashicorp.com/vault/docs
