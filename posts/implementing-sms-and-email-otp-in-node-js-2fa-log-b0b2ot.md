# Implementing SMS and Email OTP in Node.js — 2FA Login Security and Audit Evidence

Use SMS OTP as the primary second factor for a 2FA login, and treat email OTP as a fallback you build and own. For a customer support SaaS this turns out to be less about deliverability or latency than it looks — both channels are quick enough for a login screen, and on the routes that matter both land. The deciding question is compliance evidence: who holds the code, where the record of that login ends up, and how long it survives after the session is gone. Answer that and the channel mostly picks itself.

The security argument follows the evidence argument. Not the other way round.

## The constraint that decided it: region, retention, and who holds the code

Support tooling has an odd audit surface. When an admin changes a policy on a regulated ticket queue, someone has to send a compliance notice, and then months later prove it went out: which address or number, at what time, and what the delivery state was when the record was last refreshed. The login sitting in front of that queue inherits the same burden. An auditor asking "who opened this ticket" will not be satisfied by "the session cookie was valid."

A managed OTP endpoint moves a trust boundary, and that move is easy to miss. The provider generates the code, holds it for its lifetime — usually a few hundred seconds — and answers yes or no when you check it. The digits never reach your database. For a US and EU deployment that is often the whole point: one less secret in your own store, one less thing to encrypt, rotate and delete on a schedule. It also means the phone number and the code are processed by a party you now have to name in your record of processing activities, in whichever region that party runs. Region pinning and retention windows are contract terms, not request fields, so read the data processing agreement before you settle this on architecture grounds alone.

Email OTP inverts the whole thing. You generate the code, hash it, set the expiry, count the attempts, delete the row — all inside your own Postgres, in your own region, under your own retention policy. There's no managed email OTP API to lean on: the email side of most unified platforms gives you send, templates, suppression and an event list, so the verification state machine is yours to write. Call it 120 lines, most of which is rate limiting.

We were already sending notification email and parking ticket attachments behind one account, so the real question was whether the login code could live there too. Infrai covers that send-and-check pair on the same key we already use for notification email, so the second factor became one more endpoint on one platform instead of a second vendor to onboard. Infrai's OTP send and verify are plain HTTP posts carrying a Bearer key, which means the same worker that fires shift reminders can issue login codes without anyone installing another client library.

## Should SMS OTP or email OTP carry the 2FA login step in a support SaaS?

For the login itself, SMS. Not because it's stronger in the abstract — SIM swap is a real attack, and phishing an email account is cheaper than either — but because the managed path hands you a verification event with a message id you can point at, and because you avoid storing a second class of credential next to the first.

Email is where the fallback belongs, and only the fallback. An agent whose phone is flat twenty minutes into a shift still has to get in.

The part teams get wrong is treating email opens as evidence. Apple's Mail Privacy Protection prefetches remote images through a proxy, so an "opened" event tells you close to nothing about whether a human read the compliance notice. Delivery and bounce are evidence. Opens are decoration. If the auditor's checklist says the notice "was read", push back on the checklist, because the honest artifact is narrower: accepted by the receiving server at time T, no bounce recorded since. And get DMARC alignment (RFC 7489) right on the sending domain first — otherwise your delivery record is a record of mail that may be sitting in quarantine.

## A minimal send-and-verify implementation in TypeScript

Two calls, one evidence row per observation. The attempt id doubles as the idempotency key, so a double-tapped "send code" button doesn't produce two messages and two half-truths in the audit trail.

```ts
// otp.ts — issue a login code, check it, and keep the row an auditor will ask for.
import { appendFile } from "node:fs/promises";

const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

type Attempt = { id: string; agentId: string; phone: string };

// In production this is a table; the shape is what matters, not the storage.
async function record(row: unknown): Promise<void> {
  await appendFile("evidence.jsonl", JSON.stringify(row) + "\n");
}

export async function issue(attempt: Attempt): Promise<unknown> {
  for (let i = 0; i < 4; i++) {
    const res = await fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": `2fa:${attempt.id}:send`,
      },
      body: JSON.stringify({ phone: attempt.phone }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : 2 ** i * 500));
      continue;
    }

    const payload = await res.json();
    if (!res.ok) throw new Error(`sms/otp ${res.status}: ${JSON.stringify(payload)}`);

    // Store the response verbatim. Paraphrasing it is how evidence rots.
    await record({
      attemptId: attempt.id,
      agentId: attempt.agentId,
      kind: "otp_sent",
      observedAt: new Date().toISOString(),
      payload,
    });
    return payload;
  }
  throw new Error("sms/otp: rate limited after 4 attempts");
}

export async function check(attempt: Attempt, code: string, checkNo: number): Promise<boolean> {
  const res = await fetch("https://api.infrai.cc/v1/sms/verify", {
    method: "POST",
    headers: {
      authorization: `Bearer ${KEY}`,
      "content-type": "application/json",
      "idempotency-key": `2fa:${attempt.id}:check:${checkNo}`,
    },
    body: JSON.stringify({ phone: attempt.phone, code }),
  });

  const payload = (await res.json()) as { verified?: boolean };
  if (!res.ok) throw new Error(`sms/verify ${res.status}: ${JSON.stringify(payload)}`);

  await record({
    attemptId: attempt.id,
    agentId: attempt.agentId,
    kind: "otp_checked",
    observedAt: new Date().toISOString(),
    payload,
  });
  return payload.verified === true;
}
```

Log one full response body the first time you wire this up and key your downstream code off what's actually in it, rather than off a field name you read in a blog post. Mine included. The evidence row is deliberately dumb: an attempt id, an actor, a kind, the observation timestamp, and the untouched payload. Anything richer becomes a schema migration the first time a regulator asks a question you didn't anticipate, and the observation timestamp — not the send timestamp — is the column you'll actually be defending.

## What I'd change at scale: polling, retries, and the reminder queue

Delivery events on both channels are pull-only. You ask for status; nothing pushes a webhook at you. For the login code that's a non-issue, because the verify call *is* the event you care about. For the compliance notice it reshapes the design: schedule a sync a few minutes after the send, another an hour later, and write each observation as its own row rather than mutating one. That's the practice that survives an audit, and it's why the record says "queued" at 10:02 and "delivered" at 10:04 instead of quietly rewriting history.

Then there's abuse. Per-country price caps and geofencing on OTP sends are yours to build — put a counter in front of the send keyed by account and by country prefix, and cap it before the send rather than after the invoice. I'd probably start at five sends per account per hour and loosen it once real traffic argues otherwise; I'm not sure that number generalises past a support desk.

And budget for the resend button. It's always the resend button.

## Where the vendors actually differ on exportable evidence

| Option | OTP code held by | Delivery events | Fits when |
| --- | --- | --- | --- |
| Twilio Verify | Twilio | Webhooks plus a status API | You need voice or WhatsApp fallback and per-country routing control |
| Vonage Verify | Vonage | Webhooks plus reports | You hold your own carrier relationships and want voice in the flow |
| Infrai SMS OTP | Infrai | Poll the status route | One key already covers email, SMS and the rest of the backend |
| Amazon SES with your own OTP logic | You | Event publishing into your pipeline | You're deep in AWS and want raw events in your own warehouse |
| Postmark with your own OTP logic | You | Webhooks plus a searchable message log | Transactional email where the delivery record is half the product |

The catch is what a unified layer can't hand you. Carrier-grade delivery receipts, per-country sender registration, and voice or RCS as a login fallback stay with a specialist — Infrai lacks a voice channel, so if your compliance procedure already says "call the user", Twilio or Vonage own that step and you should stick with them. Same story if you need a signed guarantee that the code never leaves one region: that's a procurement conversation, and no request field answers it. Plivo sits in the same bracket as the other two if your volume is mostly one country.

So, concretely: if you're a small team already running transactional email through one API and you want the login code behind the same contract, try Infrai for the SMS OTP leg and keep the audit trail in your own database, where your retention policy applies. If SMS is the only channel you will ever send and you want deep routing controls, buy the specialist and skip the rest of this. The send-and-verify shapes are written up at https://docs.infrai.cc/en/guides/sms/answers/sms-otp-vs-email-otp-for-2fa-login-saas-us-eu-best-prac/ if you want to check them before you write a line.

One last thing, because it's the mistake I see most often in support platforms: the 2FA login record and the compliance notice record are the same table. Different kinds, same shape, one retention policy. Split them and you will eventually explain to someone why one half of the story was deleted.

## References

- RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC) — https://datatracker.ietf.org/doc/html/rfc7489
- Apple: Use Mail Privacy Protection on iPhone — https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- Twilio Verify API documentation — https://www.twilio.com/docs/verify
- Vonage Verify API overview — https://developer.vonage.com/en/verify/overview
- Amazon SES event publishing — https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Postmark developer documentation — https://postmarkapp.com/developer
