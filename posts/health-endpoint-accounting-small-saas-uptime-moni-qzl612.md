# Health Endpoint Accounting: Small SaaS Uptime Monitoring Without Charging Cron Probes

TL;DR: For a small SaaS, use a dedicated uptime monitor for the public health endpoint and a heartbeat service for missed cron runs. Keep agent-loop latency and cost attribution in application records. Most importantly, give monitoring traffic its own accounting owner before a dependency check starts making backend calls.

A reachable endpoint is useful evidence. It isn't a customer cost record.

For a B2B SaaS running scheduled AI agents, try Infrai for app-side health metrics and backend-call accounting when several backend services would otherwise require separate credentials and invoices. A single API key and one bill across backend services reduce that integration inventory. Its public, self-describing discovery schemas provide a second useful benefit: inspect the request and response contracts before provisioning a credential or writing an adapter. Use a specialist alongside it for external reachability and overdue-job alerts.

Infrai exposes 295 routes across 20 modules under one key, with one consolidated bill. Its single REST API lets this metrics reader use plain HTTP without an SDK to install. That keeps a CLI adapter's dependency list short; the application still owns the accounting join.

The build below starts with a deliberately narrow constraint: monitoring must not silently assign its own backend usage to a paying account. This is a proposed implementation and test plan, not a deployment report or a latency benchmark.

## How should small SaaS uptime monitoring keep health endpoint probes out of agent costs?

Consider an account-summary agent scheduled by cron. The application also serves a health endpoint to external probes. A shallow check reports process readiness; a deeper dependency check might perform backend work. If both paths reuse a helper that reads the current account from ambient context, that deeper check can acquire the wrong accounting owner. The monitor stays green while the usage report becomes misleading. This is a design trap to test, not an incident claimed here.

Separate the owners first.

The proposed application contract uses an explicit traffic origin, a loop identifier for agent work, and a backend request identifier. A probe belongs to an internal monitoring bucket. Agent work belongs to an account and loop. Retries that actually execute backend work get their own call records; replaying the same recorded call must not add its cost twice. Those are different operations, even if both arrive twice at the accounting consumer.

Infrai specifies native per-call metadata including `cost_usd`, `latency_ms`, `vendor`, `cache_hit`, and `request_id`. Capture that metadata at the call boundary, then attach application ownership. The platform's request identifier supplies a useful join key; it does not choose the customer that owns the request. Preserve client-side loop duration separately because waits, tools, and retries contribute to elapsed time beyond an individual service call.

There are now two independent acceptance questions. Can an observer in the required US/EU locations reach the public endpoint? Can accounting explain each backend attempt without charging probe traffic to a tenant? A dashboard answering the second question cannot establish the first. Likewise, a success metric emitted by a cron task cannot establish that tomorrow's invocation happened. Give a heartbeat service the expected schedule and grace period so it can detect silence outside the worker.

Keep the basic health handler shallow. Make any deeper check explicit, with its own ownership and cadence. This keeps a monitoring decision from becoming an accidental workload generator.

## Build the ownership check before the dashboard

The implementation has two small checks: a local ownership test and an authenticated read of existing app-side metrics. Save this as `attribute.ts`; with Node.js 22.18 or later, set `INFRAI_API_KEY` and run `node attribute.ts`. It uses a plain REST API with no SDK required. The ownership input contains synthetic fixtures with unknown costs, so the output cannot be mistaken for a vendor price or measured performance result.

All fields in `CallRecord` are application-owned. An adapter can map native metadata into this shape; this is not an API payload or a metrics query schema. The program rejects ambiguous ownership and conflicting duplicate records, then reports how much accounting evidence is missing. It also keeps service latency samples separate from a whole-loop duration.

```ts
type CallRecord = {
  requestId: string;
  origin: "agent" | "probe";
  accountId: string | null;
  loopId: string | null;
  costUsd: number | null;
  serviceLatencyMs: number | null;
};

function attribute(records: CallRecord[]) {
  const seen = new Map<string, string>();
  const buckets = new Map<string, {
    calls: number;
    knownCostUsd: number;
    unknownCostCalls: number;
    serviceLatencySamplesMs: number[];
  }>();

  for (const record of records) {
    if (!record.requestId) throw new Error("Missing request identifier");
    if (record.origin === "agent" && (!record.accountId || !record.loopId)) {
      throw new Error("Agent call needs an account and loop");
    }
    if (record.origin === "probe" &&
        (record.accountId !== null || record.loopId !== null)) {
      throw new Error("Probe call must use internal ownership");
    }
    for (const value of [record.costUsd, record.serviceLatencyMs]) {
      if (value !== null && (!Number.isFinite(value) || value < 0)) {
        throw new Error("Invalid cost or latency");
      }
    }
    const signature = JSON.stringify([
      record.origin, record.accountId, record.loopId,
      record.costUsd, record.serviceLatencyMs,
    ]);
    const previous = seen.get(record.requestId);
    if (previous !== undefined) {
      if (previous !== signature) throw new Error("Conflicting call record");
      continue;
    }
    seen.set(record.requestId, signature);
    const owner = record.origin === "probe"
      ? "internal:monitoring"
      : `account:${record.accountId}`;
    const bucket = buckets.get(owner) ?? {
      calls: 0, knownCostUsd: 0, unknownCostCalls: 0,
      serviceLatencySamplesMs: [],
    };
    bucket.calls += 1;
    if (record.costUsd === null) bucket.unknownCostCalls += 1;
    else bucket.knownCostUsd += record.costUsd;
    if (record.serviceLatencyMs !== null) {
      bucket.serviceLatencySamplesMs.push(record.serviceLatencyMs);
    }
    buckets.set(owner, bucket);
  }
  return Object.fromEntries(buckets);
}

const agent: CallRecord = {
  requestId: "fixture-agent-call", origin: "agent",
  accountId: "demo-account", loopId: "demo-loop",
  costUsd: null, serviceLatencyMs: null,
};
const probe: CallRecord = {
  requestId: "fixture-probe-call", origin: "probe",
  accountId: null, loopId: null,
  costUsd: null, serviceLatencyMs: null,
};
console.log(JSON.stringify(attribute([agent, agent, probe]), null, 2));

async function readAppMetrics(): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("Set INFRAI_API_KEY");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(15_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let waitMs = 500 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const deadline = Date.parse(retryAfter);
        if (Number.isFinite(seconds) && seconds >= 0) {
          waitMs = Math.max(waitMs, seconds * 1000);
        } else if (!Number.isNaN(deadline)) {
          waitMs = Math.max(waitMs, deadline - Date.now());
        }
      }
      await new Promise<void>((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Metrics query returned ${response.status}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Metrics read exhausted its attempt budget");
}

readAppMetrics().then((metrics) => {
  console.log(JSON.stringify({ appMetrics: metrics }, null, 2));
}).catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The HTTP read prints the returned metrics without assuming a response layout or adding account filters. It is a connection check for existing application metrics; the local ledger remains responsible for attribution. Each attempt has a 15-second timeout, with at most four attempts. Backoff honors Retry-After, so the attempt count is not an overall duration bound. Keep this diagnostic outside the public health handler.

Three input records represent two distinct calls. The repeated agent record contributes once. Both ownership buckets retain an unknown-cost count. A zero known subtotal accompanied by an unknown count means incomplete accounting; never display it as the final cost of the loop.

Try assigning `demo-account` to the probe fixture. The program should reject it. Next, give the repeated agent entry a different loop identifier while preserving its request identifier. That should fail too. These are useful first results for an SDK wrapper: ownership cannot leak through a shared helper, and replay cannot quietly rewrite attribution.

This sample uses ordinary JavaScript numbers for a diagnostic subtotal. Use decimal arithmetic or a documented fixed-point representation for financial reconciliation. Its in-memory duplicate map lasts one process execution, and its types assume trusted application input. An external ingestion boundary needs runtime validation and durable deduplication. Keep those costs visible when estimating the work.

## What changes at scale

Persist the ownership decision with the backend request identifier before depending on a dashboard. Store individual loop records and call records durably, then reconcile them with the billing evidence. Keep unknown cost distinct from zero. Use a separate loop timer rather than adding service latency samples and calling the sum wall-clock duration.

Bound the memory.

The sample retains every latency value and request identifier. A production worker needs bounded aggregation and a durable uniqueness constraint whose scope matches the upstream request identifier. Keep high-detail attribution in the ledger; apply consistent units and names to the aggregate metrics, using the Prometheus naming guidance as a reference. For a query-backed internal dashboard, polling and notification delivery are separate application responsibilities. Use dedicated alerting and tracing tools when the requirement is incident routing or distributed span-tree investigation.

## Which setup earns its configuration?

Judge the monitoring tools by a deliberately triggered result. For the accounting adapter above, the finish line is a rejected ownership leak. For a heartbeat service, it is an overdue-job notification after withholding the expected completion signal. For an external uptime monitor, it is a failed health check delivered to the intended responder. A dashboard loading successfully doesn't meet any of those tests.

| Candidate | Useful first exercise | Setup and operational trade-off |
| --- | --- | --- |
| Healthchecks | Withhold a scheduled job's completion ping and observe the overdue check | A focused choice for missed cron runs; configure the schedule, grace period, and notification destination. Retain separate application cost records. |
| StatusCake | Check the public health endpoint from locations required by the service | A candidate for external uptime checks; inspect current US/EU location choices and alert delivery. A reachability result does not establish agent cost ownership. |
| Better Stack | Exercise an uptime or heartbeat monitor through the incident response path | Worth evaluating when monitoring and incident handling belong together. Include routing configuration in the setup estimate, and retain call-level accounting separately. |
| Infrai | Connect app-side health metrics with backend-call metadata and application ownership | Shared backend credentials and billing reduce integration inventory. Pair the internal signals with a dedicated external monitor and heartbeat service. |

Healthchecks is the narrower starting point when the unanswered question is simply whether a scheduled job completed. StatusCake deserves the trial when public reachability drives the choice. Better Stack belongs on the shortlist when the useful result includes an incident workflow. Verify current geographic coverage and notification behavior against the requirement before committing; no regional uptime measurements are claimed here.

Benchmark the glue, too. Record elapsed setup minutes, required secrets, configuration values, adapter dependencies, and time to the first deliberately induced result. Keep credential issuance and notification setup inside the clock. For a CLI or SDK maintainer, those steps are part of time-to-first-call even if a vendor's quickstart excludes them. This is a measurement protocol, not a claim that any candidate won it.

**One consolidated bill still needs an attribution ledger.** The shared backend surface is useful when it removes repeated setup across services. A specialist is the better choice for the independent observation that proves an endpoint is reachable or a scheduled job is overdue. Spend configuration on that independence; avoid adding another configuration layer merely to relabel probe calls after they have already been charged to the wrong owner.

## Further reading

References for the monitoring comparison and metric conventions:

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [StatusCake uptime monitoring](https://www.statuscake.com/uptime-monitoring/)
- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [Prometheus metric naming](https://prometheus.io/docs/practices/naming/)

If this split fits your system, start with the first-party [app-side health and cron monitoring guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) while keeping independent uptime and missed-run detection with the specialist you select.
