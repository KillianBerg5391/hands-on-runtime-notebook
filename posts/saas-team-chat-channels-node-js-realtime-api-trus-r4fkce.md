# SaaS Team Chat Channels — Node.js Realtime API Trust Boundaries Explained

The hard choice in a property-management SaaS team chat and realtime API is not latency. It is deciding what a connected browser is allowed to know and do during a live poll. Give the browser a broad channel token and the poll is easy to wire, but a copied token can expose another building's vote. Make the token narrow and the reconnect path needs more deliberate state handling. I would choose the narrow token, keep the vote authoritative on the server, and treat the realtime connection as delivery rather than truth.

TL;DR: issue a short-lived token scoped to one poll and one participant role; accept votes through an authenticated server command; publish only the events that role can receive; store the final tally outside the socket. A Node.js client can then reconnect without inventing a second voting system.

## How should a SaaS team secure realtime chat channels through an API?

Consider a manager running a poll for 48 apartments about a Saturday maintenance window. The browser needs the poll question, current phase, and permitted choices. It does not need a general property channel, a list of private tenant messages, or the ability to declare the poll closed.

That boundary is easy to blur because a channel name looks like authorization. It is not. A string such as `property:17:poll:maintenance` identifies a routing destination. The server still has to decide whether the caller may join it, receive an event, submit a vote, or request the result. Those are separate capabilities, and combining them into one token is convenient until the token travels through a log, a browser extension, or a copied URL.

The useful decision matrix is small:

| Design choice | Narrow poll token | Broad property token |
| --- | --- | --- |
| Client scope | One poll, role, and participant | Several channels or actions |
| Reconnect work | Reissue or refresh scoped access | Reuse a wider credential |
| Blast radius | Limited to the poll boundary | Includes unrelated property data |
| Server complexity | More checks at issuance and join | Fewer checks at the edge, more trust in clients |

The recommendation is the first column. It costs a little orchestration. That is a fair trade for a poll where tenant identity and building boundaries matter.

## Where should the trust boundary sit?

The browser is a user interface. It is not the ballot box. A client can disable a button, but it cannot make a vote valid. The server should validate the participant, poll state, choice, and duplicate-vote rule before it records anything or broadcasts a receipt.

The WebRTC specification is useful context here because it makes the split between connection setup and application behavior explicit: a realtime transport can carry application data, but application code still defines the meaning and permissions of that data. The same principle applies when the implementation uses a WebSocket-style channel. Transport success is not authorization success.

For a live poll, I would model the command and event separately. The command asks to vote. The event confirms the accepted result. A rejected command should not be disguised as a successful update just because the socket is open.

```ts
type PollCommand = {
  kind: "cast_vote";
  pollId: string;
  choiceId: string;
  requestId: string;
};

type PollEvent =
  | { kind: "poll_snapshot"; pollId: string; phase: "open" | "closed"; choices: string[] }
  | { kind: "vote_accepted"; pollId: string; requestId: string }
  | { kind: "vote_rejected"; pollId: string; requestId: string; reason: "forbidden" | "closed" | "invalid_choice" };

function canReceivePoll(role: "tenant" | "manager", pollId: string, scopedPollId: string) {
  return role === "tenant" && pollId === scopedPollId;
}
```

The type is not a security control. It is a reminder of the contract. The actual check belongs on the server, next to the data lookup and transaction that enforce it.

## How should Node.js handle reconnects without changing the ballot?

A dropped connection is normal. The mistake is treating reconnect as a new vote session. Keep a durable history record keyed by `pollId`, participant identity, and the server's duplicate-vote policy. Give each command a `requestId`, and make the write idempotent for that identity. Then a client can retry after an ambiguous network response without creating a second accepted vote.

On reconnect, issue a fresh scoped credential and send a snapshot before incremental events. The snapshot must describe the current poll phase and the participant's accepted state. Do not ask the client to reconstruct truth from whatever messages happened to arrive before the disconnect.

A small server boundary can make the intended order visible:

```ts
type VoteStore = {
  accept(input: { pollId: string; participantId: string; choiceId: string; requestId: string }): Promise<"accepted" | "duplicate">;
  snapshot(input: { pollId: string; participantId: string }): Promise<{ phase: "open" | "closed"; choices: string[]; voted: boolean }>;
};

async function handleVote(
  command: PollCommand,
  auth: { participantId: string; scopedPollId: string; role: "tenant" | "manager" },
  store: VoteStore,
) {
  if (!canReceivePoll(auth.role, command.pollId, auth.scopedPollId)) {
    return { kind: "vote_rejected" as const, pollId: command.pollId, requestId: command.requestId, reason: "forbidden" as const };
  }

  const result = await store.accept({
    pollId: command.pollId,
    participantId: auth.participantId,
    choiceId: command.choiceId,
    requestId: command.requestId,
  });

  return { kind: "vote_accepted" as const, pollId: command.pollId, requestId: command.requestId, result };
}
```

The example deliberately leaves storage and transport unspecified. That is the point. The poll's correctness should survive a channel replacement.

## What should you measure before shipping?

Measure the moments users experience, not just socket uptime: time from join to the first valid snapshot, command acceptance latency, reconnect-to-snapshot time, rejected commands by reason, and the gap between a stored vote and its broadcast receipt. Keep a counter for duplicate `requestId` values. A rising duplicate count may mean retries are working; it may also mean the client is timing out too aggressively. The metric needs the surrounding context.

Test the uncomfortable paths with a fake transport: join with a token for another poll, submit after closure, resend the same request, reconnect after the write but before the receipt, and let a manager view the tally while a tenant cannot. These tests are more useful than a happy-path demo with two tabs.

A token expiry test belongs here too. The client should be able to distinguish an expired credential from a rejected vote. It should request a new scoped credential, reconnect, receive a snapshot, and leave the durable vote unchanged. Avoid silently retrying a command with a newly minted token unless the command's idempotency key is preserved.

Version the event envelope before multiple clients depend on it. A `v1` marker is cheap; changing `choiceId` or the meaning of `phase` after a mobile client ships is not. This is a practical history concern, not decoration.

## When is a broader channel defensible?

A broader channel can be reasonable for a trusted internal dashboard that only reads aggregate state, especially when its access is issued to a server-controlled process rather than an arbitrary tenant browser. It still needs explicit read permissions and an expiry policy.

It is a poor default for a mixed property channel. Tenant messages, manager controls, poll state, and audit events have different audiences. Bundling them reduces initial wiring, then creates a permission matrix inside every client. That is configuration bloat with a security bill attached.

The limitation is operational as well as technical. A tiny internal dashboard that only reads an aggregate may not need per-tenant poll credentials, and the extra issuance flow can be needless machinery there. Use a simpler read-only channel when the audience is fixed, the data is already aggregated, and no browser can submit a decision. The trade-off is deliberate: the narrower boundary is for mixed-trust SaaS traffic, not a universal ritual.

The durable rule is simple: scope the credential to the smallest useful poll boundary, keep the authoritative write behind a server check, and make reconnect restore state from durable storage. Realtime delivery becomes replaceable plumbing. The decision remains legible when the property team adds a second building, a manager-only action, or a new client.

## Sources

- https://www.w3.org/TR/webrtc/
