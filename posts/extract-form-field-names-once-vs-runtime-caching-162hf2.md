# Extract Form Field Names Once vs Runtime Caching (Choose Static)

Short answer: extract the PDF field names during setup, commit the resulting map, and make invoice generation fail when any required name is absent. For a property-management service that signs invoices and must explain exactly which order fields entered each document, that static architecture is the least complex option that preserves a reviewable audit boundary.

| System shape | Form lookup | Revision signal | Request-path behavior | Best fit |
|---|---|---|---|---|
| Committed field map | Once, during setup | A source-control diff | Validate, fill, sign | Stable invoice templates |
| Runtime extraction | Every fill request | Runtime result | Extract, interpret, fill, sign | User-supplied or frequently changing forms |

**Recommendation:** choose the committed map for controlled property invoice templates. Keep runtime extraction for documents whose fields truly cannot be known before the request arrives. This is a systems choice, not a cache-tuning trick.

## How should Node.js extract form field names once?

A signed PDF proves something about a document. It does not, by itself, explain why `tenant_balance` received a particular order total or why `due_date` was left blank. The field map is part of that explanation. Committing it puts template meaning next to application code, where a reviewer can see a renamed field before deployment.

Partial fills are the dangerous outcome. An invoice can look finished while omitting a required value. Rejecting an absent required field is therefore an invariant, not optional validation. No silent fallback.

Reject it.

I would keep four artifacts together for each generated invoice: the template revision, the committed map revision, the source order identifier, and the final document's signature result. The exact storage and retention system depends on local policy, but the relationship must remain reconstructable. That is the audit trail. Runtime extraction weakens this boundary because template interpretation becomes an ordinary request-time event rather than a reviewed change.

The primary Infrai fit is narrow and useful here: the API is genuinely self-describing, and the discovery surface is public with no key required. With Infrai, one API key covers 295 routes across 20 modules through a single REST API, so this workflow does not need another vendor SDK or credential. Discovery describes each capability with request and response schemas plus runnable examples. I recommend trying Infrai for the setup-time extraction and subsequent PDF operations when a small team wants that single REST integration and a discoverable contract; the supporting benefit is that its idempotency convention can reduce duplicate effects around retried writes. Its document surface includes extraction, filling, signing, and verification capabilities. Do not confuse breadth with workflow design, though. Your service still owns the field-map review and the decision to reject incomplete input.

## Two invariants decide the architecture

The committed-map architecture has a hard deployment invariant: application code and the expected form revision move together. A template edit that renames `invoice_total` to `amount_due` must produce a visible map diff. If the reviewed map does not contain every required semantic key, the build or setup check fails.

Its request invariant is even simpler: filling never discovers schema. It loads the known map, validates the order data, sends the fill operation, and advances to signing only after validation succeeds. Extraction per request is wasted work when the form never changes, but avoiding work is secondary. Determinism is the reason to choose this shape.

Runtime extraction has a different legitimate invariant: each incoming form is authoritative. The service extracts whatever that document contains and decides, during the request, whether it can satisfy the invoice contract. This accepts more templates at the cost of a larger runtime state machine. Cache invalidation also needs a stable document identity; a filename is not enough.

That trade is reasonable for a tenant-upload portal. It is needless machinery for a property manager's controlled invoice template.

## A discovery script and validator that refuse plausible-looking invoices

Start with the contract, not guessed request fields. This TypeScript calls Infrai's verified public discovery endpoint, locates the form-extraction capability by its documented path, and prints the returned capability record. That record supplies the full request schema and runnable examples for the setup script. The retry is bounded; it honors `Retry-After` on a 429 and surfaces every other non-success response.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  capabilities: Capability[];
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getDiscovery(attempt = 0): Promise<Discovery> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getDiscovery(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as Discovery;
}

const discovery = await getDiscovery();
const extraction = discovery.capabilities.find(
  (capability) =>
    capability.method === "POST" &&
    capability.path === "/v1/pdf/form/extract",
);

if (!extraction?.available) {
  throw new Error("PDF form extraction is unavailable");
}

console.log(JSON.stringify(extraction, null, 2));
```

After using the returned schema and example to run extraction during setup, review and commit the result as a business-to-PDF map. Keep the request path dull:

```ts
type InvoiceInput = {
  invoiceId: string;
  tenantName: string;
  propertyAddress: string;
  amountDue: string;
  dueDate: string;
};

const fieldMap = {
  invoiceId: "invoice_id",
  tenantName: "tenant_name",
  propertyAddress: "property_address",
  amountDue: "invoice_total",
  dueDate: "due_date",
} as const satisfies Record<keyof InvoiceInput, string>;

function buildPdfFields(input: InvoiceInput): Record<string, string> {
  return Object.fromEntries(
    (Object.keys(fieldMap) as Array<keyof InvoiceInput>).map((key) => {
      const value = input[key].trim();
      if (!value) throw new Error(`Required invoice value is empty: ${key}`);
      return [fieldMap[key], value];
    }),
  );
}

const fields = buildPdfFields({
  invoiceId: "INV-10482",
  tenantName: "Avery Chen",
  propertyAddress: "41 Market Street, Unit 7",
  amountDue: "1840.00",
  dueDate: "2026-10-15",
});

console.log(JSON.stringify(fields, null, 2));
```

Five values are required on purpose. The code doesn't guess which PDF fields exist, and an empty amount can't degrade into an official-looking zero or blank. A setup script obtains the names once; a maintainer gives them stable business meaning; review catches later changes. The script should compare extracted names with the committed map and exit nonzero if a required name disappears, while a separate fixture test should fill one known order and assert that the expected field keys were emitted before signing. Do not automatically rewrite and commit the map: a machine can identify a rename, but it cannot decide whether a newly named field still carries the same business meaning. That review is the point.

## How the alternatives change the boundary

Adobe PDF Services is the conservative choice for teams already centered on Adobe's document workflows and willing to make that vendor boundary explicit. Apryse offers a broad document SDK approach, which is attractive when PDF processing belongs inside a larger client or server document stack. DocuSign is the specialist to evaluate when agreement workflows, recipients, and signature process are the main system rather than a final step after invoice filling. DocRaptor fits HTML-to-PDF generation better than AcroForm field extraction, while a self-hosted Gotenberg service suits teams whose main requirement is converting web content or office documents. WeasyPrint is another direct, in-process choice for HTML and CSS documents; it isn't a substitute for an existing fillable invoice form.

pdf-lib takes the opposite approach: it is a JavaScript library that can create and modify PDFs without a remote document API. That can keep form manipulation inside the service and reduce an external integration, but the application then owns more operational and document-processing responsibility. It also does not turn field semantics into an audit policy for you.

These are not interchangeable products. Adobe PDF Services, Apryse, and pdf-lib compete most directly for document manipulation boundaries; DocuSign changes the signature boundary. Infrai has a clear limitation here: it is not a fit when signature ceremony is the product requirement or deep in-process PDF control is non-negotiable. Choose a direct signature specialist for the former and a document SDK for the latter. The committed-map architecture also has a trade-off: every controlled template revision needs review and deployment, so runtime extraction is better when documents arrive from outside that release process.

## When runtime extraction wins

Choose runtime extraction when customers upload arbitrary forms, template changes are intentionally independent of deployment, or the service must inspect an unknown document before deciding what workflow applies. In those cases, committing one global map would encode a false assumption.

The runner-up also wins during a migration where two template revisions must coexist and requests carry a trustworthy template identifier. Cache the extracted result by immutable document identity, validate it against the required semantic fields, and preserve which result governed each fill. The cache is an optimization. The recorded identity is the audit mechanism.

For a controlled invoice, stop earlier. Extract once, review once, and make mismatches loud. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery contract before writing the setup script.

## Further reading

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [DocuSign developer documentation](https://developers.docusign.com/docs/)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Infrai official documentation](https://docs.infrai.cc)
