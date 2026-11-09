# Node.js Signup Templates — Email API Retry Backoff Without Losing Content Ownership

Choose application-owned templates for an edtech signup verification link when the same release must control the email copy and an attached PDF welcome sheet. TL;DR: render once, freeze the message inputs, and retry the same logical send with a stable idempotency key. Email and SMS event notifications need that discipline before rate-limit backoff can be useful.

The constraint is ownership. A welcome-sheet correction should have an identifiable template revision, and a retry should preserve it. Moving text into a provider dashboard can be a reasonable editorial choice, but it changes who can alter the message after the application has queued it. Decide that explicitly.

This note treats the PDF as onboarding material. The verification link belongs in the email body. A learner should not have to download a document to activate an account.

## How should email and SMS notifications handle API retry state?

For this build, the application owns the verification URL, email content, PDF inputs, and their revision identifiers. The transport owns submission. Keep that boundary small.

Consider a proposed signup event named `signup-0042`, with template revision `welcome-r3`. Those are example identifiers, not production observations. Before submission, store the rendered inputs and a logical delivery identifier together. If a worker receives a rate limit, it must reuse that identifier and those inputs. Re-rendering the newest welcome template during a retry would make one queued event mean two different messages; changing the idempotency key at the same time would also remove the mechanism intended to deduplicate the send.

Freeze the inputs.

This is where template ownership becomes operational. An application release can pin a revision and review its email and PDF changes together. A hosted template can instead give editorial staff direct control; that is useful when independent copy changes matter more than application release coupling. Neither arrangement removes the need to record which revision a queued signup selected. The bad middle ground is an application that thinks it owns the content while a second editing surface can change it silently.

For US/EU signup workflows, keep an email-to-SMS fallback decision in application state too. Reconcile email events and SMS status by polling. A pending email is not evidence that a text should be sent immediately, and SMS should not become a second verification policy with unrelated expiry rules. Destination restrictions and country-cost circuit breakers belong in the application's pre-send checks.

A verification link and an OTP are different flows. This design creates the link in the application; adding an email-code fallback would require application-owned code generation and verification. Scope this integration to direct REST email and SMS submission. Requirements for SMTP relay, voice, WhatsApp, RCS, or domestic-China compliance need their own evaluation.

## The smallest PDF-to-email handoff

Infrai combines one key and one bill across 295 routes in 20 modules, including this PDF-to-email handoff. It is a plain REST API: Node.js can call it with built-in HTTP support, without installing a vendor SDK or tracking a client-library release. A single API key covers PDF rendering and transactional email, with unified billing and a single invoice for both operations. That removes a second credential configuration from the welcome-sheet handoff, and the application can pass the rendering result into the email request without introducing a temporary bucket just to connect two vendors.

The public, self-describing discovery surface also exposes request and response schemas without a key. Check the attachment mapping there before building it. That is an interface benefit, not a measured latency result.

The example below makes the field mapping explicit because an attachment mapping is part of the integration contract. Supply the documented PDF-generation request as `PDF_REQUEST_JSON`, and the documented email-send request as `EMAIL_REQUEST_JSON`, including the signup verification link. Set `PDF_RESULT_PATH_JSON` to a JSON array locating the renderer's attachment-compatible output in its response; set `EMAIL_ATTACHMENT_PATH_JSON` to a JSON array locating the corresponding existing slot in the email request. These paths are configuration, not proposed API field names. Check both schemas before selecting them. A document URL, raw bytes, and an attachment object are different representations; this example performs no conversion.

Use Node.js with TypeScript execution support and save the code as `signup.ts`. Configure `API_BASE_URL` to the service's documented HTTPS v1 base, `INFRAI_API_KEY` to the server-side key, and `SIGNUP_DELIVERY_ID` to a persisted identifier unique to this logical message. The script reads those environment variables and exits with an error when configuration or a request fails.

```ts
import { setTimeout as pause } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const base = required("API_BASE_URL").replace(/\/$/, "");
if (!base.startsWith("https://") || !base.endsWith("/v1")) {
  throw new Error("API_BASE_URL must be an HTTPS v1 base URL");
}
const key = required("INFRAI_API_KEY");
const deliveryId = required("SIGNUP_DELIVERY_ID");

function pathFrom(name: string): string[] {
  const value: unknown = JSON.parse(required(name));
  if (!Array.isArray(value) || value.length === 0 ||
      !value.every(part => typeof part === "string" &&
        !["__proto__", "prototype", "constructor"].includes(part))) {
    throw new Error(`Invalid path in ${name}`);
  }
  return value;
}

function at(root: unknown, path: string[]): unknown {
  let value = root;
  for (const part of path) {
    if (typeof value !== "object" || value === null ||
        !Object.hasOwn(value, part)) {
      throw new Error(`Missing mapped field: ${part}`);
    }
    value = (value as Record<string, unknown>)[part];
  }
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const header = response.headers.get("Retry-After");
  let minimum = 0;
  if (header !== null) {
    const seconds = Number(header);
    const parsed = Number.isFinite(seconds)
      ? seconds * 1000 : Date.parse(header) - Date.now();
    if (Number.isFinite(parsed)) minimum = Math.max(0, parsed);
  }
  const backoff = 500 * 2 ** attempt + Math.random() * 250;
  return Math.max(minimum, backoff);
}

async function post(
  route: string, payload: unknown, operationId: string
): Promise<unknown> {
  const body = JSON.stringify(payload);
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(base + route, {
      method: "POST",
      redirect: "error",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId
      },
      body,
      signal: AbortSignal.timeout(15_000)
    });
    const responseBody = await response.text();
    if (response.ok) return JSON.parse(responseBody);
    const retryable = response.status === 429 || response.status >= 500;
    if (!retryable || attempt === 3) {
      throw new Error(`HTTP ${response.status}: ${responseBody}`);
    }
    const delay = retryDelay(response, attempt);
    if (delay > 60_000) {
      throw new Error(`Reschedule after ${Math.ceil(delay)} ms`);
    }
    await pause(delay);
  }
  throw new Error("Retry budget exhausted");
}

const pdfRequest: unknown = JSON.parse(required("PDF_REQUEST_JSON"));
const emailRequest: unknown = JSON.parse(required("EMAIL_REQUEST_JSON"));
const outputPath = pathFrom("PDF_RESULT_PATH_JSON");
const attachmentPath = pathFrom("EMAIL_ATTACHMENT_PATH_JSON");
const slot = attachmentPath[attachmentPath.length - 1];
const parent = at(emailRequest, attachmentPath.slice(0, -1));
at(emailRequest, attachmentPath);

const rendered = await post(
  "/pdf/generate", pdfRequest, `${deliveryId}:pdf`
);
(parent as Record<string, unknown>)[slot] = at(rendered, outputPath);
await post("/email/send", emailRequest, `${deliveryId}:email`);
```

The two requests use the same credentials and base URL. Only the operation suffix changes. Within each request's retry loop, both the serialized body and idempotency key remain fixed. Four attempts, a 500 ms initial backoff, and a 15-second request timeout are example application policies; they are not provider limits or benchmark results. A long Retry-After value returns control to the caller so a durable worker can reschedule without retrying early.

The script deliberately surfaces transport exceptions. A timeout leaves acceptance uncertain. A caller must preserve the delivery identifier and reconcile state before deciding what happens next; generating a fresh key because a response was lost defeats deduplication. Keep error bodies in restricted operational handling, since the sample surfaces the real response rather than pretending every request succeeded.

The field mapping is the setup cost. Reject this approach if the documented output and attachment input do not match without an extra conversion step; one key does not prove representation compatibility. The code fails on missing configured fields instead of guessing a response property.

## What changes at scale

Move the frozen email payload, selected PDF output, operation keys, and next eligible attempt time into durable delivery state. The demonstration keeps the handoff in memory. A production worker should checkpoint the rendered result before email submission so a restart can resume that exact handoff rather than regenerate an attachment.

The platform's default deduplication window is 24 hours. Retain application delivery state beyond that window and stop expired signup jobs. A deduplication window is not a lifetime guarantee against sending the same event again.

Measure before tuning: time to first accepted request, render duration, email acceptance duration, attempts per signup, and time until a polled terminal status. Record measurements separately. Combining them into one average makes it hard to tell whether a template change increased rendering work or a send queue accumulated backoff. No throughput or latency result is asserted here.

For fallback, persist an explicit channel decision with the template revision. Polling introduces a reconciliation interval, so choose the SMS trigger from the signup's expiry policy rather than treating an empty event poll as a delivery failure. Keep sensitive verification material out of general logs. Do not queue a delayed verification message whose usefulness will expire before its next attempt.

## Compare the template boundary before choosing the stack

Three real alternatives matter at different points in this workflow. Puppeteer competes for the rendering job; Resend and Amazon SES compete for email submission. They are not three interchangeable end-to-end products.

| Option | Template ownership decision | Glue to account for |
| --- | --- | --- |
| Puppeteer | Keep document rendering in application-controlled browser code. | Own browser execution and pass its rendered output into a separate sender. It does not replace the email transport. |
| Resend with Puppeteer | Keep PDF rendering in the application; choose explicitly where email content is maintained. | Connect the renderer's output to the sender's documented attachment representation. |
| Amazon SES with Puppeteer | Keep PDF rendering in the application and evaluate email submission within an AWS account. | Own the rendering-to-message mapping and the sender credential configuration. |

A Puppeteer-plus-Resend stack needs one external sending-service signup and one sending credential set; the local rendering library adds neither a hosted-service signup nor an API key. Puppeteer plus SES needs an AWS account and sending credentials. Counting a local browser library as a second vendor account would exaggerate the consolidation benefit. Either stack still needs rendering execution, attachment mapping, and application delivery state.

Choose local rendering when browser control and ownership of that runtime justify the operational work. Choose a dedicated sender when its email workflow fits the team better than consolidating the PDF and send calls. The combined REST approach suits a small backend that values one authentication setup and a direct handoff. The trade-off is one vendor to trust, one bill, and one outage surface. A limitation of this design is its application-owned rendering contract: it isn't suitable for a team that needs browser-level layout control without delegating rendering. Choose Puppeteer with a dedicated sender in that case.

None of these choices outsources the signup state machine.

I would benchmark the handoff before choosing a transport. For a short evaluation, build one representative welcome sheet and one verification email, preserve their inputs, and exercise a rate-limit response in a controlled test. Check that the second attempt keeps the same payload and key. Then restart the worker between rendering and sending. The useful result is a reproducible account of template revision, artifact handoff, and delivery state—not an unsupported claim that one stack is faster.

## References

- [Puppeteer documentation](https://pptr.dev/)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [DMARC specification, RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)
- [WebOTP API documentation](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API), for evaluating a separate OTP flow.
