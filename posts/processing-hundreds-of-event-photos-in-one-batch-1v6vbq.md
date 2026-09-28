# Processing Hundreds of Event Photos in One Batch With Progress (and a Storage Budget)

**Short answer:** use an asynchronous batch job with bounded workers, durable originals, immutable derivatives, and a polling or event endpoint for progress.

Processing hundreds of event photos in one HTTP request is a storage problem disguised as a JavaScript problem. My default design is an asynchronous batch job: accept a manifest, put originals in durable object storage, process a bounded number of images at a time, and stream progress from a separate endpoint. That keeps Express responsive and makes the cache policy explicit.

The choice is less exciting than a clever worker pool. It is also the choice that survives a photographer uploading 640 ten-megapixel files while three customers request previews.

Keep the web request short.

## The decision matrix

| Shape | Progress for the browser | Storage behavior | Best fit | Main catch |
| --- | --- | --- | --- | --- |
| One synchronous request | A final 200 or timeout | Temporary files pile up in the web process | Fewer than 20 tiny images | A timeout turns partial work into guesswork |
| Async job plus polling | Predictable percentage and item states | Originals and derivatives have separate retention | Most event galleries | The client must remember a job ID |
| Queue plus event stream | Near-live updates | Workers can scale independently | High-volume, repeatable workloads | More moving parts and a reconnect story |

For an Express application serving event galleries, choose the middle row first. Add a queue and a stream when measurements show that one worker process cannot drain the backlog. Starting with a queue is not automatically more serious engineering; it is often config bloat before you know the workload. The useful test is a deliberately ugly one: load a manifest with 640 entries, make every tenth source slow, kill a worker halfway through, restart it, and verify that the status endpoint still reports the same completed IDs. If the bar jumps backward, the state lives in the wrong place. If a retry emits a second derivative, the key is missing an input or profile version. If memory rises with every retry, the decoder or stream is not being released. Those are storage and lifecycle bugs, even though the symptom appears in a progress widget.

The two numbers I watch are bytes retained per attendee and the time between progress updates. A cache hit is useful only if its key identifies the source bytes and every transformation option. A progress bar is useful only if it moves when an item changes state, not when a loop happens to print a log line.

## How should a Node.js Express batch expose progress for hundreds of event photos?

Treat the batch as a small state machine. `queued` means the manifest passed validation. `processing` means a worker owns the item. `ready` means a derivative is durable and addressable. `failed` is an item-level result with a safe reason, not a reason to discard 639 successful photos. The batch itself can be `complete` when every item is terminal.

The HTTP layer only creates jobs and reads state. It should not decode pixels. A `POST /batches` handler can store a manifest containing object keys, content types, and a requested output profile, then return `202 Accepted` with a stable job ID. A client can poll `GET /batches/{id}` every two seconds, or use server-sent events if it needs a live dashboard. Either way, the response should include `done`, `total`, and item counts so a reconnect does not reset the bar to zero.

Here is the deliberately boring TypeScript shape I use at the boundary. It has no image library hidden inside the route and no promise that one request will finish the whole event.

```ts
import express from "express";

type BatchState = "queued" | "processing" | "complete";
type Batch = {
  id: string;
  total: number;
  done: number;
  state: BatchState;
  failed: number;
};

const app = express();
app.use(express.json({ limit: "256kb" }));

const batches = new Map<string, Batch>();

app.post("/batches", async (req, res) => {
  const files = Array.isArray(req.body?.files) ? req.body.files : [];
  if (files.length === 0 || files.length > 1000) {
    return res.status(400).json({ error: "files must contain 1 to 1000 entries" });
  }

  const id = crypto.randomUUID();
  batches.set(id, { id, total: files.length, done: 0, failed: 0, state: "queued" });
  await enqueueBatch({ id, files });
  return res.status(202).json({ id, statusUrl: `/batches/${id}` });
});

app.get("/batches/:id", (req, res) => {
  const batch = batches.get(req.params.id);
  if (!batch) return res.status(404).json({ error: "batch not found" });
  return res.json({ ...batch, percent: Math.floor((batch.done / batch.total) * 100) });
});

async function enqueueBatch(input: { id: string; files: unknown[] }): Promise<void> {
  // Send this to a durable queue in production; the route remains fast either way.
  void input;
}

app.listen(3000);
```

The in-memory map is only a readable contract for the example. A restart would erase it, so production state belongs in a durable database or job store. That distinction matters for progress: a percentage computed from process memory can lie after a deploy.

## Storage and cache rules that keep the batch affordable

Keep three identities separate: the original upload, the normalized working image, and the delivery derivative. Never use a filename as a cache key. Camera exports routinely reuse names such as `IMG_0001.JPG`; the key should include a content hash or an upload ID, plus the transformation profile and encoder version.

For event photos, I retain originals for the customer’s agreed window, keep medium derivatives for the gallery, and expire transient working files quickly. The policy is a data contract, not a cleanup cron you hope runs. Record the expiry decision next to the object metadata so an operator can explain why a preview disappeared.

The cache should be content-addressed and immutable. If a resize profile changes from `cover-1600-v1` to `cover-1600-v2`, those are two keys. Overwriting the old object creates a race between a browser cache and a worker that is still reading it. It also makes a retry hard to reason about.

Measure bytes, not just object counts. Six hundred thumbnails can be cheap while six hundred uncompressed intermediates quietly consume the budget. Emit `input_bytes`, `output_bytes`, `cache_hit`, and `duration_ms` per item. Then calculate the retained-byte curve for a typical event instead of arguing about a nominal per-request price.

## A worker loop with bounded concurrency

Concurrency is a memory setting. If decoding one 24-megapixel JPEG expands to tens of megabytes, launching 100 decoders can take down the worker before the CPU becomes the bottleneck. Start with a small limit, measure resident memory, and raise it only when the queue stays healthy.

This worker uses a generic `processOne` function so the storage and image libraries remain replaceable. The important behavior is idempotency: the same item and profile can be retried without producing a second logical result.

```ts
type Item = { sourceKey: string; profile: string };
type Result = { status: "ready" | "failed"; outputKey?: string; reason?: string };

async function runBatch(items: Item[], limit: number, onResult: (r: Result) => Promise<void>) {
  let next = 0;
  async function worker() {
    while (true) {
      const index = next++;
      if (index >= items.length) return;
      const item = items[index];
      try {
        const outputKey = await processOne(item); // Reads source, writes an immutable derivative.
        await onResult({ status: "ready", outputKey });
      } catch (error) {
        const reason = error instanceof Error ? error.message : "processing failed";
        await onResult({ status: "failed", reason });
      }
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
}

declare function processOne(item: Item): Promise<string>;
```

Do not report `done += 1` before the derivative is durable. That tiny ordering mistake produces a progress bar that reaches 100% while the last URLs still return 404. I learned to test this with a fake storage adapter that delays the commit; it catches the race without requiring a real gallery.

## Failure handling and the cases for a queue

Retries need a budget and a category. A malformed image should become a terminal item failure after validation. A temporary network timeout can be retried with backoff. Put the error class and attempt count in the item record, then expose a redacted reason to the browser. Avoid returning raw decoder messages: they can contain local paths or metadata.

The catch is operational weight. A queue, worker fleet, dead-letter flow, and event stream are not suitable when a small team processes a few dozen images a week and can tolerate polling. Stick with a single durable job table and a scheduled worker in that case. Conversely, move to a real queue when jobs overlap, deploys routinely interrupt work, or one slow customer can block everyone else.

I am not sure there is one ideal progress transport. Polling is easier to cache and replay; server-sent events feel better for an operator watching a live ingest; WebSockets add bidirectional machinery that this workflow rarely needs. Your mileage may vary with mobile clients and corporate proxies. Decide from reconnect behavior and observability, not from the novelty of the protocol.

The practical rule is simple: make the upload resumable, make each item idempotent, and make retention visible. Once those are true, Express can stay a thin front door while workers spend their time on pixels instead of holding sockets open.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
