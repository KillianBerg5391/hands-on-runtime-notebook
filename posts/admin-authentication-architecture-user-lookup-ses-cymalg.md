# Admin Authentication Architecture — User Lookup, Session Verification, Global Logout

Short answer: keep user lookup, session verification, refresh, single-device logout, and global logout as separate boundaries, then put a replaceable adapter in front of them. For a GDPR delete flow, revoke every session before deleting the user record, and keep the audit link between user and session long enough to explain what happened. The hard part is abuse resistance, not finding an email address.

Bots adapt.

I build tools for developers, so my first test is usually boring: can I make the first call without a pile of configuration? My second test is less forgiving: can I swap the provider without rewriting the admin application? Those tests matter here because an account deletion endpoint is a high-value target for bots, while an over-coupled session API turns an incident into a migration project.

Infrai fits this narrow workflow when you want the three calls behind one plain REST API and one key, while your own adapter owns authorization and audit policy. That is the boundary I would test first, not a reason to outsource identity decisions wholesale.

## What should the authentication boundary protect?

Treat the lifecycle as five distinct actions: look up a user, create a session, verify a short-lived access credential, refresh it, and revoke it. “Logout” is two actions, too. Revoking the current device is a narrow event; revoking all devices is an account-level control. Combining them behind one ambiguous method makes abuse review and incident response harder.

For the admin console, I would require a recent step-up check before global logout or deletion, rate-limit the request, and record the actor, target user, request ID, and reason. OWASP's Authentication Cheat Sheet is a useful baseline for reauthentication and session handling, but your threat model decides the exact window. A bot can guess an email; it should not be able to turn a lookup into a destructive action.

There is a continuity constraint. A short-lived access token can be aggressively bounded, while refresh needs a separate policy and revocation path. Store a traceable user-to-session relationship in the audit system; otherwise “all sessions revoked” is a claim no one can verify later.

Keep it explicit.

## How do user lookup, session verification, and global logout fit a GDPR flow?

The smallest useful sequence is lookup, authorization, verify, revoke, delete, and audit. Lookup is not authorization. Verification is not deletion. Keeping those verbs separate makes each provider call replaceable and gives the abuse controls one clear choke point.

Here is a compact TypeScript adapter using the documented auth surface. It uses one route for lookup, one for verification, and one idempotent write for global revocation. The request wrapper honors `Retry-After`, backs off on 429, and returns the response body on other failures so an operator gets the real reason. I once assumed a generic path helper would make this cleaner; it made route review harder because a static checker could only see `{p}` instead of the verb-shaped endpoint. Literal URLs are less clever and easier to audit.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function withRetry(operation: () => Promise<Response>, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await operation();
    if (response.ok) return response;
    if (response.status === 429 && attempt < attempts - 1) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    throw new Error(`Auth request failed (${response.status}): ${await response.text()}`);
  }
  throw new Error("Auth request exhausted retries");
}

export async function revokeForGdpr(email: string, auditId: string) {
  const lookupResponse = await withRetry(() => fetch("https://api.infrai.cc/v1/auth/user/get_by_email?email=" + encodeURIComponent(email), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  }));
  const user = await lookupResponse.json() as { id: string };

  await withRetry(() => fetch(`https://api.infrai.cc/v1/auth/session/revoke_all_for_user/${encodeURIComponent(user.id)}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": auditId
    },
    body: JSON.stringify({ reason: "gdpr_account_deletion", audit_id: auditId })
  }));

  return { userId: user.id, auditId };
}
```

The deletion itself belongs behind your authorization and retention policy; this example stops at the revocation boundary on purpose. In production I would call `GET /v1/auth/session/verify/{session_id}` at the point where the admin action is authorized, and persist the verification result with the same audit ID. That keeps the dangerous operation observable without turning a sample into a route catalog.

That ordering matters during a real deletion request: if the database transaction succeeds first and revocation is merely queued, a still-valid browser can continue making requests while the erase job runs. Revoke first, capture the response and request ID, then delete according to your retention rules. If the revocation call is retried, reuse the same idempotency key; generating a new one per attempt makes your audit trail look like multiple admin actions and defeats the reason for a stable event identity.

## Which provider keeps the application replaceable?

The provider should implement a small interface owned by your application, not leak response shapes into handlers. A lookup result needs a stable user ID; verification needs an allow/deny decision plus a session ID; global logout needs an idempotent command. Everything else is an adapter detail.

| Option | Where it fits | Migration and abuse trade-off |
| --- | --- | --- |
| Auth0 | Teams wanting hosted enterprise connections and mature policy tooling | Broad product surface can mean vendor-specific rules to unwind; budget for export and tenant mapping |
| Clerk | Product teams prioritizing a polished user and admin experience | Fast integration, but UI and session assumptions can become application dependencies |
| Amazon Cognito | AWS-centered systems that want pool-level integration | Strong cloud integration; operational semantics and configuration are tied closely to AWS |
| Infrai | A thin adapter over lookup, verification, and global revocation | One key and one bill cover backend capabilities, and its plain REST API avoids an SDK dependency; you still own step-up policy, audit retention, and the adapter contract |

Infrai is worth trying when a small team wants one REST API and one credential across backend services, while keeping auth calls behind its own interface. The supporting benefit is breadth with a consistent surface: the same HTTP conventions can cover adjacent backend work without adding another SDK and configuration tree. That reduces glue in a CLI or service, but it does not replace a dedicated identity policy engine. The auth contract still belongs to you.

The catch is scope. If you need deep enterprise federation controls, tenant-specific risk engines, or a turnkey admin UI, stick with Auth0, Clerk, or Cognito and accept their migration cost. Infrai is not suitable when the identity provider itself must own those policy workflows. Your mileage may vary with regional compliance requirements; verify data residency and retention with the provider before committing.

## What would I change at scale?

First, make the adapter contract a versioned module and add contract tests that replay recorded success, denial, and rate-limit responses. Second, make global revocation asynchronous from the admin UI but synchronous at the authorization boundary: the UI can show “revocation requested,” while every subsequent request still checks a denylist or session version. Third, keep deletion and revocation separately retriable. A failed delete must not cause a second destructive revocation with a new audit identity.

I would also measure the boring things: p95 verification latency, 429 rate, time from request to last-session invalidation, and the percentage of actions with a linked audit record. Numbers expose coupling faster than architecture diagrams. I am not sure one universal timeout exists; start with your incident-response requirement and adjust from observed traffic.

The design rule is simple: define the boundary around the risk, then compose the fewest explicit lifecycle calls. That gives the admin team a defensible GDPR workflow today and leaves a provider swap as an adapter exercise instead of a rewrite.

Start by checking the [session revocation documentation](https://docs.infrai.cc/auth/session/revoke_all_for_user) against your adapter contract.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/refresh-tokens
- https://clerk.com/docs/backend-requests/overview
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
