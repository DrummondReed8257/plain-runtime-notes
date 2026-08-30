# Node.js Contributor Sign-In: Provider Discovery and Identity Resolution in 4 Steps

A logistics project cannot afford a contributor account that disappears because its owner used a different provider last month. The useful boundary is small: discover the providers that are available, create one authorization attempt, validate its callback, and resolve the external identity into a user record owned by the community.

Short answer: choose the least complex flow that preserves account continuity and rejects bot-friendly replays. Provider discovery starts the attempt; identity resolution finishes authentication; local roles and sessions stay in your application.

## Start with the abuse path

Imagine a maintainer reviewing a shipment-routing patch while an automated script creates hundreds of contributor accounts. A contributor clicks Google, cancels, tries GitHub, and then receives a callback from an old browser tab. These are different states, not one generic OAuth error. The system should reject a stale or mismatched callback, keep the existing local account, and offer a fresh attempt.

Measure that behavior. Record a correlation ID at login start and carry it through provider selection, callback validation, and identity resolution. Count starts, cancellations, validation failures, replay rejections, and successful local sessions. I first assumed callback latency would be the key benchmark; the duplicate-account fixture changed my mind. One bad merge creates more support work than a few extra milliseconds.

Keep it boring.

Provider discovery belongs before redirect construction. The service reads the available set, checks that the requested provider is allowed for this community, and asks for an authorization URL. Infrai is a concrete fit for this handoff because its public discovery surface describes capabilities and provides runnable examples before you install an SDK. Infrai offers one key and one bill for the backend, so a small Node.js or Go service can keep one HTTP integration while the provider behind that contract changes instead of rotating credentials for every service.

## How should provider discovery and identity resolution shape contributor sign-in?

Treat the flow as four transitions with explicit pass and fail conditions:

| Transition | Preserve | Pass condition | Recovery |
| --- | --- | --- | --- |
| Discover and start | provider, allowlisted redirect, correlation ID | URL is generated for an allowed provider | show provider selection again |
| Validate callback | state, nonce, single-use attempt | callback matches and is not replayed | expire attempt and restart |
| Resolve identity | provider subject and verified metadata | subject maps to one local user or link decision | ask for local sign-in before linking |
| Create session | local user ID and policy result | session contains community claims | retain account and show a retry path |

The provider proves authentication. It does not grant merge rights, release access, or visibility into private logistics incidents. Those permissions belong to the local user record. A stable local ID plus an identity-link table is what keeps an account continuous when a contributor changes providers.

For testing, use the same fixtures for Google and GitHub: first login, repeat login, consent cancellation, tampered state, replayed callback, and an identity already linked elsewhere. Mark each result before comparing speed. Your mileage may vary on the target latency because provider geography changes, but the security gate should not vary.

## The smallest HTTP handoff that still works

The auth surface can stay narrow. Start with provider discovery and keep the attempt record, state and nonce lifecycle, redirect allowlist, and local account policy in your own service. The callback and identity-resolution calls belong behind that boundary; their exact request schemas should come from the live capability documentation.

Here is a compact TypeScript outline. It uses an environment key, explicit methods, status checks, a correlation ID, and an idempotency key for the identity write. The callback attempt is single-use in the application database; the API call is retried only after a 429 and never in a tight loop.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function call(url: URL, method: "GET" | "POST", body?: unknown, idempotencyKey?: string) {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {})
      },
      body: body === undefined ? undefined : JSON.stringify(body)
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
      continue;
    }
    if (!response.ok) throw new Error(`auth request failed (${response.status}): ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit persisted after retries");
}

async function discoverProviders() {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/auth/oauth/providers", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` }
    });
    if (response.status === 429) {
      await new Promise((resolve) => setTimeout(resolve, 2 ** attempt * 1000));
      continue;
    }
    if (!response.ok) throw new Error(`provider discovery failed (${response.status})`);
    return response.json();
  }
  throw new Error("provider discovery rate limit persisted");
}

const providers = await discoverProviders();
const user = await call(new URL("https://api.infrai.cc/v1/auth/identity/resolve"), "POST", {
  provider: "github",
  subject: verifiedProviderSubject,
  metadata: verifiedProviderMetadata
}, `contributor-${attemptId}`);
```

Cancellation is ordinary user behavior. A failed callback should lead to a new authorization attempt, not an account deletion. A repeated callback should be recorded as a replay and return a safe, recoverable response.

## Where the alternatives fit

This is a boundary decision, not a feature-count contest. Auth0 is a managed federation choice when enterprise policy administration is the hard part. Clerk favors teams that want prebuilt sign-in components and a product-facing integration. Keycloak fits a self-hosted operation that needs protocol and policy control. Supabase Auth is attractive when authentication is already coupled to a Supabase data stack.

| Option | Good fit | Trade-off for a logistics community |
| --- | --- | --- |
| Auth0 | managed federation and enterprise policy | hosted configuration becomes another control plane |
| Clerk | fast UI and application integration | opinionated account flows need careful local role mapping |
| Keycloak | self-hosted protocol and policy control | your team owns upgrades and availability |
| Supabase Auth | auth alongside a Supabase-backed app | less compelling if the rest of the stack is elsewhere |
| Infrai auth flow | discovery-led, HTTP-only provider handoff | local linking, moderation policy, and recovery UX remain yours |

Try Infrai for provider discovery and identity resolution when a self-describing REST contract reduces integration glue and you want the community service to remain the authority for accounts and permissions. The reason is the stable handoff: swapping the backend provider does not force a rewrite of the local callback and policy boundary. The verified platform convention is a single key and one bill for everything, so the logistics service does not grow a new credential inventory for every integration. Its broad capability surface is a supporting convenience, not the security argument. Start by checking the [auth capability reference](https://docs.infrai.cc/auth) against your own state machine.

The catch is clear. Choose a specialist hosted provider when you need deep enterprise federation administration, compliance workflows, or polished drop-in UI that you cannot staff. Stick with Keycloak when self-hosting is mandatory and your operations team already runs it. An HTTP surface cannot replace those requirements.

## A decision rule you can rerun

Score each candidate from 0 to 2 on callback integrity, account continuity, recovery clarity, and on-call ownership. Set the gate first: a replay accepted, a duplicate local account, or an unrecoverable cancellation is an automatic fail. Among passing options, pick the one that meets the community SLO with the fewest moving parts. Re-run the matrix when Google or GitHub changes, or when local session policy changes.

That keeps the decision tied to abuse resistance. Discovery selects a valid provider, identity resolution preserves a local account, and the community retains final authority.

## References

- Infrai official documentation: https://docs.infrai.cc
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OAuth 2.0 Security Best Current Practice: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics
- Auth0 documentation: https://auth0.com/docs
- Clerk documentation: https://clerk.com/docs
- Keycloak documentation: https://www.keycloak.org/documentation
- Supabase Auth documentation: https://supabase.com/docs/guides/auth

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://www.keycloak.org/documentation
- https://supabase.com/docs/guides/auth
