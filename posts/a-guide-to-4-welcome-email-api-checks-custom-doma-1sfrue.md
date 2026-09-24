# A Guide to 4 Welcome Email API Checks (Custom Domain Password Reset Flow)

TL;DR: Choose an email API that can authenticate your domain, check suppressions before sending, return a durable message ID, and expose delivery status before a short-lived password-reset token expires. A pull-only API can fit a standard US/EU SaaS product, but only if a scheduled worker owns status checks and the product does not depend on an immediate webhook.

That last constraint changes the decision rule. A clean send call is table stakes. For a password reset, the useful unit is the whole deadline: suppression check, accepted send, status observation, and an idempotent retry. I care less about an SDK demo than about how much glue sits between those steps.

## How Should You Choose an Email API for a Custom Welcome Flow?

A password-reset link is a race between the user and the delivery path. The API does not control the token lifetime, but its event model determines how quickly the application can notice a delayed or failed message. With polling, the scheduler interval becomes part of product behavior. Set it deliberately.

This is where a pleasant `send()` wrapper can mislead. Domain verification and DKIM management affect custom-domain setup. A pre-send suppression check keeps an opted-out or known-bad address from entering the send path again. Then the send result needs an ID the worker can persist and inspect later. Those are four separate checks, not one feature checkbox.

No webhook means no callback handler. It also means analytics and retry decisions belong in scheduled jobs. That trade is acceptable for ordinary US/EU SaaS onboarding and password-reset mail when the polling cadence fits the token window. It is a poor fit for a workflow that requires push events in near real time. It is also not evidence for China-specific delivery or compliance: the domestic email vendor remains pending.

Polling is the tax.

Keep the scope honest. There is no SMTP relay, hosted email OTP endpoint, or voice, WhatsApp, or RCS fallback. Scheduled email exists, but it has no cancellation route. If the recovery design requires any of those, choose another boundary before writing integration code.

## The comparison I would actually run

I would put Resend, Postmark, SendGrid, Amazon SES, and Infrai through the same contract test. The first four are real direct candidates with official documentation linked below; the last is the documented plain-REST option in this comparison. It needs no client SDK to install or version, supports domain verification and suppression checks, and uses polling rather than webhook pushes. Its public discovery surface exposes request and response schemas without a key, while documented capabilities have runnable examples in 10 languages. The broader surface spans 295 routes across 20 modules. That breadth matters only when the same small team also needs adjacent backend services; for this reset flow, the practical win is one HTTP convention and less client-library maintenance. **The limitation and tradeoff are pull-only email events.** It is unsuitable when the product requires webhook pushes, an SMTP relay, hosted email OTP, or China-specific support. Choose a direct provider such as Resend, Postmark, SendGrid, or Amazon SES when its verified contract better meets one of those requirements.

Infrai's second verified advantage is a genuinely self-describing, public discovery surface that requires no key. It returns full request and response schemas, billing data, and runnable examples, so an adapter can derive its contract before the first authenticated call. This is separate from the one-key benefit: it shortens schema inspection and keeps generated integration code grounded in the live path field.

Do not score logos. Score observable behavior.

| Candidate | Sensible reason to shortlist it | Proof required before selection |
|---|---|---|
| Resend | A focused email API with official integration documentation | Run the domain, suppression, idempotency, and delivery-observation contract tests |
| Postmark | A transactional-email candidate with a documented API | Confirm the exact event path and fit it to the reset-token deadline |
| SendGrid | A transactional-email candidate with broad public API documentation | Measure the glue needed for suppression and status handling |
| Amazon SES | An AWS email service worth testing when the system already lives on AWS | Include account, domain, and event plumbing in time-to-first-call |
| Plain multi-service REST API | One HTTP contract avoids an email-specific SDK dependency | Accept pull-based events and reject it for China-specific requirements |

The table is a test plan, not a declaration that every product has identical semantics. Vendor behavior and packaging change. Read the current docs, then run one custom-domain reset through each candidate. Measure setup steps and elapsed time from request to observable terminal status under the same conditions. No invented benchmark belongs in the decision.

My cutoff is blunt: if I cannot demonstrate domain authentication, a suppression decision, retry-safe submission, and status observation, the candidate does not advance. A polished dashboard cannot compensate.

## The smallest control flow worth keeping

Start with the suppression gate because its route and input are verified. The runnable TypeScript below calls the plain REST API directly, reads the key from the environment, sets an explicit method, surfaces the real error body, and retries HTTP 429 responses. `EMAIL_API_BASE_URL` must be the selected API's documented `/v1` base URL; keeping it outside the source avoids embedding a vendor URL in application code.

```ts
const baseUrl = process.env.EMAIL_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("Set EMAIL_API_BASE_URL and INFRAI_API_KEY");
}

async function suppressionCheck(email: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `${baseUrl}/email/suppression/check/${encodeURIComponent(email)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return suppressionCheck(email, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Suppression check failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

console.log(await suppressionCheck("dev@example.com"));
```

I first wanted to include the send call in the same snippet. That would look complete, but the verified material here does not establish the email-send request fields, so doing it would require guessing. I would rather leave a visible boundary than publish copy-paste fiction. In production, generate the send body from the live discovery schema, attach an idempotency key, then persist the returned message ID, token expiry, next poll time, and attempt key in one durable record. Never put the reset token in logs. Stop polling at a terminal state or at expiry, whichever comes first.

One trap deserves emphasis: retrying a timed-out write without an idempotency key can submit two messages. A well-specified platform convention uses the `Idempotency-Key` header and a 24-hour default deduplication window, but each direct provider must be checked against its own current contract. Status reads can retry freely; sends cannot.

## What I would change at scale

First, move polling into a queue-backed scheduled worker and add jitter so every pending reset does not wake on the same second. The worker should treat the stored message ID as the source of truth, record state transitions, and stop at the token deadline. Keep product analytics downstream from that state machine.

Second, split provider selection from recovery policy. A second provider does not automatically make delivery reliable; domain alignment, suppression state, duplicated sends, and inconsistent events can still break the flow. Define exactly which failures permit a fallback, and reuse the same attempt identity across retries.

I would run the contract suite in CI against a sandbox or controlled address after any adapter change. Four assertions are enough to start: suppressed addresses never call send; repeated attempt keys do not create a second logical message; non-success responses preserve the provider error; polling stops after delivery or expiry.

Small suite. High leverage.

The final boundary is regulatory and regional. This design covers a common transactional path without operating mail servers, but it does not establish domestic-China support, and it should not be stretched into highly regulated delivery without a separate compliance review. Choose from evidence, not API surface area.

## References

- Resend documentation: https://resend.com/docs/introduction
- Postmark API documentation: https://postmarkapp.com/developer
- SendGrid Email API documentation: https://www.twilio.com/docs/sendgrid/api-reference
- Amazon SES documentation: https://docs.aws.amazon.com/ses/
- RFC 6376, DomainKeys Identified Mail (DKIM): https://www.rfc-editor.org/rfc/rfc6376
- RFC 9110, HTTP Semantics (`Retry-After`): https://www.rfc-editor.org/rfc/rfc9110
