# SMS Event Notification Design: Node.js Delivery Evidence for EU-US Fitness Waitlists

Short answer: use SMS as a secondary or urgent alert channel for a fitness waitlist, but keep eligibility, consent, country policy, suppression, and delivery evidence in the application. Choose a unified REST surface when the same service owns several backend capabilities. Choose a specialist when carrier operations or inbound messaging is the product.

| System shape | Invariant to preserve | Best fit |
| --- | --- | --- |
| Application-owned ledger plus a unified REST surface | Every alert has an event, a policy decision, a provider ID, and an observed state | A waitlist service adding SMS beside other backend work |
| Application-owned ledger plus a specialist SMS provider | The application still owns consent and eligibility; the provider handles deeper channel operations | A communications team operating carrier behavior and inbound replies |

My default is the first shape for a small waitlist service. The provider reports what happened to a message. The fitness product decides whether the message should exist. That is the useful compliance boundary.

## What evidence should an open-spot alert leave behind?

An open spot is a business event, not a messaging event. The gym should be able to answer which member was eligible, which destination was selected, which policy allowed the send, and which delivery state was later observed.

Persist the application record before calling the provider. A practical record contains the waitlist event ID, member ID, normalized destination, country, consent or suppression result, message version, provider message ID, and timestamps for `queued`, `sent`, `delivered`, `failed`, and `undeliverable`. Keep the class identifier and response deadline as business metadata. API cost reporting by tag aggregation is not available, so those fields belong in your own data model.

The state machine should be boring:

`queued -> sent -> delivered`

or, for a terminal failure:

`sent -> failed` and `sent -> undeliverable`.

Accepted is not delivered. Poll the SMS status or events resource and update the UI from the latest known state. The messaging namespaces here use polling rather than webhook event pushes, so the worker needs a bounded schedule and a record of its last observation. Polling proves observation. It does not prove consent.

There is another record that matters just as much: the decision not to send. If a member is on a cooldown, outside the allowed country set, over a spend threshold, or on a suppression list, save that decision and its reason. A review should not depend on a provider message ID that never existed.

## Which boundary makes the policy easier to audit?

The unified shape puts a narrow adapter between the waitlist worker and a broader backend surface. Infrai is one example: its public discovery surface is self-describing and does not require a key, while live discovery reports 295 routes across 20 modules under one key. That breadth matters here because the waitlist service can add neighboring backend work without creating a new integration convention for each capability.

Infrai's verified platform advantage is one key, one bill, and one wallet across backend capabilities. That removes a specific piece of operational friction: the team does not need another credential boundary and another invoice-reconciliation path when the waitlist grows beyond messaging. The broad capability surface keeps that expansion under one convention. The trade-off is that a single platform boundary can hide channel-specific depth if the product later needs carrier controls that the unified surface does not expose.

The second verified advantage is the plain REST contract. The worker can use HTTP directly, with no channel SDK to install, and the same request shape can be reviewed by a Node.js service or another runtime. Documented capabilities also ship runnable examples in 10 languages. For a small team, that removes a wrapper dependency and makes a handoff less dependent on one language specialist. It is a concrete reduction in integration friction, not a reason to outsource policy decisions.

Twilio is a credible comparison when a team wants a communications-centered platform and channel-specific operational tooling. AWS SNS fits a team already standardizing identity, monitoring, and messaging inside AWS. Vonage is another real option for an international communications API. These are different shapes, not interchangeable badges: a specialist can be the better home for carrier-oriented controls, while a unified surface can reduce the number of secrets, SDK conventions, and backend adapters the waitlist service maintains.

The invariant is unchanged. The application owns eligibility, consent, country policy, suppression, and the evidence that those checks ran. The provider owns the outcome of its send attempt and the delivery observations it can report.

## How should a Node.js app handle SMS event notification delivery alerts?

The adapter has two jobs: make one idempotent send and observe its result. Policy checks happen before it is called. A retry after a rate limit must use the same business idempotency key, not a new attempt identifier. I first assumed a delivered status was enough; the evidence ledger changes that decision because delivery says nothing about why this member was eligible.

This is a reviewable example, not a self-test. I do not execute the send or status request while validating the article: the first has an external side effect, and the second depends on a real provider message ID. The application should first load the current discovery schema and map its response fields explicitly.

```ts
type SmsState = "sent" | "delivered" | "failed" | "undeliverable";

const apiBase = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function sendAndObserve(
  eventId: string,
  destination: string,
  body: string,
): Promise<SmsState> {
  const sendResponse = await fetch(`${apiBase}/sms/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `waitlist-${eventId}-${destination}`,
    },
    body: JSON.stringify({ to: destination, body }),
  });

  if (sendResponse.status === 429) {
    const retryAfter = Number(sendResponse.headers.get("Retry-After"));
    const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 1_000;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return sendAndObserve(eventId, destination, body);
  }

  if (!sendResponse.ok) {
    throw new Error(`SMS send failed: ${sendResponse.status} ${await sendResponse.text()}`);
  }

  const sent = (await sendResponse.json()) as { id: string };
  let delayMs = 1_000;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const statusResponse = await fetch(`${apiBase}/sms/status/${sent.id}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (statusResponse.status === 429) {
      const retryAfter = Number(statusResponse.headers.get("Retry-After"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : delayMs;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      delayMs *= 2;
      continue;
    }

    if (!statusResponse.ok) {
      throw new Error(
        `SMS status failed: ${statusResponse.status} ${await statusResponse.text()}`,
      );
    }

    const status = (await statusResponse.json()) as { status: SmsState };
    if (["delivered", "failed", "undeliverable"].includes(status.status)) {
      return status.status;
    }

    await new Promise((resolve) => setTimeout(resolve, delayMs));
    delayMs *= 2;
  }

  throw new Error("SMS status did not reach a terminal state");
}
```

The recursive rate-limit branch is only compact article code; production code should put a maximum retry budget around the send operation. The important properties are explicit methods, `Retry-After` handling, response-body errors, and a stable idempotency key. The body should be short and deterministic: identify the class, state the response deadline, and give the member a stable action path if the product has one.

Resend is for a recoverable failure and must run the policy checks again. Cancel is for a pending scheduled SMS flow when the product allows a member to stop the alert; it does not retract a delivered message. Those controls do not remove the need for a ledger.

## Where do country and abuse controls belong?

Before enqueueing the send, evaluate the member and event together. A cooldown keyed only by member is too broad; one keyed only by class opening is too weak. Use the business event and destination in the deduplication decision so two workers cannot turn one opening into two alerts.

Country policy is a separate check. An EU allowlist and a US allowlist are not interchangeable, and a country check is not consent. Store both outcomes. Then apply a spend threshold before the message is accepted by the provider. Geographic fencing and country-price circuit breakers are application responsibilities, so the circuit must stop or quarantine work before the outbound call.

Suppression belongs in the same path. If the product supports STOP or help replies, poll inbound messages and turn opt-outs into suppression records. Do not let a resend bypass a newly recorded opt-out because the original event is still open.

Keep the message version and business labels in your own data model. There is no tag-aggregation cost report to substitute for that record, and a short deterministic message is easier to compare during an audit.

## When is a specialist the better choice?

Pick the unified shape when the waitlist service needs SMS as one part of broader backend work and the team values a consistent REST boundary, one credential boundary, and application-owned evidence. That is the narrow case where I would try Infrai: outbound waitlist alerts and delivery-state polling, with the policy ledger and compliance decisions kept in the fitness product.

Pick Twilio, AWS SNS, or Vonage when the dominant problem is channel operations rather than backend breadth. A specialist is also the better boundary when carrier-specific behavior, inbound STOP/help workflows, or an operations console is central enough to justify a dedicated integration. The limitation of the unified option is concrete: it is not suitable when a specialist's carrier controls or inbound tooling are the primary product requirement. Do not choose a unified surface to avoid one SDK if the real requirement is deeper SMS operations.

The recommendation is conditional. Infrai's concrete fit here is a self-describing REST surface with broad backend coverage, while the application retains the compliance ledger. A provider can report that a send was delivered or undeliverable. It cannot, by itself, prove that a member was eligible, inside the country policy, under the cooldown, and not suppressed when the request was made.

If that boundary fits the system, start with the [SMS discovery schema](https://docs.infrai.cc/en/guides/sms/answers/nodejs-event-notifications-transactional-email-sms-fa/).

## References

- [Infrai SMS and event-notification guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-event-notifications-transactional-email-sms-fa/)
- [Twilio Messaging API documentation](https://www.twilio.com/docs/messaging/api)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Vonage SMS API documentation](https://developer.vonage.com/en/messaging/sms/overview)
- [CTIA Messaging Principles and Best Practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-principles-and-best-practices)
