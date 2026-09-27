# Transactional Email API: Gaming Password Reset Flow with Repository-Owned Templates

A gaming account-recovery message has an awkward constraint: the player needs it now, but an invalid address must stop consuming attempts now too. That changes the API choice. **Short answer:** keep the password-reset template and its variables in the application repository, put a narrow transport interface behind it, and require structured bounce events that can update a suppression record before the next send. Custom-domain authentication is an acceptance test, not a differentiator.

This is less glamorous than comparing SDK method names. It is also the part that survives a provider change. The application should decide what a reset message means; the mail transport should decide how to submit it. A dashboard-owned template reverses that boundary and makes a security-sensitive flow depend on state that code review cannot see.

## Should a transactional email API own the password reset flow?

Password-reset copy is executable product behavior. It contains a destination URL, user-visible context, expiry language, and the exact variable contract between account services and mail delivery. In a gaming system, that contract may also distinguish a player handle from the login address. Treating the template as console content creates two deployable artifacts with separate histories.

I would test a candidate transport with one deliberately boring question: can a fresh checkout render and test the complete message without opening an administrative console? If the answer is no, time-to-first-call is hiding configuration work. The setup may still be acceptable, but template ownership has moved outside the normal review path.

Repository ownership has a cost. Designers lose some direct editing freedom, and every copy change follows the application's release process. I take that trade because reset mail is tied to authentication behavior. Marketing mail has a different risk profile and can justify a different owner.

That cost is real.

The limitation is clear: repository-owned templates are a poor fit when a content team must publish frequent, independent changes without an application release. In that case, a console-owned workflow may be the better choice, provided the team tests remote template versions and keeps their variable contract under review. For account recovery, my trade-off goes the other way because a broken variable or stale reset instruction is part of the authentication path, not a cosmetic defect.

There is another trap. "Accepted" is not "delivered." A successful submission only tells the application that the transport accepted the request. Delivery can fail later, so a useful abstraction needs both a synchronous submission result and an asynchronous outcome. Collapsing them into one boolean makes retry code dangerous: a timeout can lead to a duplicate message, while a later permanent bounce can leave a bad address eligible forever.

## The smallest boundary I would ship

The implementation below keeps vendor vocabulary out of the account service. It also makes the suppression check explicit. No decorator maze. No configuration object with forty optional fields.

```ts
type ResetMail = {
  messageId: string;
  recipient: string;
  playerName: string;
  resetUrl: string;
};

type Submission =
  | { accepted: true; transportId: string }
  | { accepted: false; retryable: boolean; reason: string };

interface MailTransport {
  sendPasswordReset(message: ResetMail): Promise<Submission>;
}

interface SuppressionStore {
  has(recipient: string): Promise<boolean>;
  put(recipient: string, reason: "invalid-recipient"): Promise<void>;
}

export async function sendReset(
  message: ResetMail,
  transport: MailTransport,
  suppressions: SuppressionStore,
): Promise<Submission> {
  if (await suppressions.has(message.recipient)) {
    return {
      accepted: false,
      retryable: false,
      reason: "recipient-suppressed",
    };
  }

  return transport.sendPasswordReset(message);
}
```

The transport implementation may call any transactional email API. The template does not need to know. Keep the rendered subject, text body, HTML body, sender domain, and variable validation on the application side; keep credentials, request signing, and provider response translation inside the adapter.

The `messageId` matters even though the example is small. It gives submission logs and later delivery events a shared application key without making a provider-generated identifier the primary identity. It should identify one logical reset notification. Retry policy can then distinguish another transport attempt from another user request.

For tests, use a fake transport and snapshots of both the text and HTML renderings. Reject unknown variables. Exercise the suppressed path and assert that the transport was never called. Then run a separate integration check against the chosen transport, because a unit test cannot prove DNS authentication or event delivery.

## Bounces are state changes, not analytics

A bounce handler should translate transport-specific event payloads at the edge, authenticate the incoming event using the mechanism offered by that transport, and pass a small internal event onward. Parsing raw payloads throughout the codebase is glue that multiplies during a migration.

```ts
type DeliveryEvent = {
  messageId: string;
  recipient: string;
  outcome: "delivered" | "temporary-failure" | "invalid-recipient";
  occurredAt: string;
};

export async function recordDeliveryEvent(
  event: DeliveryEvent,
  suppressions: SuppressionStore,
): Promise<void> {
  if (event.outcome === "invalid-recipient") {
    await suppressions.put(event.recipient, "invalid-recipient");
  }
}
```

Keep this consumer idempotent. Event systems retry. A repeated invalid-recipient event should produce the same stored state, not another side effect. The suppression record also needs an auditable source event and a deliberate removal path, even if the compact interface above omits those storage details.

Do not use opens as the success signal for this loop. Apple Mail Privacy Protection can prevent senders from seeing whether a recipient opened a message and masks the recipient's IP address. That makes open telemetry unsuitable for deciding whether a reset address is valid. Submission, delivery events, bounce classification, and an actual completed reset answer different questions.

Domain authentication belongs in the trial as well. SPF authorizes sending infrastructure, DKIM attaches a validated signing identity, and DMARC evaluates identifier alignment and publishes handling policy. The practical test is to inspect messages sent from the real custom domain and verify alignment, rather than stopping when a setup screen displays a green check. DMARC also supports staged policy application through its `pct` value, expressed from 0 through 100, but rollout policy should be owned alongside the domain rather than buried in an SDK adapter.

One detail is easy to miss: DMARC evaluation concerns the domain visible to the recipient and its alignment with authenticated identifiers. Passing an isolated SPF or DKIM check does not, by itself, establish DMARC alignment. Test the final message shape.

Test the message, not the badge.

## What changes at US and EU scale

At small scale, one suppression table and one event consumer are enough. At larger scale, I would separate global invalid-recipient state from regional event processing, then document which fields cross a regional boundary. An email address, event timestamp, transport identifier, and diagnostic text do not all need the same retention period. Decide that schema before selecting a data-region toggle.

The transport evaluation should use a fixed harness. Measure time from a clean checkout to the first authenticated custom-domain submission; record the manual DNS steps; inject a known-invalid test recipient through a permitted test path; verify how quickly the suppression state becomes visible; replay the same event; and confirm that a suppressed retry never reaches the adapter. Benchmarks need the same message, region, and concurrency or they are theater.

The primary comparison is operational ownership:

| Decision | Application-owned | Console-owned |
|---|---|---|
| Template review | Travels with code review and release history | Travels with separate account roles and audit history |
| Variable contract | Can be type-checked before submission | Must be synchronized with remote template state |
| Provider migration | Renderer remains; adapter changes | Templates must be exported or rebuilt |
| Non-engineer edits | Follow the application release path | Can follow a content-specific workflow |

This table does not make application ownership universally correct. It makes the cost visible. For a password-reset flow, I want code review, repeatable local rendering, and a small adapter more than I want instant console edits.

Finally, define failure policy before load testing. Temporary transport failure may be retried with a bounded policy. An invalid recipient should update suppression. An unauthenticated or malformed delivery event should be rejected and observed. A user-facing reset endpoint should not reveal whether an account exists through different responses. These are application rules, so they should not depend on whichever SDK happens to be installed.

The selection rule is compact: choose the transport whose custom-domain authentication and event model pass the harness while allowing the repository to remain the source of truth for reset templates. Everything else is adapter work. Keep it small.

## Sources

References:

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
