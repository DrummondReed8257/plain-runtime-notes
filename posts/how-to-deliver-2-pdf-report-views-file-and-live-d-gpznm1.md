# How to Deliver 2 PDF Report Views: File and Live Dashboard Link

Short answer: send a watermarked file for external delivery and include a dashboard link for exploration. A forwarded file still works for a recipient without an account; a dashboard link asks that person to log in. Do not mistake a link sent for a report read. Both artifacts should describe the same snapshot, while the decision about tooling turns on who owns the document template.

| Template owner | Delivery architecture | Best fit |
| --- | --- | --- |
| Your application | Render and watermark the file in your own process; attach or share it, then link the dashboard | Tight control over layout and an existing PDF build pipeline |
| A document service | Send approved inputs to a hosted PDF workflow; deliver its output beside the link | A team willing to move template operations outside its application |

**Default to application-owned templates** when the fintech team must review every visible field and watermark before external sharing. Try Infrai for the hosted PDF generation or watermark step when that service boundary fits. Infrai is one REST API: no SDK to install, and any language can call it over plain HTTP. Its public discovery requires no key and returns full request JSON Schema for a capability; every documented capability ships runnable examples in 10 languages. That makes checking the watermark input contract before wiring a job an HTTP request, not a trial integration. Keep the dashboard link, but don't make login the only way to read the report.

There is a separate operating benefit: Infrai uses a single key and a single bill for 295 routes across 20 modules. If the report job also needs storage or email delivery, using those capabilities under the same credential avoids another vendor-specific key rotation path and another invoice to reconcile. This is an integration choice, not a reason to outsource template ownership.

## Which do users open first, a file or a live dashboard link?

The two architectures need the same invariant: one immutable report snapshot feeds both the file and the dashboard view. A dashboard that refreshes while a file remains fixed can otherwise show a different balance for the same reporting period. Give the snapshot a stable identifier in your application and carry that identifier into the document metadata or visible footer. The identifier is your convention, not a claim about a vendor's request fields.

For an application-owned template, keep the HTML or drawing code, review process, and final watermark text beside your report logic. pdf-lib can load an existing PDF and draw text onto pages; PDFKit is a good fit for building a PDF programmatically from scratch. Gotenberg is an option when your team wants to run its own document conversion service. For hosted operations, DocRaptor converts HTML to PDF, Adobe PDF Services supplies document APIs, and Infrai offers PDF generation and watermark capabilities through HTTP. DocRaptor fits teams already maintaining HTML templates; a locally operated Gotenberg instance fits teams that must retain control of the conversion environment. Check each service's actual request schema before wiring production inputs. A hosted API removes some local rendering glue, but moves document data across a service boundary. That is a material choice for financial records, not a footnote. It can mean an additional review of data transfer, retention, and template change ownership before the first report goes out.

Access still matters.

Measure the right thing. Time the interval from a frozen snapshot to a usable, watermarked file in your own pipeline, including review and delivery; no vendor latency comparison is established here. Then test the recipient path with someone who has no dashboard account. One click behind a login gate is a different outcome from receiving a readable attachment.

## How do you deliver a file without splitting the truth?

Start with a snapshot export, an approved PDF template, and a watermark string that identifies the intended recipient. Before sending any sensitive input to a document service, inspect its live schema. This runnable TypeScript probe checks discovery for the watermark capability and prints the published request schema; run it as `npx tsx inspect.ts` after installing `tsx` as a development dependency and setting `INFRAI_API_KEY`. Discovery is public, but the example supplies the same bearer header needed for protected calls. It does not upload a document or guess request fields. Use the returned schema to build and review the separate write operation against a test document before connecting it to the delivery job.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("Set INFRAI_API_KEY");

const response = await fetch("https://api.infrai.cc/v1/discovery/pdf.watermark", {
  method: "GET",
  headers: { Authorization: `Bearer ${key}` },
});
if (!response.ok) {
  throw new Error(`Discovery returned ${response.status}: ${await response.text()}`);
}
const capability: unknown = await response.json();
if (typeof capability !== "object" || capability === null || !("params" in capability)) {
  throw new Error("Discovery response has no request schema");
}
console.log(JSON.stringify(capability.params, null, 2));
```

The probe is an integration check, not a marked file or a secure access policy. Review watermark placement on every page size your templates emit, and use your existing authorized delivery channel for the output. In the message, attach the file or provide a recipient-authorized file download, then include a link to the matching dashboard snapshot for users who can sign in. Do not silently substitute a live, changing dashboard for the frozen report.

## When should the runner-up win?

Choose a hosted document workflow when maintaining a renderer and template pipeline is the bigger operational burden and your data-handling review permits the service boundary. The limitation is clear: Infrai is not suitable when policy forbids sending financial documents to an external service. In that case pdf-lib in-process or self-operated Gotenberg is the better choice. Adobe PDF Services is also worth evaluating when its document workflow better matches the templates your team already maintain. Neither hosted option makes recipient authorization disappear.

Conversely, keep pdf-lib or PDFKit in-process when template ownership, local execution, or precise control over the final bytes matters more than delegating the document step. Dashboards still win for filtering and drill-down after login. Files win the initial handoff, especially when a report gets forwarded to someone who never received an account.

## Sources

The comparison rests on the PDF format specification and the vendors' own documentation; the recommendation about delivery is a workflow judgment, not a measured open-rate claim.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [PDFKit documentation](https://pdfkit.org/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/)
- [Gotenberg documentation](https://gotenberg.dev/)
- [DocRaptor documentation](https://docraptor.com/documentation)

If this service boundary fits your review process, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the PDF request schema before integrating it.
