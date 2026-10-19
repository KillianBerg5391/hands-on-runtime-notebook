# Seller Alerts: Build SaaS Event Emails with Owned Templates and DKIM

Short answer: build SaaS event alert emails around an application-owned new-order template, a narrow transport interface, and a custom sending domain that must pass DKIM verification before production delivery. The deciding constraint is ownership. A marketplace changes order fields and seller policy more often than it should change mail transport, so letting a delivery dashboard become the source of truth creates needless config and a second deployment path.

This split gives the team one reviewable artifact for subject lines, escaping, localization, and order data. It also keeps DKIM where it belongs: at the domain and delivery boundary, rather than tangled into template rendering. Authentication does not rescue a misleading message, and a clean template does not authenticate its sender. Both have to pass.

No dashboard fixes that.

## How should Node.js build SaaS event alert emails?

A new-order alert is part of the marketplace's behavior. The seller needs the order identifier, item summary, total, destination context, and a stable link back to the marketplace. Those fields already have meanings in application code. Moving the final message into a remote editor duplicates that schema in configuration, where a renamed property can survive code review and fail only after an order arrives.

Make the template input deliberately smaller than the order model. Do not hand a renderer a database row or an arbitrary event payload. Map the event into a versioned notification contract first. Product code then owns what the message says; the transport owns how the resulting bytes leave the system.

That boundary matters.

This boundary is easy to benchmark. Measure event-to-render time, render failures, queue delay, and accepted versus rejected handoffs separately. Open rate is a poor correctness signal because Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened a message. A seller clicking through may still be useful product telemetry, but an apparent open should not mark an order notification as delivered or read.

Tiny contracts win.

## The smallest implementation worth shipping

The code has no remote template ID and no delivery-specific fields in the order event. It escapes untrusted text, emits a plain-text alternative, and gives the transport an idempotency key. An SMTP relay or an email API can implement the interface without changing the template contract.

```ts
type NewOrderNotice = {
  schemaVersion: 1;
  orderId: string;
  sellerEmail: string;
  shopName: string;
  itemCount: number;
  total: string;
  orderUrl: string;
};

type Message = {
  from: string;
  to: string;
  subject: string;
  text: string;
  html: string;
  idempotencyKey: string;
};

interface MailTransport {
  send(message: Message): Promise<{ accepted: boolean; messageId: string }>;
}

const escapeHtml = (value: string): string =>
  value.replace(/[&<>\"]/g, (character) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
  })[character] as string);

function renderNewOrder(input: NewOrderNotice): Omit<Message, "from"> {
  if (input.schemaVersion !== 1) throw new Error("Unsupported notice schema");
  if (input.itemCount < 1) throw new Error("An order must contain an item");

  const shop = escapeHtml(input.shopName);
  const order = escapeHtml(input.orderId);
  const total = escapeHtml(input.total);
  const url = new URL(input.orderUrl);

  return {
    to: input.sellerEmail,
    subject: `New order ${input.orderId}`,
    text: `${input.shopName}, you have a new order (${input.itemCount} items, ${input.total}). Review it: ${url.href}`,
    html: `<p>${shop}, you have a new order.</p><p>Order ${order}: ${input.itemCount} items, ${total}</p><p><a href="${url.href}">Review order</a></p>`,
    idempotencyKey: `new-order:${input.orderId}:v1`,
  };
}

async function notifySeller(transport: MailTransport, input: NewOrderNotice): Promise<void> {
  const result = await transport.send({
    from: "orders@notify.marketplace.example",
    ...renderNewOrder(input),
  });
  if (!result.accepted) throw new Error("Message handoff was rejected");
}
```

A successful handoff does not prove inbox placement. Nor should bounce processing run inside the request that created the order. Queue the notice after the order transaction commits, then let a worker render and send it. Record the order ID, template version, attempt number, transport message ID, and outcome. Never log the rendered body; it can contain seller or buyer data.

Test three layers. Snapshot the text and HTML for contract changes. Run the renderer against hostile shop names such as `<script>` and quotes. Finally, deliver fixtures through the real sending domain to controlled inboxes and inspect authentication results and visible content. A mock transport proves only that an object crossed an interface.

The limitation is clear: repository ownership is a poor fit when non-engineers must publish urgent copy changes without an application release. In that case, use an editorial system that produces an immutable, versioned artifact, validate that artifact against the same input contract, and pin its version in every queued notification. This adds a publishing pipeline and preview tooling. It buys independent copy releases without turning a mutable remote name into production state.

## Domain verification is a release gate

Use a dedicated sending subdomain so operational ownership is obvious, then publish the authentication records required by the chosen transport. Google's sender guidance requires authentication for mail sent to personal Gmail accounts and sets additional requirements for bulk senders, including SPF, DKIM, DMARC, alignment, TLS, valid DNS, and low spam rates. Read the current thresholds in the guideline instead of copying them into a long-lived config comment; policy can change.

Treat verification as deploy state, not as a checkbox someone remembers. The release pipeline should query verified-domain state, or perform equivalent DNS validation, and block production sends until it passes. Keep selector and domain configuration outside application code. Rotate signing keys through infrastructure change control while the application continues to render the same `Message`.

The envelope, visible From address, and DKIM signing identity are separate pieces of the path. Their alignment matters for DMARC evaluation. Test the final production combination, including forwarding and replies, rather than proving only that a DNS record exists. Configure a monitored reply path when sellers can reasonably reply. A polished no-reply address is still a dead end.

Deliverability work is mostly operational hygiene: authenticate, send wanted transactional mail, process bounces, suppress addresses that should not receive another attempt, and watch complaint signals. Do not retry every rejection blindly. Classify temporary and permanent outcomes at the transport boundary, cap attempts, and preserve the idempotency key so a worker restart does not produce two alerts.

Retries have a cost.

## What changes at scale

The first scaling change should be a template test matrix, not a visual builder. Add locales, long item names, zero-width or right-to-left text where supported, large totals, and expired order links. Render every fixture in CI. Keep schema version 1 readable until all queued version 1 events have drained.

Next, use an outbox tied to the order transaction. The order write and outbox write succeed together; a separate worker claims the event and sends it. This closes the gap where checkout succeeds but the process exits before enqueueing the notification. It adds a table, worker, retention policy, and replay tooling. For an alert whose omission sends sellers hunting through dashboards, the machinery earns its keep.

The trade-off is operational weight. A young service with a low notification rate and a reconciler that can safely find unsent orders may defer the outbox; a marketplace that promises prompt seller action should not. Write that decision down, including the failure window accepted, instead of installing infrastructure by reflex.

Repository-owned templates give engineers atomic review with event-contract changes, deterministic tests, and transport portability. They also put copy edits on the code release path. If support or localization teams must publish copy independently, use a content workflow that exports a pinned, reviewable template artifact. The application should still validate its input schema and record the exact artifact version used. Do not resolve a mutable template by a friendly name at send time.

Watch four numbers together: queued notices, render failures, transport rejections, and terminal delivery outcomes. Add queue age as a distribution, not a lone average. The p95 can hide an old stranded message, so alert on the oldest ready item too. Twenty unlabeled counters are config bloat wearing an observability badge.

For a marketplace new-order email, correctness means more than calling `send()`: the committed order creates one durable notification intent, schema validation produces one reviewed message version, the verified custom domain signs it, and delivery outcomes feed a bounded retry or suppression decision. Build that chain first. Vendor selection comes later.

## References

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
