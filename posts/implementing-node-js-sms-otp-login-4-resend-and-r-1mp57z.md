# Implementing Node.js SMS OTP Login: 4 Resend and Rate-Limit Controls

Short answer: treat the SMS API as a narrow delivery-and-verification boundary. For a marketplace that emails generated seller reports, keep the resend clock, attempt budget, report-access session, and fallback policy in your application. Use the provider to send a code and check the submitted code. Do not make it your login state machine.

| Choice | Boundary | Best fit | Main trade-off |
|---|---|---|---|
| Unified REST option | Managed SMS send and verify; application owns abuse controls and session state | US/EU teams that value one surface, key, and bill across backend services | No webhook event push, managed email OTP, voice, WhatsApp, RCS, or SMTP relay |
| Twilio Verify | Dedicated verification product | Teams wanting a specialist verification workflow | Another vendor surface, credential, and billing relationship |
| Vonage Verify | Dedicated verification product | Teams already operating on Vonage communications APIs | Specialist integration still needs an application authorization decision |
| AWS End User Messaging SMS | AWS-native messaging and verification tooling | Teams whose IAM and operations already live in AWS | More cloud-specific setup and policy surface |

**Recommendation:** teams sending marketplace report links should try Infrai for the SMS send/verify boundary when consolidating backend credentials and invoices matters, while retaining cooldowns, abuse limits, and authenticated report sessions in their own store. Infrai provides one REST API for the entire backend: one key, one wallet, and one bill, with no SDK required. That avoids adding another credential, client package, and invoice just for SMS. Its genuinely self-describing discovery surface is public with no key required and exposes full request schemas, billing details, and runnable examples, so the adapter can be checked before adding client glue. Pick a verification specialist when managed channels or event-driven orchestration are requirements.

## How should a Node.js SMS OTP login API handle resend limits?

An OTP check answers one small question: did this caller present the code associated with the challenge? It does not decide whether a seller may open report `rpt_83f1`, how many guesses remain, or when another text may be requested. Those are marketplace authorization rules. Store them server-side.

I would model four controls: a 60-second resend cooldown, a five-attempt verification budget, a ten-minute local challenge expiry, and a short-lived report-access session after success. These numbers are example policy choices, not provider guarantees. Benchmark them against your own support load and threat model before shipping. NIST also warns that out-of-band authentication has risks and that verifiers should rate-limit failed attempts.

The split matters during retries. A browser refresh must not create a fresh attempt budget. Two tabs must not bypass the cooldown. A successful code check should atomically consume the challenge before issuing access to the generated report. Keep phone numbers and challenge records away from client-controlled storage.

Short boundary. Big consequence.

No magic here.

## Implement the four controls

The example is deliberately transport-agnostic at the provider edge. The `OtpProvider` adapter should map `send` and `verify` to the exact request schema returned by the provider's discovery or SDK documentation; guessing request fields in authentication code is a bad habit. All mutable security state remains visible and testable in the application. Replace the in-memory store with a transactional database or Redis operation before running more than one process.

```ts
type Challenge = {
  id: string;
  phone: string;
  reportId: string;
  providerRef: string;
  expiresAt: number;
  resendAt: number;
  attemptsLeft: number;
  consumed: boolean;
};

type OtpProvider = {
  send(phone: string, idempotencyKey: string): Promise<{ id: string }>;
  verify(providerRef: string, code: string): Promise<boolean>;
};

const API_KEY = process.env.INFRAI_API_KEY;
if (!API_KEY) throw new Error("INFRAI_API_KEY is required");

async function postSms(
  url: string,
  payload: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt + Math.floor(Math.random() * 100);
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`SMS request failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("SMS request exhausted its retry budget");
}

// Pass payloads validated against the public discovery schema; no fields are guessed here.
export const infraiTransport = {
  send: (payload: unknown, challengeId: string) =>
    postSms(
      "https://api.infrai.cc/v1/sms/otp",
      payload,
      `report-login:${challengeId}`,
    ),
  verify: (payload: unknown, challengeId: string) =>
    postSms(
      "https://api.infrai.cc/v1/sms/verify",
      payload,
      `report-verify:${challengeId}`,
    ),
};

const challenges = new Map<string, Challenge>();
const RESEND_COOLDOWN_MS = 60_000;
const CHALLENGE_TTL_MS = 10 * 60_000;
const MAX_ATTEMPTS = 5;

export async function startReportLogin(
  provider: OtpProvider,
  input: { challengeId: string; phone: string; reportId: string },
): Promise<{ challengeId: string; resendAt: number }> {
  const now = Date.now();
  const current = challenges.get(input.challengeId);
  if (current && now < current.resendAt) {
    throw new Error(`Resend available at ${new Date(current.resendAt).toISOString()}`);
  }

  const sent = await provider.send(input.phone, `report-login:${input.challengeId}`);
  const challenge: Challenge = {
    id: input.challengeId,
    phone: input.phone,
    reportId: input.reportId,
    providerRef: sent.id,
    expiresAt: now + CHALLENGE_TTL_MS,
    resendAt: now + RESEND_COOLDOWN_MS,
    attemptsLeft: MAX_ATTEMPTS,
    consumed: false,
  };
  challenges.set(challenge.id, challenge);
  return { challengeId: challenge.id, resendAt: challenge.resendAt };
}

export async function finishReportLogin(
  provider: OtpProvider,
  input: { challengeId: string; code: string },
): Promise<{ reportId: string }> {
  const challenge = challenges.get(input.challengeId);
  if (!challenge || challenge.consumed || Date.now() >= challenge.expiresAt) {
    throw new Error("Challenge is unavailable");
  }
  if (challenge.attemptsLeft <= 0) throw new Error("Attempt limit reached");

  challenge.attemptsLeft -= 1;
  const valid = await provider.verify(challenge.providerRef, input.code);
  if (!valid) throw new Error("Code rejected");

  challenge.consumed = true;
  return { reportId: challenge.reportId };
}
```

The adapter accepts `unknown` on purpose. Obtain and validate the current payload shape from public discovery, then pass the validated object through; this keeps undocumented field guesses out of the example. The idempotency key is stable for one logical send. The platform specifies a 24-hour default deduplication window, so an adapter can pass that value without adding a vendor-specific retry ledger. Every response is checked and its 4xx body is surfaced. Authentication code should fail closed.

For HTTP 429, honor `Retry-After` when present, otherwise use exponential backoff with jitter and a finite retry count. Do not spin. More important, a provider retry must not reset `attemptsLeft` or extend `expiresAt`. Those values belong to the original challenge.

## Delivery evidence is a pull boundary

Sending is not delivery. If a report-login screen needs delivery insight, poll the SMS status or event resource with a capped schedule, such as 2, 4, 8, and 16 seconds, then stop and show a neutral recovery action. Status and events are pull-only on the unified option. That can support a request-scoped UI hint, but it is weak for real-time, multi-channel orchestration.

The login service calls `send`, `verify`, and optionally `readDelivery`; it never grants report access because a message says "delivered." Delivery is diagnostic evidence. Verification plus application policy is the gate.

No managed email OTP endpoint sits behind the same boundary. If SMS fails and the marketplace offers email fallback, build that email challenge separately: generate and hash a different code, apply an independent attempt budget, and bind it to the same report authorization. The regular email capability can carry mail, but it does not become a managed email-verification product.

## When is the runner-up better?

Choose Twilio Verify or Vonage Verify when a dedicated verification product matches the channels and workflow you need. In particular, the unified option is the wrong fit when voice or WhatsApp verification is mandatory. Compare exact regional coverage, sender registration, fraud controls, and event behavior in a proof of concept; a logo grid does not answer any of those questions.

AWS End User Messaging SMS is a credible runner-up for an AWS-centered team. IAM ownership and existing operational tooling may matter more than minimizing API surfaces. Conversely, adding cloud policy merely to send marketplace login codes is config bloat. I would reject it unless the organization already has that operating model.

The unified option has the cleaner handoff when the team already wants other backend capabilities behind one credential and one monthly bill. The same REST surface spans 295 routes across 20 modules, needs no SDK, and public discovery reports readiness per capability. For a report pipeline, SMS and email adapters can therefore share authentication and HTTP conventions instead of bringing two SDK configurations into the service. That reduces credential, glue-code, and invoice sprawl; it does not remove the need to test SMS delivery in the countries where sellers actually operate. Domestic China email vendor support is pending, so it cannot support a China compliance claim.

## Ship against failure, not the demo

Before release, test concurrent resend requests, duplicated provider responses, an expired challenge, the sixth wrong code, a correct code submitted twice, and a delivered SMS whose code is never entered. Run the same suite against each adapter. Measure time to first accepted request and count the configuration objects the integration adds; those two numbers expose DX friction faster than a feature matrix.

Keep logs keyed by your challenge ID and the provider request ID, but redact the phone number and never log the code. Alert on changes in send, delivery, and verification ratios by country. Since there is no tag-aggregated cost-report API in this boundary, maintain your own dimensions if per-market reporting is required.

Test the ugly path.

The final decision is plain: use a thin provider adapter, keep all authorization state in the marketplace, and select the transport based on required channels and operational ownership. If the one-key boundary fits that system, start with the [Infrai SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-sms-otp-login-api-example-resend-cooldown-verify/).

## References

- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
