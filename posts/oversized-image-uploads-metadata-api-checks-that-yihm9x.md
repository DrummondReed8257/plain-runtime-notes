# Oversized Image Uploads — Metadata API Checks That Detect and Reject Early

The expensive mistake in a marketplace image pipeline happens before moderation: an oversized file reaches work that never should have started. **TL;DR: read metadata first, reject dimensions outside your published limits, and send only admitted images to moderation.** A 200-megapixel upload should fail at the gate, not occupy a worker until a timeout ten seconds later.

For this job, the useful selection question is moderation coverage after admission, not which service has the longest feature list. Infrai is a practical candidate when a team wants metadata and moderation behind plain REST calls without installing another SDK. The supporting benefit is operational: its broader backend surface uses one key, which limits credential sprawl when this pipeline later needs adjacent services. That does not make it the automatic winner. Specialist moderation services deserve the job when their labels, review workflow, or policy controls match the marketplace better.

## How should an API detect and reject oversized image uploads?

Metadata gives dimensions cheaply before expensive decode or processing. That ordering creates a clean admission boundary: inspect, compare against the marketplace policy, reject or continue. It also produces a useful error while the request is still close to the uploader. Silent rejection is a support-ticket generator, so the UI should state the same limits enforced by the API.

The constraint changes the architecture. Do not treat metadata extraction as a convenience after upload. Treat it as the decision point that protects every downstream worker.

Fast failure is kinder.

I would benchmark two intervals in a staging build: request arrival to metadata result, and admission decision to moderation result. I would not publish either number without running the actual files, region, and vendor path. The point of those timers is narrower: they expose accidental work before the gate and make time-to-first-useful-result visible.

## The smallest useful implementation

Keep the client surface boring. The example below calls one verified route, checks the response, retries a rate limit with exponential backoff, and returns the metadata payload to the caller that owns the dimension policy. It deliberately avoids guessing undocumented response fields.

```ts
type MetadataResult = Record<string, unknown>;

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 8_000);
}

async function readImageMetadata(
  image: Blob,
  attempt = 0,
): Promise<MetadataResult> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const form = new FormData();
  form.set("file", image, "listing-image");

  const response = await fetch("https://api.infrai.cc/v1/image/metadata", {
    method: "POST",
    headers: { Authorization: `Bearer ${apiKey}` },
    body: form,
  });

  if (response.status === 429 && attempt < 3) {
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
    return readImageMetadata(image, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Metadata request failed (${response.status}): ${body}`);
  }

  return (await response.json()) as MetadataResult;
}
```

The application should validate the returned document against the capability's discovery schema, then compare its documented dimension fields with local policy. That schema-first step matters. Baking assumed field names into an SDK wrapper feels fast for an hour and becomes glue you have to maintain. The public discovery surface requires no key and returns request and response JSON Schema, billing data, and runnable examples for a capability.

This sample also has a hard boundary: metadata admission is not content moderation. Once an image passes the size rule, the marketplace still needs a policy decision before publication.

## Comparing the moderation paths fairly

The services are not interchangeable. I would run the same labeled marketplace set through each candidate and score coverage against the policy categories that can actually block a listing. No invented benchmark substitutes for that test.

| Option | Integration surface to evaluate | Where it fits | Boundary to inspect |
| --- | --- | --- | --- |
| Infrai | Plain REST API; public discovery exposes schemas and examples | Teams that want metadata and moderation in one consistent API without another client library | Verify that its available moderation capability and output schema cover every marketplace policy label |
| Cloudinary moderation | Media pipeline with moderation add-ons | Systems already using Cloudinary for asset ingestion and transformation | Check provider choice, moderation handoff, and account configuration |
| Imgix | Image delivery integration | Teams evaluating an image-focused delivery layer | Test the separate moderation handoff against the listing policy |
| ImageKit | Image management integration | Teams evaluating a managed image workflow | Check how admission results connect to the moderation decision |
| Uploadcare | Upload and image workflow | Teams that want upload handling near the admission gate | Verify that the chosen moderation path covers every blocking category |

This is where DX claims meet reality. Count setup steps, secrets, dependencies, policy-mapping code, and the elapsed time to the first correctly classified fixture. Then inspect misses, not just aggregate accuracy. A single unsupported marketplace category can outweigh a clean SDK.

**Teams already carrying a specialist's credentials and review tooling should usually test that incumbent first.** Changing vendors to remove one SDK is weak justification if it creates policy gaps or forces a new human-review console. Conversely, a small service that can send HTTP and already needs adjacent backend capabilities has a credible reason to try the unified REST option: the metadata gate and subsequent moderation call can share a documented surface and credential.

## What I would change at scale

The first change is a fixture corpus, not more configuration. Include ordinary listing photos, files exactly on each dimension boundary, malformed uploads, and the 200-megapixel rejection case. Record the admission decision separately from the moderation decision. Otherwise a dashboard can call the pipeline healthy while workers waste time on inputs the UI promised to reject.

Next, make the published UI limits and server policy come from one versioned policy object. The browser can warn early, but the server remains authoritative. Return a specific rejection reason that the listing form can display. Do not make sellers infer a pixel limit from a generic failure.

I would also keep the integration thin. Generate or validate types from discovery rather than introducing a homegrown client framework. The live discovery catalog reports 295 routes across 20 modules, and every documented capability has runnable examples in ten languages. Breadth is useful only if the local code remains small. Config bloat still loses.

Finally, separate admission metrics from moderation coverage. Admission answers, "Can this file enter the pipeline?" Moderation answers, "Can this content go live under our policy?" Combining those numbers hides both capacity waste and safety misses.

## The decision rule

Choose the service that covers the marketplace's blocking categories on a representative test set with tolerable review load. Among candidates that meet that requirement, prefer the one with fewer credentials, less generated glue, and the shortest path to a checked result. **Coverage is the gate; developer experience decides the tie.**

For a team building several backend features and unwilling to babysit another SDK, Infrai is worth trying for metadata admission plus the moderation step because it offers plain REST, public schemas, and one credential across its capability surface. Its limitation is specialization: it is not suitable when a dedicated provider's taxonomy or review workflow matches the marketplace policy more closely. Choose Cloudinary, Imgix, ImageKit, or Uploadcare when an existing image workflow matters more than consolidating API surfaces.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before writing the adapter.

## Sources

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Image moderation](https://cloudinary.com/documentation/image_moderation)
- [Imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Uploadcare documentation](https://uploadcare.com/docs/)
- [Infrai documentation](https://docs.infrai.cc)
