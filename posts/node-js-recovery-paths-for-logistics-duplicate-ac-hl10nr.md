# Node.js Recovery Paths for Logistics Duplicate Account Identity Resolution

Short answer: treat email lookup as a candidate finder, then use identity resolution and a reviewed recovery path to decide whether two logistics accounts are the same person. Never merge records because two rows share a normalized email or phone number.

The useful unit is not a clever matching score. It is a traceable decision: which signals agreed, which contradicted each other, who approved the merge, and how the original accounts can be restored. That is the difference between cleaning duplicate accounts and quietly moving a driver's shipments into somebody else's login.

## Why duplicate accounts appear in a logistics login flow

Logistics apps create identity collisions at awkward boundaries. A dispatcher may invite a driver with a work email, then the driver signs in with a phone one-time code on a personal device. A carrier can change domains after a contract, while an old email remains attached to completed delivery records. Shared warehouse phones add another wrinkle: the number is real, but it is not a person identifier.

Exact email lookup is still useful. It is fast, explainable, and a good first index. It is not proof. Normalize case and surrounding whitespace, but preserve the original value for display and audit. Do not remove plus-addressing or punctuation unless your mail provider's rules make that transformation explicit; an over-aggressive normalizer creates false matches.

I once started a cleanup script with `LOWER(email)` as the whole rule. The first export looked excellent: 312 collisions disappeared. Then a warehouse alias mapped three operators to one account. The rollback was more work than the query. That is the trap.

Phone numbers need the same discipline. Store an E.164 representation when the user supplies a country context, record how that context was chosen, and keep an “unknown country” state instead of guessing. A one-time code proves control of a channel at that moment. It does not prove that the channel owner is the same person who owns an older email account.

## How should identity resolution and email lookup trace duplicate accounts?

Use a two-stage trace. Stage one retrieves candidates with exact email and phone indexes. Stage two evaluates independent evidence and sends ambiguous cases to recovery review. The trace should be append-only, with a reason code rather than an opaque score alone.

| Signal | What it can establish | What it cannot establish |
| --- | --- | --- |
| Exact normalized email | A likely shared mailbox or reused address | One human owner |
| Verified phone OTP | Control of a phone channel now | Historical ownership or employment |
| Organization and role | A plausible account relationship | That two people are interchangeable |
| Device or session history | Continuity worth investigating | Identity by itself |
| Recovery evidence | A basis for a human-reviewed merge | Permission to skip authorization |

Keep the result boring: `candidate`, `confirmed_duplicate`, `related_accounts`, or `no_match`. A score can prioritize a queue, but the state transition should name the evidence and the actor. OWASP's authentication guidance is clear about treating authentication and recovery as security-sensitive flows, not as a convenience lookup.

Here is a small Node.js example that keeps candidate retrieval separate from the merge decision. The endpoint names are application routes, not a claim that a provider has a matching API.

```ts
type Account = {
  id: string;
  emailOriginal: string | null;
  emailNormalized: string | null;
  phoneE164: string | null;
  orgId: string | null;
  recoveryState: "ready" | "review" | "locked";
};

type Trace = {
  candidateId: string;
  signals: string[];
  decision: "candidate" | "confirmed_duplicate" | "related_accounts" | "no_match";
};

function traceCandidate(source: Account, candidate: Account): Trace {
  const signals: string[] = [];
  if (source.emailNormalized && source.emailNormalized === candidate.emailNormalized) {
    signals.push("exact_email");
  }
  if (source.phoneE164 && source.phoneE164 === candidate.phoneE164) {
    signals.push("same_phone");
  }
  if (source.orgId && source.orgId === candidate.orgId) {
    signals.push("same_organization");
  }

  const independent = signals.filter((signal) => signal !== "same_organization");
  const decision = independent.length >= 2
    ? "candidate"
    : signals.includes("exact_email")
      ? "related_accounts"
      : "no_match";

  return { candidateId: candidate.id, signals, decision };
}
```

The important detail is what the function refuses to do: it never changes ownership. A separate recovery transaction must require a fresh OTP, an authenticated session, authorization to act on the organization, and an audit event that records the old and new account IDs. Add an idempotency key so a retried merge cannot apply twice.

## The recovery path is the product decision

For a logistics app, recovery should preserve operational continuity without making identity weaker. Let a driver prove control of the new phone, then ask for a second factor that is independent of that phone: an existing session on a trusted device, a dispatcher-approved invitation, or a documented support review. Which option is available depends on your threat model and employment model.

Do not send a magic “merge now” link to both email addresses and call that recovery. Links leak through forwarded mail, and an old mailbox may belong to a former contractor. Use short-lived, single-use artifacts, rate-limit attempts, and show the user which account data will be retained before committing. Recovery should also have a lockout and escalation route; a permanently blocked driver at a loading dock is an availability incident, but an instant merge is an account-takeover opportunity.

The catch is that high-assurance recovery adds queue time. If deliveries cannot wait, keep the accounts separate and grant a narrowly scoped organization invitation while review is pending. That is less elegant than a merge, yet it preserves shipment history and limits blast radius.

Keep the queue visible.

Consider a concrete night-shift case. A driver signs in with a phone OTP from a number that used to belong to another contractor. Email lookup finds an old account with the same normalized address, while the current account has three completed routes and the old account has a pending invoice. The tempting fix is to copy the current phone onto the old row and delete the newer row. That sequence loses the provenance of the route records and can make the invoice visible to the wrong organization. The safer flow creates a recovery case containing both immutable IDs, snapshots the matching signals, and asks the driver to authenticate the current session again. A dispatcher who owns the carrier organization can then verify the invitation and invoice relationship without receiving the driver's OTP. If the dispatcher cannot establish that relationship, the case remains `related_accounts`; support can still see both histories, but authorization stays attached to the original records. If the driver later proves control of the old mailbox through a separate, single-use challenge, the reviewer can approve a field-by-field merge. Each step is replayable in the audit log, and an idempotency key makes a duplicate approval harmless. This is slower than a SQL update, but the delay is a deliberate control around manifests and proof-of-delivery data.

Small pause.

## Testing the ugly cases before production

Test the matrix, not just the happy path. Include a shared warehouse number, a recycled phone number, an email alias with different punctuation, a driver who loses the old device, and two accounts in different organizations with the same contact details. Assert that every path emits an audit record and that retries are idempotent.

Property-based tests are useful here: generate pairs of accounts, then assert that changing one signal cannot silently upgrade a `related_accounts` result to `confirmed_duplicate`. Run the same corpus against the database query and the Node.js resolver so normalization drift is visible.

Operationally, measure time-to-first-review, false-merge reversals, OTP delivery latency, and the percentage of candidates that remain unresolved after a day. I benchmark these because a resolver that is accurate but leaves 40% of drivers waiting is still a failed login system. Your mileage may vary by carrier policy and regional phone coverage; I’m not sure one threshold transfers cleanly between fleets.

## When should you keep accounts separate?

Keep separate records when the evidence only says “same contact channel,” when organization ownership conflicts, or when the person cannot complete an independent recovery step. A related-account link can give support staff context without collapsing authorization boundaries.

Choose a merge only when the user has authenticated both sides or an authorized reviewer has verified the recovery evidence. Preserve immutable source IDs, move only explicitly approved fields, and provide a reversible tombstone rather than deleting the losing account. The right outcome is sometimes no merge.

That rule feels conservative because it is. A duplicate account costs a support ticket; a mistaken merge can expose manifests, addresses, and proof-of-delivery records.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc3966
- https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API
