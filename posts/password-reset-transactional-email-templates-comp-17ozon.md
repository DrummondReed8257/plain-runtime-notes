# Password Reset Transactional Email Templates: Compile Localization Before Marketplace Delivery

**Short answer:** compile each localized password-reset email template into a versioned HTML and text artifact before deployment, then send it through a narrow transactional transport adapter.

The highest-leverage choice is where a template becomes immutable. Compile localized subject, HTML, and plain text in the application build; send that artifact through a narrow transport adapter. This catches missing copy before a password-reset message reaches a user. For a property marketplace that also alerts a seller about a new order, the same contract prevents a catalog edit from changing mail already queued.

The trade-off is blunt: copy releases become application releases.

| Choice | Preview fidelity | Locale checks | Runtime dependency | Best fit |
|---|---|---|---|---|
| Build-owned compiled artifacts | Same renderer in preview and send | Exhaustive before deploy | Transport only | Security and order events |
| Runtime server rendering | High if versions stay pinned | Request-path validation | Renderer plus transport | Rapid copy iteration |
| Provider-hosted templates | Depends on revision discipline | Depends on remote workflow | Template store | Operations-owned campaigns |

**Recommendation:** own the source and compile it into versioned artifacts. Keep delivery behind a tiny interface. A password reset is a security workflow, not a miniature marketing campaign, and delivery reliability improves when rendering cannot drift between preview, queue, and retry.

## What transactional email template approach should a password reset use?

A browser preview route is convenient. It is also too forgiving. It proves that one locale, one payload, and the developer's current checkout can render. It does not prove that every supported locale has a subject, that the plain-text body contains the recovery link, or that the exact output approved in review is the output sent after deployment.

Make preview output boring: a directory of deterministic files. For each message kind and locale, emit 3 artifacts: subject, HTML, and text. Attach the rendered directory to CI. Reviewers can open the HTML, diff the text, and see the locale key in the filename. No dashboard archaeology.

Never put a real reset token into a checked-in preview.

Use a conspicuous placeholder such as `PREVIEW_ONLY_RESET_URL`, then assert that production rendering receives a nonempty, already-created recovery URL. The template should not know how tokens are generated, stored, or expired. Its job is presentation.

New-order mail exercises a different edge of the same contract. A seller name may be long, an apartment label may contain punctuation, and monetary display may vary by locale. Preview fixtures should cover those shapes without pretending that a successful render means successful delivery.

Rendering and transport are separate failure domains.

## Two gates matter more than template syntax

The first gate is **locale completeness**. Pick one canonical locale, define its message shape, and require every other locale to satisfy that shape at build time. Do not silently fall back field by field. A mixed-language password reset looks suspicious, and a missing seller-order field can hide the required action. If a locale is unsupported, choose the fallback once for the whole message and record that choice.

The second gate is **revision identity**. A queued job should carry `messageKind`, `locale`, `templateRevision`, and domain data. It should not carry arbitrary HTML assembled by a controller, nor should a retry look up an unspecified latest revision. Pinning the revision makes a retry explainable. It also lets logs answer a useful question: which template produced this message?

Do not log the reset URL, token, or rendered body.

Log identifiers and outcomes: message ID, template revision, requested locale, resolved locale, attempt number, and transport classification. This is enough to correlate a complaint without turning logs into a second credential store. For order notifications, use the order ID as correlation data, not as a substitute for an idempotency key.

I would benchmark this workflow on time-to-first-reviewed-message, not lines of configuration. Count the commands required to add one locale, generate every preview, and run assertions from a clean checkout. If that path needs a remote console, local code, and a hand-maintained mapping file, the glue has already won.

## A small TypeScript boundary

The useful abstraction is not a universal email SDK. It is a compiler with a strict input union and a transport that accepts a finished message. This example keeps password-reset and marketplace-order payloads distinct, resolves the locale once, and returns data that can be previewed or queued.

```ts
const copy = {
  en: {
    resetSubject: "Reset your password",
    resetIntro: "Use this link to choose a new password:",
    orderSubject: "A new marketplace order is ready",
    orderIntro: "Review the new order for your property.",
  },
  es: {
    resetSubject: "Restablece tu contrasena",
    resetIntro: "Usa este enlace para elegir una contrasena nueva:",
    orderSubject: "Hay un nuevo pedido en el mercado",
    orderIntro: "Revisa el nuevo pedido de tu propiedad.",
  },
} as const;

type Locale = keyof typeof copy;
type MessageInput =
  | { kind: "password-reset"; resetUrl: string }
  | { kind: "seller-new-order"; orderId: string; propertyLabel: string };

type CompiledMessage = {
  subject: string;
  html: string;
  text: string;
  locale: Locale;
  revision: string;
};

function resolveLocale(requested: string): Locale {
  return requested in copy ? (requested as Locale) : "en";
}

function compileMessage(requestedLocale: string, input: MessageInput): CompiledMessage {
  const locale = resolveLocale(requestedLocale);
  const words = copy[locale];

  if (input.kind === "password-reset") {
    return {
      subject: words.resetSubject,
      html: `<p>${words.resetIntro}</p><p><a href="${input.resetUrl}">${words.resetSubject}</a></p>`,
      text: `${words.resetIntro} ${input.resetUrl}`,
      locale,
      revision: "credential-mail-v3",
    };
  }

  return {
    subject: words.orderSubject,
    html: `<p>${words.orderIntro}</p><p>${input.propertyLabel} / ${input.orderId}</p>`,
    text: `${words.orderIntro} ${input.propertyLabel} / ${input.orderId}`,
    locale,
    revision: "seller-order-v2",
  };
}
```

Production code should escape untrusted values with a maintained templating library; a hand-rolled helper is config debt wearing a tiny hat.

The boundary is the point: template compilation returns a value, while transport accepts that value. An adapter can add credentials, request IDs, and timeout behavior without leaking those concerns into copy.

A focused test suite should fail on 4 conditions: a locale misses a canonical key, compilation throws for a fixture, HTML or text omits the intended action URL, or output changes without an approved snapshot update. Render every locale and both message kinds.

Two happy-path screenshots are not a suite.

## Delivery still needs its own state machine

An API acceptance response is not proof of inbox delivery. Model attempts explicitly. The queue record needs a stable message ID and template revision; the transport result needs a normalized category such as accepted, retryable, or terminal. Retry only the retryable class, with bounded backoff and jitter. The application should suppress a duplicate request using the stable message ID even if a worker restarts between the remote request and the local acknowledgement.

Keep reset requests non-enumerating at the product boundary: the outward response should not reveal whether an account exists. The mail pipeline can record an internal decision without placing account status in public logs. Rate limits belong around request creation and send attempts. Template logic cannot defend either boundary.

Measure each stage separately: compile failures, queue age, transport latency, accepted responses, later delivery events, bounces, and complaints. Do not collapse them into a single success counter. A pretty preview says nothing about queue age.

Acceptance is not delivery.

An accepted transport request says nothing about how a receiving system handled the message.

SMS is a separate channel, even if product copy calls both of them notifications. Consent, opt-out handling, message length, and carrier policy require their own model. Reusing the event and correlation ID is sensible; reusing HTML-oriented template assumptions is not. CTIA's messaging principles are a starting point for the US messaging context, but they do not turn an email template engine into an SMS compliance layer.

## When is the runner-up better?

Runtime server rendering is the runner-up when copy changes must ship independently of application releases and the team can pin renderer versions, validate every locale in CI, and retain the exact rendered revision with each queued job. It trades a simpler editorial release for more runtime surface. That can be reasonable. Measure it.

Provider-hosted templates fit when an operations team truly owns copy deployment and has a review, rollback, and revision-pinning process. The dangerous version is split ownership: engineers own variables, operations owns markup, and nobody owns the contract. A remote preview can look correct while a production payload omits a renamed variable. Contract fixtures should run against the revision about to be activated.

The decision rule is compact. Choose the approach that can reproduce one sent message from its revision and input metadata, validate all locales before activation, and retry without changing content. If an approach cannot do those three things, fast setup is noise.

## Sources and References

- https://www.rfc-editor.org/rfc/rfc5322
- https://www.rfc-editor.org/rfc/rfc8058
- https://html.spec.whatwg.org/multipage/syntax.html
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- https://resend.com/docs/introduction
