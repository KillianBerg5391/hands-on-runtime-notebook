# Validating LLM Summary JSON at the Node.js API Boundary

Short answer: treat an LLM summary as untrusted input, validate its JSON at one Node.js API boundary, and keep action items tied to source evidence before any other code can use them.

The deciding constraint is downstream behavior. A title can be slightly awkward and still be useful. A fabricated task that reaches a ticket queue is different. So the build starts with the consumer contract and the failure response, not with prompt wording or a provider SDK.

Keep it narrow.

The smallest useful output has a `title`, `bullets`, and `action_items`. Each action also carries a `source_quote`, because syntax alone cannot show that the task came from the input. This adds a little output weight, but it gives a reviewer something concrete to inspect. I care about time-to-first-call, yet I care more about how much defensive glue every caller would otherwise repeat.

## How should a Node.js API validate LLM summary JSON?

Validation has three separate jobs. First, parse the transport payload. Second, reject values that violate the application shape. Third, check claims against the source. Combining those jobs into one vague “JSON mode worked” flag hides the most important failure: valid JSON can still contain unsupported content.

Why three?

Because each failure has a different owner and a different fix.

| Output path | Best fit | Main limit |
|---|---|---|
| Free-form prose | A person reads every result | Callers cannot depend on stable fields |
| Validated summary JSON | Software renders or queues the result | Shape checks cannot prove source support |
| Deterministic parsing | Inputs follow a fixed machine template | Paraphrase and ambiguity break fixed rules |

The contract below makes its choices explicit. The title and bullets cannot be blank. Unknown keys fail. An empty `action_items` array is allowed because a source may contain no task at all; inventing one to satisfy a minimum would be worse than returning none. The exact length limits are product decisions, so this example does not pretend there is one universal cap.

```ts
type ActionItem = {
  task: string;
  source_quote: string;
};

type Summary = {
  title: string;
  bullets: string[];
  action_items: ActionItem[];
};

type ModelAdapter = (request: {
  source: string;
  outputSchema: object;
}) => Promise<unknown>;

const outputSchema = {
  type: "object",
  additionalProperties: false,
  required: ["title", "bullets", "action_items"],
  properties: {
    title: { type: "string", minLength: 1 },
    bullets: {
      type: "array",
      minItems: 1,
      items: { type: "string", minLength: 1 },
    },
    action_items: {
      type: "array",
      items: {
        type: "object",
        additionalProperties: false,
        required: ["task", "source_quote"],
        properties: {
          task: { type: "string", minLength: 1 },
          source_quote: { type: "string", minLength: 1 },
        },
      },
    },
  },
} as const;
```

A schema object is useful to an adapter that supports constrained output, but it is not runtime validation inside the application. The API still receives `unknown`. I don't let a type assertion turn that unknown value into trusted data, and I don't strip Markdown fences or search a prose response for the first pair of braces. Those repairs make malformed output look successful. There is one deliberate duplication in this small version: the static TypeScript type and the runtime checks describe the same shape. For a single module, the cost is visible and manageable. At scale, I would use the codebase's established validator and derive the type from one schema definition. Adding a new validation stack just to remove a few lines would be config bloat in a nicer shirt.

## The smallest working implementation

The core function accepts text and an injected adapter. It knows nothing about a vendor route, model ID, authentication header, or SDK. That keeps tests fast and makes the boundary obvious: adapters produce candidates; this module produces summaries.

```ts
function isNonBlank(value: unknown): value is string {
  return typeof value === "string" && value.trim().length > 0;
}

function hasExactKeys(
  value: Record<string, unknown>,
  allowed: readonly string[],
): boolean {
  const keys = Object.keys(value);
  return keys.length === allowed.length && keys.every((key) => allowed.includes(key));
}

function parseActionItem(value: unknown, source: string): ActionItem {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("SUMMARY_SHAPE_INVALID: action item must be an object");
  }

  const item = value as Record<string, unknown>;
  if (!hasExactKeys(item, ["task", "source_quote"])) {
    throw new Error("SUMMARY_SHAPE_INVALID: action item keys do not match");
  }
  if (!isNonBlank(item.task) || !isNonBlank(item.source_quote)) {
    throw new Error("SUMMARY_SHAPE_INVALID: action item fields cannot be blank");
  }
  if (!source.includes(item.source_quote)) {
    throw new Error("SUMMARY_EVIDENCE_MISSING: source quote was not found");
  }

  return { task: item.task.trim(), source_quote: item.source_quote };
}

function parseSummary(value: unknown, source: string): Summary {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("SUMMARY_SHAPE_INVALID: summary must be an object");
  }

  const record = value as Record<string, unknown>;
  if (!hasExactKeys(record, ["title", "bullets", "action_items"])) {
    throw new Error("SUMMARY_SHAPE_INVALID: summary keys do not match");
  }
  if (!isNonBlank(record.title)) {
    throw new Error("SUMMARY_SHAPE_INVALID: title cannot be blank");
  }
  if (!Array.isArray(record.bullets) || !record.bullets.every(isNonBlank)) {
    throw new Error("SUMMARY_SHAPE_INVALID: bullets must be nonblank strings");
  }
  if (record.bullets.length === 0 || !Array.isArray(record.action_items)) {
    throw new Error("SUMMARY_SHAPE_INVALID: lists do not match the contract");
  }

  return {
    title: record.title.trim(),
    bullets: record.bullets.map((bullet) => bullet.trim()),
    action_items: record.action_items.map((item) => parseActionItem(item, source)),
  };
}

export async function summarize(
  rawText: string,
  invokeModel: ModelAdapter,
): Promise<Summary> {
  const source = rawText.trim();
  if (!source) throw new Error("SUMMARY_INPUT_EMPTY");

  const candidate = await invokeModel({ source, outputSchema });
  return parseSummary(candidate, source);
}
```

Exact substring evidence is intentionally conservative. It works when the model copies a short span from the source. It can reject a legitimate quote after Unicode normalization, whitespace folding, or transcription cleanup. I'm not sure one normalization policy fits every input channel; the right answer depends on whether the stored source is raw text, cleaned text, or a timed transcript. Pick one canonical representation, document it, and test it. Don't silently fall back to accepting an unmatched quote.

For audio, speech recognition belongs before this function. An open-source recognizer such as Whisper can produce the transcript, but transcription and summarization should remain separate stages. The summary tests then use fixed text fixtures, while transcription tests cover their own language and timing concerns. One pipeline. Two contracts.

## Failures worth designing before deployment

I would start with a fake `ModelAdapter`, not a live call. Feed the parser an array instead of an object, an extra top-level key, an empty bullet, a string in `action_items`, and a quote absent from the source. Each case should produce a stable application error. Then add semantic fixtures: conflicting statements, repeated speaker labels, commands embedded in the source, and discussion that names a possible task without assigning it. A JSON schema can reject the wrong type. It cannot decide whether “we could revisit this” is an actual action item. This is also where HTTP behavior matters. RFC 9110 defines idempotent methods in terms of the intended effect of repeated identical requests. A summary request commonly uses a non-idempotent method, so a generic retry can cause another model invocation even if the caller never saw the first response. Put timeout and retry policy in the transport adapter, not in `summarize`. Retry only failures the transport contract identifies as transient, use a stable idempotency mechanism when the service supports one, and never retry a schema or evidence rejection. The response arrived. It failed your contract. I've kept those failures separate because a retry loop around validation can convert one bad response into repeated cost without improving the answer.

Return a client-safe status and an internal code without leaking the source text. For example, an API can map empty input to a client error and malformed model output to an unavailable dependency response while logs retain only the application code, schema version, input size, and timing. The exact status mapping is part of the API contract. More important is consistency: callers must be able to distinguish “change your request” from “try later” without parsing an English message.

Observability needs restraint. Measure total duration, adapter duration, validation duration, input size, output size, retry count, schema version, and the final validation code. Compare cold and warm execution paths because initialization can sit outside the model call while still defining user-visible latency. Do not log source documents, transcripts, generated summaries, or evidence quotes by default. They may contain names, private decisions, credentials copied into a meeting, or customer material. A debug flag that duplicates all of it into log storage is expensive glue with a privacy bill attached.

Benchmarks should answer a decision, not decorate a dashboard. Run the same fixture set against candidate adapters and record the distribution from API receipt through validated output, plus the fraction rejected for shape and evidence. Use your own input lengths and concurrency. Any universal latency claim would be suspect because runtime location, model choice, source length, cold starts, and queueing all move the result. Your mileage may vary — measure the path you operate.

## What changes at scale, and when should you skip an LLM summary API?

At higher volume, add bounded concurrency and admission control before adding prompt complexity. Separate interactive work from batch jobs so a large backfill cannot consume every slot. Version the contract, keep evaluation fixtures with that version, and compare old and new outputs before a rollout. If a downstream action can send a message, modify a ticket, or schedule work, require human confirmation or a domain-specific policy check. Evidence makes review possible; it does not make an inference correct.

The catch is strict summary JSON is not suitable for every reading task. Use reviewed prose when nuance and surrounding context matter more than field stability. Return cited spans when a reviewer must audit each claim. Keep the transcript and timing metadata available when recognition uncertainty matters. And skip the LLM when the input is already a stable machine-generated template that deterministic parsing can handle. A parser is easier to test and removes semantic variation altogether.

There is another trade-off: evidence quotes increase payload size and may repeat sensitive text. If the summary crosses a trust boundary, return source offsets or opaque evidence IDs instead, then resolve them only for authorized reviewers. That requires more storage coordination, but it avoids copying raw passages through every consumer. Stick with inline quotes for a small, single-trust-zone service; switch to references when access control or document size makes duplication risky.

The production criterion is plain. Accept an LLM summary only after shape validation, evidence checks, and explicit downstream policy. Keep the adapter replaceable, keep retries out of business logic, and measure the full request path. The model call is the variable part. The contract around it should be boring.

## References

- RFC 9110: HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- openai/whisper, open-source speech recognition: https://github.com/openai/whisper
