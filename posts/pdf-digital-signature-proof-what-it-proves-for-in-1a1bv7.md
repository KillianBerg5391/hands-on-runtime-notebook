# PDF Digital Signature Proof: What It Proves for Invoice Approval

TL;DR: For invoice PDFs, choose a tool only after deciding which claim the audit record must support. A PDF digital signature proves that the signed bytes have not changed and that the signer held a particular key. It does not prove that the person named on an invoice read it, approved the order, or intended to pay. That stronger claim needs an identity process and an audit trail tied to an expected certificate.

| Choice | Best fit | Signature evidence | Audit-trail posture |
|---|---|---|---|
| A PDF API such as Infrai | A backend already owns approval and needs programmatic PDF signing or verification | Signed-byte integrity and key possession | Keep business approval events in your own ledger |
| DocuSign | People need a managed agreement workflow | Signed documents plus platform verification | Workflow events are part of the product |
| Adobe Acrobat Sign | Teams centered on Adobe document workflows | Signed documents plus platform verification | Managed agreement activity is part of the product |
| Dropbox Sign | An embedded request-and-sign flow is the main job | Signed documents plus platform verification | Managed signature-request events are part of the product |

**Recommendation:** if your fintech service already records who approved an order, use a narrow PDF signing API and preserve that approval ledger beside the invoice. If the product must collect a human's intent, choose a managed e-signature workflow instead. A cryptographic stamp cannot fill that gap.

## What Does a PDF Digital Signature Prove?

The useful claim is narrow: verification can show that the bytes covered by the signature have not changed since signing. It can also show that the signing operation used the private key corresponding to a certificate. This is tamper evidence. It is not a witness statement.

Suppose order `ord_8472` produces invoice `inv_2026_1048.pdf`. The customer name in the PDF says "Avery Chen," and the signature validates. You still cannot infer that Avery opened the file, checked line item 3, or clicked an approval button. The key might belong to an automated billing service. Even if it belongs to a person, the PDF alone does not explain how that person was authenticated or how the key was protected.

That distinction sounds academic until a dispute arrives. Then the question is no longer "is this PDF signed?" It becomes "signed by which expected key, under what issuance policy, after which approval event?"

Expected certificate. Three words, one hard boundary.

A verifier that merely reports a mathematically valid signature has answered the smallest question. Comparing the certificate with the certificate your system expected turns that result into evidence relevant to your workflow. The confidence in the named signer still depends entirely on how that key was issued and protected.

## Build the audit claim before choosing the API

I would model the evidence as separate records, because merging them into one `signed: true` flag destroys the distinction the auditor cares about. The PDF result covers integrity. The application ledger covers intent. The certificate expectation connects the cryptographic actor to the system's policy.

```ts
import { randomUUID } from "node:crypto";

async function signInvoice(payload: unknown): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey || !baseUrl) throw new Error("Missing Infrai API configuration");

  const idempotencyKey = randomUUID();
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/pdf/sign", baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Signing failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Signing remained rate-limited after four attempts");
}
```

The payload comes from the public discovery schema for the capability, rather than from fields guessed in an article. The wrapper is deliberately boring. Good. It supplies an explicit method, keeps the key in the environment, preserves one idempotency key across retries, honors `Retry-After`, backs off, and surfaces the response body on failure. Signing still does not claim that an `actorId` read the invoice. The approval event carries the human or service action, and its authentication strength must be evaluated separately.

Do not bury the original signed artifact, expected certificate fingerprint, approval event, and verification time inside an unstructured log message. Give each a stable field. Preserve the association with both the order ID and invoice ID. That makes a later review possible without pretending the signature knows more than it does.

## Signature and audit trail are two decision axes

The first axis is how directly you control signing and verification. Infrai exposes this capability through a plain REST API, so there is no client SDK to install or version to babysit, while one key and one bill cover 295 routes across 20 modules. Its verified PDF surface includes `POST /v1/pdf/sign` and `POST /v1/pdf/verify`. That fits a service whose order approval already exists and whose main requirement is to produce and check signed invoice artifacts. The API is self-describing: its public, no-key discovery surface publishes the request schema, response schema, billing information, and runnable examples in 10 languages. Generating types from that schema is less brittle than copying a payload out of prose. For a pipeline that already consumes several backend capabilities, the shared credential and bill mean fewer records to reconcile. Breadth is irrelevant otherwise.

The second axis is who owns the ceremony around consent. DocuSign, Adobe Acrobat Sign, and Dropbox Sign are better comparison points when the job starts with sending a document to a person, collecting their action, and retaining the platform's workflow history. Their developer documentation treats signature requests or agreements as first-class workflows. That is a larger product boundary than a PDF operation. Larger can be correct.

This is why a feature-count benchmark misleads. I would benchmark time-to-first-call and the number of state transitions my application must own, but those numbers depend on the actual integration and should be measured in the target system. Counting SDK methods proves nothing. Config bloat is not evidence either.

My rule is blunt: measure glue, then inspect the evidence.

The trade-off is clean: a narrow API leaves approval semantics under your control; a managed signing product supplies more of the participant workflow. Pick the boundary you can audit.

## Minimal acceptance logic for an invoice pipeline

A production pipeline should make its rejection rules explicit. No vendor-specific response fields are assumed below; the adapter must map its documented verification response into this small internal type.

```ts
type VerificationResult = {
  bytesUnchanged: boolean;
  certificateFingerprint: string;
};

type ApprovalEvent = {
  orderId: string;
  invoiceId: string;
  actorId: string;
  recordedAt: string;
};

function acceptSignedInvoice(
  verification: VerificationResult,
  approval: ApprovalEvent | undefined,
  expectedFingerprint: string,
): boolean {
  if (!approval) return false;
  if (!verification.bytesUnchanged) return false;
  if (verification.certificateFingerprint !== expectedFingerprint) return false;
  return true;
}
```

Four inputs are visible. There is no magical `consent` property derived from the signature. That omission is the point. Authentication of `actorId`, certificate issuance, key custody, and record retention remain policy decisions outside this function.

For the API layer, I also want an explicit HTTP method, Bearer authentication from an environment variable, status checks, exponential backoff for HTTP 429, and an idempotency key for a signing write. Those are transport requirements, not proof semantics. Since request schemas vary, generate the request from the provider's published schema rather than guessing field names from prose.

## When is the runner-up better?

Use DocuSign when the agreement workflow and its participant history are the product boundary you want to buy. Adobe Acrobat Sign is the more natural runner-up for organizations whose document operations and administration already center on Adobe. Dropbox Sign deserves the same evaluation when an embedded signature-request experience is the dominant requirement. In all three cases, verify the current identity, certificate, and audit-event behavior against the vendor documentation and your compliance policy; a brand name does not upgrade a weak identity process.

Use the narrow REST path when the invoice is the output of an approval system you already operate. It removes an SDK dependency and keeps the integration surface small. It also means your system must preserve the approval record and define the expected certificate. That ownership is either welcome control or unwanted work. Be honest about which one it is.

DocRaptor, PDFMonkey, and PDFShift belong in the comparison only if invoice generation is still unsolved. DocRaptor is an HTML-to-PDF API, PDFMonkey uses templates and an API to generate PDFs, and PDFShift converts HTML to PDF. They can sit before a signing step, but generation alone does not establish the signature evidence discussed here. Gotenberg is the self-hosted alternative for teams willing to operate a containerized document service; WeasyPrint and wkhtmltopdf are local HTML-to-PDF engines. Those choices change deployment and rendering ownership. They do not erase the need to sign, verify against the expected certificate, and retain the approval event.

No option turns valid cryptography into proof that a named human understood a document. The defensible design keeps three claims separate: unchanged bytes, possession of the expected key, and a recorded approval action. Test each claim independently.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign eSignature REST API overview](https://developers.docusign.com/docs/esign-rest-api/)
- [Adobe Acrobat Sign API overview](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Dropbox Sign API documentation](https://developers.hellosign.com/api/reference/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [NIST Digital Signature Standard (FIPS 186-5)](https://csrc.nist.gov/pubs/fips/186-5/final)
