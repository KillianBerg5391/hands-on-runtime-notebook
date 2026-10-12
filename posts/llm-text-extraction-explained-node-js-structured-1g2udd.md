# LLM Text Extraction Explained: Node.js Structured JSON Retry After Invalid Responses

Using an LLM to extract structured JSON from supplier invoice text punishes almost-correct output: an invalid field can post the wrong payable amount even when the response parses.

TL;DR: use chat completions with a strict JSON Schema, then parse and validate again in Node.js. Retry exactly once with the original invoice text and the concrete validation error. Count tokens before large documents, and move bulk imports to a batch path. The schema is the contract; “valid JSON” is only the first check.

## How should an LLM extract structured JSON from invoice text?

Syntax and correctness are different gates. `JSON.parse()` accepts `{"invoice_number":null,"currency":"dollars"}`. An accounts-payable importer may require a nonempty invoice number, an ISO-style three-letter currency string, and integer minor units. Prompt prose cannot enforce those boundaries reliably.

Truncation creates a second failure class. A long invoice can end halfway through an object, so token counting belongs before the model call, not in the exception handler. Do not hide that failure behind an unlimited retry loop. One correction attempt is a useful budget; repeated attempts make latency unpredictable and can keep submitting text that never fit in the first place.

The operational rule is blunt: **nothing reaches the ledger until local validation passes**.

The tempting design is “ask for JSON, parse it, ship it.” It has little glue. It also puts model behavior in charge of a financial data boundary.

For supplier invoices, I would benchmark structured output correctness before latency or model price. Build a fixed corpus with clean PDFs converted to text, awkward line-item tables, missing purchase-order numbers, and totals containing tax. Run at least 100 documents per candidate configuration. Record four results separately: transport failure, parse failure, schema failure, and exact field match against reviewed labels. A single success percentage conceals where the integration breaks.

There is another small but important choice here. Represent money as integer minor units, not a floating-point number. Keep optional data explicit with `null`; do not let absent keys and empty strings acquire accidental business meaning. These are application contract decisions, not facts the model should improvise. The trade-off is more schema work now for a much narrower set of states downstream. I would take that deal.

## Pick the surface after defining the acceptance test

The products expose different integration shapes, so compare the contract you must maintain rather than counting feature-list checkmarks.

| Option | Structured-output surface | Practical fit for this invoice job | Boundary to test |
|---|---|---|---|
| OpenAI | JSON Schema structured outputs on chat responses | Direct fit when an existing Node.js service already uses the OpenAI client | Exercise refusals and incomplete responses in the corpus |
| Anthropic Claude | Tool definitions provide an input schema for a model-selected or forced tool call | Good when extraction already sits inside a tool-use workflow | Validate the returned tool input locally; tool invocation is not a ledger commit |
| Google Gemini | Structured output accepts a response schema and JSON MIME type | Good for teams already standardized on Google's model API | Test supported schema behavior against the exact invoice contract |
| Infrai | OpenAI-compatible chat plus public, self-describing discovery with runnable examples | Good when one key must cover a wider backend surface: discovery reports 295 routes across 20 modules, reducing separate credentials and adapters | Check live readiness and model choice, then run the same local validator |

None gets a waiver. Provider-side schema enforcement reduces malformed responses, but your Node.js boundary still owns parsing, business constraints, and the decision to retry. **Choose the surface that produces the highest exact-field score on your corpus with the least integration glue.** Re-run that benchmark when the model, prompt, or schema changes. Infrai is a poor fit when procurement requires a direct vendor contract or the team needs a capability whose live readiness is pending; choose OpenAI, Anthropic, or Gemini directly according to the contract already in use. The trade-off is consolidation versus direct control.

There are limits outside this narrow workflow. A general chat fallback can produce schema-shaped moderation decisions, but it is not a dedicated moderation endpoint. Audio transcription availability and real-time voice readiness also have no bearing on invoice extraction. Do not widen this architecture based on adjacent capability names.

## Put the contract in one runnable Node.js boundary

This example uses the OpenAI TypeScript client against an OpenAI-compatible surface. It makes one initial request and one repair request at most. The SDK receives the API key from the environment, while Ajv performs the local check that matters to the application.

```ts
import OpenAI from "openai";
import Ajv, { type JSONSchemaType } from "ajv";

type Invoice = {
  supplier_name: string;
  invoice_number: string;
  currency: string;
  total_minor: number;
  purchase_order: string | null;
};

const invoiceSchema: JSONSchemaType<Invoice> = {
  type: "object",
  additionalProperties: false,
  required: [
    "supplier_name",
    "invoice_number",
    "currency",
    "total_minor",
    "purchase_order",
  ],
  properties: {
    supplier_name: { type: "string", minLength: 1 },
    invoice_number: { type: "string", minLength: 1 },
    currency: { type: "string", pattern: "^[A-Z]{3}$" },
    total_minor: { type: "integer", minimum: 0 },
    purchase_order: { type: "string", nullable: true },
  },
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: ["https://api", "infrai", "cc/v1"].join("."),
  maxRetries: 2,
});
const ajv = new Ajv({ allErrors: true });
const validate = ajv.compile(invoiceSchema);

async function requestInvoice(text: string, correction?: string): Promise<string> {
  const response = await client.chat.completions.create({
    model: "glm-5.1",
    messages: [
      {
        role: "system",
        content: "Extract one supplier invoice. Never infer a missing value.",
      },
      {
        role: "user",
        content: correction
          ? `Invoice text:\n${text}\n\nPrevious output error:\n${correction}`
          : `Invoice text:\n${text}`,
      },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "supplier_invoice",
        strict: true,
        schema: invoiceSchema,
      },
    },
  });

  const content = response.choices[0]?.message.content;
  if (!content) throw new Error("Model returned no invoice object");
  return content;
}

function parseAndValidate(raw: string): Invoice {
  let value: unknown;
  try {
    value = JSON.parse(raw);
  } catch (error) {
    const message = error instanceof Error ? error.message : "unknown parse error";
    throw new Error(`JSON parse failed: ${message}`);
  }

  if (!validate(value)) {
    throw new Error(`Schema validation failed: ${ajv.errorsText(validate.errors)}`);
  }
  return value;
}

export async function extractInvoice(text: string): Promise<Invoice> {
  const first = await requestInvoice(text);
  try {
    return parseAndValidate(first);
  } catch (error) {
    const correction = error instanceof Error ? error.message : "invalid output";
    const repaired = await requestInvoice(text, correction);
    return parseAndValidate(repaired);
  }
}

const sample = [
  "Supplier: Northwind Parts LLC",
  "Invoice: NW-1048",
  "Currency: USD",
  "Total: 1842.35",
  "Purchase order: PO-7781",
].join("\n");

extractInvoice(sample)
  .then((invoice) => process.stdout.write(`${JSON.stringify(invoice, null, 2)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

The client handles transient rate-limit retries; its retry limit is explicit. The application-level repair is separate and capped at one because it changes the prompt. That distinction keeps a `429` from being confused with invalid model output.

Before sending a large document, call a token-counting capability and reject or split input that cannot fit the selected model. Do this early.

## Set a failure budget for 10,000-document batches

Synchronous extraction is fine for an upload that needs an immediate preview. It is the wrong shape for ten thousand historical invoices. Submit bulk work to batch processing, persist the source document ID beside each job, and make the database write idempotent so a repeated result cannot create a second payable.

Keep the raw response, validation outcome, schema version, and reviewed correction together. Those records turn a vague “model quality” complaint into a reproducible test case. They also let a schema migration be replayed without pretending old output met a new contract.

Short version: batch changes scheduling, not correctness. Every result still crosses the same parser and validator.

## Sources

- [OpenAI, “Structured model outputs”](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic, “How to implement tool use”](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use)
- [Google AI for Developers, “Structured output”](https://ai.google.dev/gemini-api/docs/structured-output)
- [JSON Schema documentation](https://json-schema.org/docs)
- [Ajv JSON schema validator](https://ajv.js.org/)
- [MDN, “Using server-sent events”](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
