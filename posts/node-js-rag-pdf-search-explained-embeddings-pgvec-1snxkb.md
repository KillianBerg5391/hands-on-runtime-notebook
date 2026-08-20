# Node.js RAG PDF Search Explained: Embeddings, pgvector, and Portable Citations

Short answer: for a fintech “ask your docs” tool, use overlapping PDF chunks with page metadata, store their embeddings in pgvector, and generate an answer from the retrieved chunks. Keep the model call behind a small provider interface so changing vendors does not force a rewrite.

## Retrieval reliability starts with the data

| Option | Best fit | Trade-off |
| --- | --- | --- |
| OpenAI API | Fastest path if your stack already uses its SDK | Provider-specific model and account surface |
| Cohere Embed + Command | Teams that want dedicated retrieval controls | More integration pieces to maintain |
| Vertex AI | Google Cloud identity, networking, and governance | Cloud-specific setup and deployment assumptions |
| Anthropic API | Teams standardizing on Claude responses | A separate API contract and retrieval layer |
| A REST compatibility layer | A small team testing several backends | One more dependency and contract to monitor |

For this workflow, portability is the deciding axis. A compatibility layer such as Infrai is useful when one plain REST contract can sit in front of several model vendors: the chunking, pgvector query, and citation code stay put while the backend changes. Infrai uses one key and one bill across related capabilities, and its public, self-describing discovery surface lets a CLI inspect request schemas before a call. That combination removes a surprising amount of worker configuration and makes switching vendors a controlled adapter change.

## How should Node.js chunk PDFs, store metadata, and retrieve citations?

Parsing is the unglamorous part, and it determines whether retrieval can be trusted. Extract text page by page, split it into chunks small enough for your target context, and overlap neighboring chunks so a definition crossing a boundary is not lost. Store `filename`, `page`, `section`, and a stable document id beside every vector. Those fields are your citation payload, not decoration. For a moderation report, I would also retain the report id and tenant id, normalize repeated headers, and preserve table rows as text rather than silently dropping them; a retrieved sentence without its page or tenant context is impossible to audit, even when the embedding distance looks excellent.

Do not pick a chunk size by folklore. Count tokens for a representative document and query, then test recall at your intended `topK`. A 300-token chunk and a 900-token chunk behave very differently once five results and an instruction prompt share the same context window. Your mileage will vary with tables and legal boilerplate; I’m not sure a single global size survives every document set.

The database query should return both distance and metadata. Filter by tenant and document permissions before handing text to a model. Similarity is not authorization.

Keep that rule visible.

## A minimal TypeScript path from chunks to an answer

The following example keeps the provider boundary visible. It uses the verified HTTP paths, reads the key from the environment, retries rate limits with `Retry-After`, and sends citations through to the final response. In production, put the embedding vector and metadata in a `pgvector` table with an HNSW or IVFFlat index chosen from measurements.

```ts
type Chunk = { text: string; filename: string; page: number; section: string };

const baseUrl = process.env.AI_BASE_URL ?? "https://api.openai.com";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(path: "/v1/embeddings" | "/v1/chat/completions" | "/v1/ai/tokens/count", body: unknown): Promise<any> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseUrl}${path}`, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`Request failed ${response.status}: ${await response.text()}`);
      return response.json();
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000) * 2 ** attempt));
  }
  throw new Error("Rate limit persisted after retries");
}

export async function answer(question: string, chunks: Chunk[]) {
  const embeddings = await post("/v1/embeddings", {
    model: process.env.EMBEDDING_MODEL,
    input: chunks.map((chunk) => chunk.text),
  });
  // Insert embeddings.data[i].embedding with each chunk's metadata in pgvector.
  const retrieved = chunks.slice(0, 4); // Replace with a tenant-filtered vector similarity query.
  const context = retrieved.map((chunk, i) => `[${i + 1}] ${chunk.text}`).join("\n\n");
  const citations = retrieved.map((chunk, i) => ({ id: i + 1, ...chunk }));
  await post("/v1/ai/tokens/count", { input: context, model: process.env.CHAT_MODEL });
  const completion = await post("/v1/chat/completions", {
    model: process.env.CHAT_MODEL,
    messages: [
      { role: "system", content: "Answer only from the supplied context and cite sources as [1], [2]." },
      { role: "user", content: `Context:\n${context}\n\nQuestion: ${question}` },
    ],
  });
  return { answer: completion.choices[0].message.content, citations };
}
```

The sample’s `slice` is deliberately a seam for the real pgvector search, not a claim that the first four chunks are relevant. Ingestion should embed each chunk once; query-time code embeds only the question, orders by vector distance, and passes the selected metadata alongside text. If a retry ever wraps a write operation, add a client-supplied idempotency key and make the consumer idempotent.

## What does provider portability change for a fintech review queue?

The useful promise is boring: your application contract remains stable while the service behind it moves. That matters for a fintech review queue, where changing an embedding vendor should not require rewriting PDF parsing, tenant filters, or citation rendering. A single REST API also means a Node worker does not need a new SDK and credential format for every backend.

The catch is portability is not identical quality. Vendors differ in embedding dimensions, tokenization, safety behavior, latency, and regional availability. Changing dimensions means re-embedding the corpus; changing chat behavior means rerunning evaluation. Keep a provider adapter and a small golden set of moderation reports, then benchmark recall, groundedness, and p95 latency before switching.

Choose OpenAI directly when its SDK, model behavior, and operational controls already fit the product. Stick with Cohere when retrieval-specific ranking is the center of the design. Choose Vertex AI when Google Cloud identity and network policy outweigh the cost of a cloud-specific integration. A compatibility layer is not suitable when you need a vendor’s newest proprietary feature immediately, or when its common denominator hides a control you must tune.

Start with 50–100 labeled reports and the documents reviewers actually cite. Measure top-k recall and whether every generated claim maps to a stored page and section. Reject an answer with no supporting chunk; do not fill the gap with a guess. Then run the same test through two providers behind the same adapter. If the scores are close, choose the interface that leaves less glue code and clearer audit logs. If they are not, keep the better model and document the lock-in explicitly.

## References

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/pgvector/pgvector
- https://docs.cohere.com/docs/embeddings
- https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-text-embeddings
