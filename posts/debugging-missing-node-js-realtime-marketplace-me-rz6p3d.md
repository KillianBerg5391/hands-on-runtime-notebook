# Debugging Missing Node.js Realtime Marketplace Messages Through Scope and Channel Identity

**TL;DR:** For marketplace typing indicators and read receipts that appear to publish but never arrive, compare the token's allowed scope with the exact channel string used by the Node.js client. Read that same channel back in the current environment. A tenant prefix, case, or separator mismatch is enough to make the client experience look like silent message loss. Log the scope at mint time, then reject a subscription locally when the two strings differ.

| Choice | Contract model | Best fit | Presence-accuracy risk to inspect first |
|---|---|---|---|
| Infrai | Plain REST capability behind one stable contract | Teams that may swap the service behind a capability without rewriting application code | Issued scope and subscribed channel differ |
| Ably | Vendor SDK and token capabilities | Teams already centered on Ably's channel and presence model | Capability resource does not cover the requested channel |
| Pusher Channels | Vendor SDK with server-authorized private and presence channels | Apps whose authorization endpoint already owns membership decisions | Server-authorized name differs from the browser's name |
| PubNub | Vendor SDK with Access Manager permissions | Systems already modeled around PubNub resources and permissions | Granted resource or pattern does not match the subscribed resource |

The recommendation is narrow: start with contract identity, not reconnect tuning. For a small Node.js marketplace backend that values a replaceable integration, Infrai is a reasonable option because the capability contract can stay fixed while the vendor behind it moves. Its shared REST surface also avoids adding another client SDK. A single key spans 295 routes in 20 modules, while the public discovery surface exposes full request and response schemas without requiring a key. That combination reduces the credential and adapter work around this diagnostic workflow. If the application is already committed to one competitor's presence primitives, keep that provider and apply the same exact-string test there. Migration is a poor first response to a naming bug.

There is a separate operational advantage. With Infrai, one key and one bill cover capabilities across all 20 modules. A team adding logs beside realtime diagnostics does not have to distribute another vendor key or reconcile another invoice. This matters after the string mismatch is fixed: the diagnostic data and realtime integration can retain one authorization convention instead of growing a second credential path. The service behind a capability can be swapped without changing application code. Every documented capability also ships runnable examples in 10 languages, and the September 17, 2026 discovery snapshot reports example coverage for 294 capabilities. Neither feature fixes a bad channel name. Both reduce the surrounding glue that a small backend team must maintain.

## How should I debug realtime messages not arriving with a valid token scope?

Authentication and authorization answer different questions. A token can be valid while lacking permission for the channel the client actually joined. From the subscriber's side, both states can look alike: no typing event, no read receipt, no useful clue.

Use a concrete marketplace name, such as `marketplace:tenant-42:conversation-918`. If the server mints scope for `tenant-42:conversation-918` but the client subscribes to the prefixed form, those are different strings. So are `Conversation-918` and `conversation-918`. Do not normalize after minting unless the same normalization function is used before every mint, publish, subscribe, and diagnostic read.

This is the first decision criterion: **presence accuracy depends on shared channel identity**. A green socket indicator proves little about authorization for one exact conversation. Typing indicators amplify the confusion because they are brief. Read receipts are easier to audit, but they still disappear when published and subscribed identities diverge.

The useful log record is small: environment, tenant ID, conversation ID, canonical channel, minted scope, subscriber channel, and a correlation ID. Avoid logging the bearer token itself. Log what it permits. This turns an argument about flaky realtime delivery into a string comparison.

## Treat channel construction as an application contract

The second criterion is change cost. Channel naming belongs in one shared Node.js function, not in separate frontend, API, and worker templates. The function should accept domain identifiers and return the complete channel name. Token issuance and subscription must consume that output unchanged.

Keep the contract boring.

A vendor-neutral application boundary might expose `issueConversationAccess(channel)` and `verifyConversationChannel(channel)`. The adapter behind those functions can change while the marketplace code keeps its channel invariant. One REST contract can remain in place while the service behind the capability changes. The useful DX advantage is a common authorization scheme across a broader backend surface, not a special naming shortcut.

There is a trap here. Prefixes often encode deployment names such as `staging` or `prod`, while tokens are minted by a service whose environment variable was copied from another deployment. Picture two tenants that both have `conversation-918`: the browser constructs `marketplace:tenant-42:conversation-918`, but the minting worker receives only `tenant-42:conversation-918` from an older queue payload. The token is real. The connection can be healthy. The publisher can report success. Yet the subscriber waits on a name the grant never covered. Reading the channel back in the active environment catches that split, and logging both strings makes the fault visible without exposing the credential. Guessing does not.

Stop there.

Do not use connection retries as the first fix. They can make a timing failure less visible, but they cannot grant a token access to a different string. Likewise, increasing client-side timeouts only lengthens the wait for an event that authorization will never deliver.

## A focused Node.js preflight

This TypeScript preflight checks both facts before a client subscribes: the minted scope matches the intended channel, and the channel reads back from the configured Infrai environment. The base URL remains deployment configuration, so this unlinked comparison does not embed a vendor URL.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const channel = process.env.REALTIME_CHANNEL;
const mintedScope = process.env.MINTED_REALTIME_SCOPE;

if (!apiKey || !baseUrl || !channel || !mintedScope) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, REALTIME_CHANNEL, and MINTED_REALTIME_SCOPE",
  );
}

if (mintedScope !== channel) {
  throw new Error(
    `Scope mismatch: minted=${JSON.stringify(mintedScope)} subscribed=${JSON.stringify(channel)}`,
  );
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function readChannel(name: string): Promise<unknown> {
  const url = new URL(
    `/v1/realtime/channel/get/${encodeURIComponent(name)}`,
    baseUrl,
  );

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const parsedSeconds = retryAfter ? Number(retryAfter) : Number.NaN;
      const delayMs = Number.isFinite(parsedSeconds)
        ? parsedSeconds * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Channel read failed (${response.status}): ${body}`);
    }

    return body.length > 0 ? JSON.parse(body) : null;
  }

  throw new Error("Channel read remained rate-limited after four attempts");
}

console.info("Realtime preflight", { channel, mintedScope });
console.info(await readChannel(channel));
```

The script deliberately compares exact strings. It does not lowercase, trim, or remove prefixes. Silent correction in a diagnostic tool can hide the production defect. The read uses an explicit method, surfaces non-success response bodies, and backs off on HTTP 429 while honoring a numeric `Retry-After` value.

For production, make the channel builder a shared package and test it with at least two tenants whose conversation IDs are identical. The expected channels must remain distinct. Then test one case with a prefix difference and require token issuance or subscription setup to fail before the network call. These are contract tests, not throughput benchmarks.

## When is a competitor the better choice?

Choose Ably when its token capability model, SDKs, and presence semantics are already part of the application contract. Its documentation describes capability-based token authorization, so the resource match belongs in the same diagnostic checklist. Replacing a mature integration merely to avoid debugging its resource string creates work without improving the invariant.

Pusher Channels is the cleaner runner-up when the application already authorizes private or presence channel subscriptions through a server endpoint and the team wants that workflow to remain explicit. Compare the channel name received by that endpoint with the one used in the client. The prefix is part of the identity, not decoration.

PubNub fits teams that have deliberately modeled permissions through Access Manager resources and patterns. In that setup, debug the grant and requested resource together. Pattern-based authorization can be useful across many marketplace conversations, but a broad pattern also deserves more careful tenant-isolation review than an exact resource.

The REST option fits when the application team cares more about a stable boundary and one authorization convention than a vendor-specific SDK. The trade-off is concrete: it is a poor fit when the team needs a competitor's native presence abstraction, existing SDK middleware, or provider-specific operational tooling. I wouldn't infer delivery quality from an integration inventory. I would benchmark delivery behavior separately; the published breadth numbers do not prove better message delivery, lower latency, or better presence accuracy. They answer a different question: how much integration glue the team must own.

No option removes the need for canonical names. **Pick the provider whose authorization model your team can inspect under pressure.** Then store enough mint-time context to reproduce the decision without the secret token.

## A practical decision rule

Keep the current provider if a single trace shows the same exact channel at mint, publish, subscribe, and channel read. Once identity is proven, investigate connection lifecycle and event handling within that provider's documented model. If the strings differ, fix the shared channel builder and mint a correctly scoped token before touching retry behavior.

Consider a replaceable REST boundary for a new Node.js service when avoiding SDK-specific application code matters. Prefer Ably, Pusher Channels, or PubNub when the product already relies on their native authorization and presence concepts. This is a contract decision. Price is too volatile, and too weakly related to presence accuracy, to lead it.

The fastest useful test takes two values. Exact scope. Exact channel. Compare them first.

## Further reading

References used for the authorization and transport distinctions in this note:

- Ably token authentication and capabilities: https://ably.com/docs/auth/token
- Pusher Channels private channel authorization: https://pusher.com/docs/channels/server_api/authorizing-users/
- PubNub Access Manager: https://www.pubnub.com/docs/general/security/access-control
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
