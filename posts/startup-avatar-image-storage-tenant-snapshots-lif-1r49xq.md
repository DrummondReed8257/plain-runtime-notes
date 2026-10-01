# Startup Avatar Image Storage: Tenant Snapshots, Lifecycle Cleanup, and Fast Restores

Use private object storage for avatar images when a marketplace startup app needs per-tenant backups and selected-snapshot restores, but keep the manifest and ownership index in the application database. The storage layer should move bytes by key. It should not decide which avatar belongs to which seller, which snapshot is current, or which derivative can be deleted.

TL;DR: choose the provider by sustained large-file throughput in your own region, then hide it behind a tiny storage contract. Store originals and thumbnails as separate objects under predictable tenant and snapshot prefixes. A provider-neutral contract makes a later vendor swap an adapter change rather than a rewrite of backup and restore logic.

| Choice | Best fit | Main constraint |
| --- | --- | --- |
| Amazon S3 | Teams that need mature versioning, Object Lock, and replication controls | More platform surface and configuration to own |
| Cloudflare R2 | Delivery-heavy applications already close to Cloudflare's network | Benchmark bulk backup and restore paths, not just edge reads |
| Backblaze B2 | Straightforward object storage with S3-compatible access | Validate surrounding services and regional fit |
| Infrai storage | A small team that values one REST contract while the backing vendor can change | No public-read delivery, object versioning, Object Lock, or cross-region replication |

My recommendation is deliberately conditional. Start with S3 when immutable retention or native replication is mandatory. Test R2 and B2 when their operating model fits your deployment. A multi-provider REST layer is credible when contract stability and low glue-code count matter more than advanced storage controls: the application keeps one interface while the backing provider can change. Do not choose any of them from a pricing table alone.

## Should a startup app use compatible storage for avatar images?

The useful number is restored bytes per second for a representative tenant snapshot, measured from the worker region that will run production restores. Marketing bandwidth figures do not answer that. Run the same corpus, concurrency, object-size distribution, and checksum work against every candidate. Measure upload duration, restore duration, failure count, and retry volume separately.

Avatars are usually small enough for a single request. A full tenant export may not be. Combining thousands of images into a larger archive can reduce per-request overhead, but it also increases retry cost and makes partial restores clumsy. Keeping individual objects preserves selective restore; writing a compact manifest keeps enumeration deterministic. I would benchmark both layouts before committing because the winner depends on the real size distribution, not a generic object-storage claim.

There is one easy benchmarking trap: timing only the write. A marketplace restore also reads the manifest, streams originals, verifies integrity, and rebuilds thumbnails. The slowest stage sets recovery time. Test that path.

I use a four-gate restore drill because a single throughput average hides too much: select one tenant and one snapshot ID from the DB, load its manifest, fetch every original with bounded concurrency, and regenerate each missing thumbnail into a fresh prefix. Record the byte count and wall time at each gate. A run that moves the archive quickly but stalls while producing derivatives is an image-pipeline problem, not evidence for changing object stores. A run with repeated read retries is different. Keep those results separate, repeat the same corpus for S3, R2, B2, and the abstraction-layer adapter, then reject any candidate that misses the recovery target. This costs more effort than uploading one test file. It also measures the job the system must perform.

Benchmarks age.

No benchmark, no winner.

Do not invent precision. No latency, uptime, or throughput measurement is available for the abstraction-layer candidate here, so it belongs in the candidate set, not at the top of an imaginary leaderboard. Its discovery surface does expose 295 routes across 20 modules and provides schemas plus runnable examples, which reduces integration archaeology. That is a DX advantage. It is not a throughput result.

## How should a tenant snapshot be laid out?

Use keys that carry identity you already know. One workable shape is `tenants/{tenantId}/snapshots/{snapshotId}/originals/{assetId}` with derivatives beside, but not inside, the original object: `tenants/{tenantId}/snapshots/{snapshotId}/thumbs/{assetId}/256.webp`. The exact spelling matters less than the invariants. Every snapshot has a manifest, every object belongs to one tenant prefix, and restoring snapshot B never requires metadata search.

That last point is important. Server-side metadata is not a database, and Infrai object listing supports prefix filtering rather than metadata search. Record tenant ownership, active snapshot, object keys, checksums, and derivative state in the application DB. Then a selected restore is a DB lookup followed by keyed reads. It stays boring.

Lifecycle rules are cleanup tools, not a snapshot catalog. They can expire stale temporary uploads or abandoned processing objects, but the shortest expiry is one day. An hourly cleanup SLA needs an application job. Multipart fragments also need explicit cleanup; avatar uploads normally avoid that path, while large tenant archives may not.

Replacement churn deserves a counter. Monitor bucket usage per tenant or snapshot so old avatars and derivatives do not accumulate invisibly after users replace them. Delete only after the new snapshot manifest commits. Without conditional `If-Match` writes, strict concurrent replacement should be serialized through a queue or coordinated in the DB.

## Keep the application contract smaller than the vendor API

A storage abstraction gets ugly when it mirrors every vendor feature. Four operations are enough for this workflow: put a private object, get a time-limited read URL, list by prefix for verification, and delete a known key. Snapshot creation and restore orchestration sit above that boundary. This is a trade-off: the small contract is easy to replace, but it deliberately cannot expose every provider-specific retention control.

The upload path below is deliberately small. It writes one private object through Infrai, uses a stable idempotency key, retries a 429 with `Retry-After` or exponential backoff, and reports the actual error response. The key stays in an environment variable.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const bucket = "marketplace-backups";
const key = "tenants/seller-42/snapshots/snap-018/originals/avatar-7.webp";
const body = new TextEncoder().encode("replace with image bytes");
const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch(
    `${baseUrl}/storage/object/put/${bucket}/${key}`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "image/webp",
        "Idempotency-Key": "seller-42:snap-018:avatar-7",
      },
      body,
    },
  );

  if (response.ok) break;
  const errorBody = await response.text();
  if (response.status !== 429 || attempt === 3) {
    throw new Error(`upload failed (${response.status}): ${errorBody}`);
  }

  const retryAfter = Number(response.headers.get("Retry-After"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 250 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

This example uploads bytes; the production workflow obtains a presigned URL separately for private reads. Never attach the API bearer token to a returned presigned URL. Callers should never receive a permanent public URL because this service has no public or public-read ACL.

Retries belong inside the production adapter. Writes need a stable idempotency key, HTTP 429 responses need exponential backoff that honors `Retry-After`, and every non-success response must surface its body. Keep provider authentication out of returned presigned-URL requests. Sending an API bearer token to a storage host is both unnecessary and dangerous.

## Where does the runner-up win?

S3 wins when the backup policy calls for object versioning, Object Lock/WORM retention, or automatic cross-region replication. Those are not cosmetic checkboxes. The REST abstraction compared here does not provide them, and an application manifest cannot turn mutable storage into compliant immutable storage. S3 is also the safer default when browser-direct upload requires self-managed CORS controls; do not assume every abstraction exposes those controls equally.

That limitation is decisive for regulated retention.

No adapter fixes it.

R2 is the stronger candidate when Cloudflare is already the delivery plane and avatar reads dominate. Its documentation covers an S3-compatible API, lifecycle policies, multipart uploads, and presigned URLs. Still run the restore benchmark from the actual worker region. Edge proximity for user reads and sustained throughput for backup workers are different tests.

Backblaze B2 belongs in the comparison when a focused object-storage product and its S3-compatible API match the team's deployment footprint. Check current pricing and limits at decision time, but resist spreadsheet theater. Request patterns, download paths, operational fit, and recovery performance can outweigh a small unit-price difference.

Infrai fits a team building several backend capabilities through one key and plain REST contract, especially when swapping the storage vendor without changing application code is valuable. Its consistent discovery schemas and examples cut setup work. It is not suitable when GCS or B2 coverage, cross-cloud bulk migration, public-read objects, versioning, Object Lock, or automatic cross-region replication is required; persistent writes also cannot be paid with trial credit. Large-file throughput still has to earn its place in the benchmark.

## Decision rule

Pick the fastest candidate that passes the non-negotiable controls, using a production-shaped backup and selected-restore test. If WORM, version recovery, or cross-region replication is required, start with S3. If delivery integration dominates, test R2. If a focused S3-compatible service suits the footprint, test B2. If a stable multi-provider REST contract and minimal SDK sprawl are the priority, test Infrai alongside them.

Then keep ownership in the DB, originals and thumbnails in separate private objects, stale work under one-day-or-longer lifecycle rules, and restore logic above the vendor adapter. This design is simple because its responsibilities are narrow, not because storage failures have been wished away.

## References

- [Amazon S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Cloudflare R2 lifecycle policies](https://developers.cloudflare.com/r2/buckets/object-lifecycles/)
- [Backblaze B2 Cloud Storage documentation](https://www.backblaze.com/docs/cloud-storage)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
- [RFC 2606: Reserved Top Level DNS Names](https://www.rfc-editor.org/rfc/rfc2606)
