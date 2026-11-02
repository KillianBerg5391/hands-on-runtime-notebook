# Error Tracking Across HTTP, Cron Jobs, and Queue Workers (3 Cohort Snapshots)

| Capture design | Reconstruction value | Main trade-off |
| --- | --- | --- |
| HTTP exception filter alone | Explains failed requests | Scheduled and queued work needs separate capture |
| Filter plus job wrappers and assignment snapshots | Connects failures to the experiment assignment used during execution | Requires a small context contract at each entry point |
| HTTP interceptor alone | Observes errors in the request stream it wraps | Does not cover independent background execution |

Short answer: use an HTTP exception filter plus explicit wrappers for cron jobs and queue workers, and attach an immutable experiment assignment to each failure. The extra context is worth carrying when a marketplace needs to compare tenant cohorts after an incident. A request-only view cannot answer what a background job actually evaluated.

This note answers the NestJS production error-tracking question through incident reconstruction: which tenant cohort saw a failure, which experiment revision assigned it, and which execution attempt failed? Filters and interceptors supply useful hooks. They don't supply the missing experiment history.

## Can you reconstruct the assignment that actually ran?

Start with evidence.

Fowler's discussion of experiment toggles describes routing users into cohorts and keeping their experience consistent. For a marketplace experiment assigned by tenant, apply that principle to the tenant assignment. Record the result when the application makes the decision. During reconstruction, querying today's configuration answers a different question: what would this tenant receive now?

Consider a deliberately synthetic fixture: tenant `market-a` receives the `treatment` variant of `seller-ranking` at revision `r17`. Its HTTP request enqueues a ranking refresh. Before that work executes, the experiment changes to `r18`. If the payload carries only the experiment name, a later lookup cannot establish which assignment the original request used. That is a concrete reconstruction gap, even if every thrown exception was captured successfully.

Choose the semantics before choosing the hook. If the worker must execute the request's decision, put the assignment snapshot in its payload. If the worker must evaluate the latest decision, capture that new assignment at execution and preserve the originating operation ID separately. Both policies can be valid. Mixing them makes the comparison ambiguous.

The first decision criterion is historical fidelity: can each failure identify the assignment used by that execution? A useful contract includes an opaque tenant key, experiment key, variant, assignment revision, operation ID, and execution path. Background attempts also need an attempt identifier. Treat these as an application contract; they are not fields the framework automatically provides. Never substitute a current flag lookup for a missing historical assignment.

## Count failed operations before comparing cohorts

The second criterion is denominator integrity. Three retries of one failed refresh shouldn't silently become three affected tenants. Keep attempt-level evidence for debugging, then define a separate comparison unit: unique operations, tenant operations, or tenants. Pick one explicitly.

An exception stream alone cannot establish a cohort failure rate. You also need the corresponding eligible executions or exposures, including successful ones. For the synthetic fixture, a comparison might count failed unique ranking operations divided by all eligible ranking operations within each variant and assignment revision. This is a proposed analysis definition, not a measured result. It does not by itself establish statistical significance or a causal treatment effect.

Expected HTTP exceptions need a policy too. NestJS represents HTTP errors with `HttpException`; its default exception handling distinguishes recognized HTTP exceptions from unrecognized errors. Keep the response status separate from the experiment outcome. A rejected request may be expected input validation, while a successful enqueue response may precede a failed ranking refresh. Equating HTTP status with the entire marketplace operation loses that distinction.

One reporting owner.

A filter can own HTTP exception reporting. An interceptor can collect request timing and exposure context without reporting the same exception again. NestJS interceptors wrap a request's observable stream, and exception filters handle uncaught exceptions; independent cron callbacks and queue processors need their own entry-point handling. Don't assume that registering a global HTTP filter installs a catch around arbitrary background functions.

## How should an HTTP filter and interceptor divide error tracking?

The example below defines application-owned fields and a synchronous, best-effort reporting interface. It preserves the original thrown value and lets the standard HTTP exception handler produce the response. The reporter must enqueue locally and return promptly; blocking network delivery inside `record` would add latency directly to the failing operation. Delivery retries belong in the reporting implementation (outside the application's exception path).

```ts
import { ArgumentsHost, Catch } from '@nestjs/common';
import { AbstractHttpAdapter, BaseExceptionFilter } from '@nestjs/core';

type Assignment = Readonly<{
  tenantKey: string;
  experiment: string;
  variant: string;
  revision: string;
}>;

type Evidence = Readonly<{
  assignment: Assignment;
  operationId: string;
  path: 'http' | 'cron' | 'queue';
  attemptId?: string;
}>;

type Failure = Readonly<{
  error: unknown;
  evidence?: Evidence;
}>;

type Reporter = { record(failure: Failure): void };

function report(
  reporter: Reporter, error: unknown, evidence?: Evidence,
): void {
  try {
    reporter.record({ error, evidence });
  } catch {
    // Reporting must not replace the application failure.
  }
}

@Catch()
export class CaptureFilter extends BaseExceptionFilter {
  constructor(
    adapter: AbstractHttpAdapter,
    private readonly reporter: Reporter,
  ) {
    super(adapter);
  }

  catch(error: unknown, host: ArgumentsHost): void {
    const request = host.switchToHttp().getRequest<{
      captureEvidence?: Evidence;
    }>();
    report(this.reporter, error, request.captureEvidence);
    super.catch(error, host);
  }
}

export async function captureWork<T>(
  reporter: Reporter,
  evidence: Evidence,
  work: () => Promise<T>,
): Promise<T> {
  try {
    return await work();
  } catch (error: unknown) {
    report(reporter, error, evidence);
    throw error;
  }
}
```

Register this HTTP-only filter with the application's HTTP adapter. Populate `captureEvidence` in a guard or another request hook before the handler runs, after authenticating the tenant and evaluating the assignment. Failures before assignment can still be reported with missing evidence; keep them in an unassigned bucket instead of guessing a cohort. Don't register this filter as a universal handler for non-HTTP transports.

For scheduled work, invoke `captureWork` inside the scheduled callback. If a cron run iterates over tenants, place the wrapper around each tenant's operation so the evidence identifies the tenant that failed. Decide separately whether the batch should continue after a tenant failure; the wrapper rethrows. For a queue, invoke it inside the processor with the validated payload snapshot and the current attempt ID. Returning that promise lets rejection reach the processor's caller and preserves its opportunity to apply its retry policy.

There is deliberately no generic payload spread into the error event. Validate the queued evidence, authenticate its producer, and copy only the approved fields. TypeScript's `Readonly` is a compile-time constraint, not runtime validation or a deep freeze. The reporter must serialize or copy evidence before buffering it, and sanitize errors before export: messages and stacks may contain sensitive data.

Small contracts beat configuration piles, but omission still costs something. The best-effort catch protects the application's exception; it cannot guarantee telemetry delivery. Give the reporter an independently observable dropped-event counter, a bounded buffer, and a documented overflow policy. Otherwise an empty dashboard can masquerade as a clean experiment.

## Benchmark the reconstruction, then test the release

A capture demo is cheap. Reconstruction is the test.

Use the synthetic `r17` to `r18` change as an acceptance fixture across all 3 paths. Trigger an HTTP exception, a rejected scheduled operation, and a rejected queue attempt. Assert that the response or rejection retains its original behavior, that each chosen capture boundary emits once per failed attempt, and that the reconstructed assignment matches the declared execution policy. Also test a failure before assignment, a throwing reporter, and overlapping work for different tenants. The last test catches shared mutable context that accidentally labels one tenant's failure as another's.

Then benchmark capture enabled against capture disabled under the same workload. Record request latency percentiles, processor throughput, event bytes, and reporter drops. Include normal execution, failure bursts, and a full reporting buffer. These are measurements to run, not results claimed here. Set acceptance limits from the service's existing latency and memory budgets; an arbitrary universal overhead promise is useless.

Deployment matters because queued work can outlive a release. Version the payload contract and keep consumers able to interpret the assignment evidence already in flight. Before a rollout, replay an older payload fixture through the new consumer. During reconstruction, distinguish application release, experiment revision, and processing attempt. They answer different questions.

Keep operation IDs and tenant keys in restricted event records rather than automatically making each one a metric label. Prometheus documentation cautions against high-cardinality labels. Retention, event volume, and access controls are practical engineering costs here; pricing comparisons add little to the reconstruction decision.

## When the smaller option is enough

The runner-up is a request-only filter. It is reasonable when the experiment's entire operation finishes within the HTTP request and no scheduled or queued continuation can change its outcome. Add exposure accounting and assignment evidence even then.

An interceptor is useful when the requirement is request timing or context around handler execution. If it also reports exceptions, explicitly assign reporting ownership so the filter doesn't duplicate those events. For a marketplace operation that crosses into background work, choose capture points by execution ownership and judge the result by whether an incident can be reconstructed from retained evidence. Hook count is a poor success metric.

## References

The sources below cover cohort routing, framework capture boundaries, asynchronous execution, and metric cardinality. The evidence schema, synthetic fixture, and acceptance procedure above are proposed application design.

## Sources

- Martin Fowler, Feature Toggles: https://martinfowler.com/articles/feature-toggles.html
- NestJS, Exception filters: https://docs.nestjs.com/exception-filters
- NestJS, Interceptors: https://docs.nestjs.com/interceptors
- NestJS, Task scheduling: https://docs.nestjs.com/techniques/task-scheduling
- NestJS, Queues: https://docs.nestjs.com/techniques/queues
- TypeScript Handbook, Object Types: https://www.typescriptlang.org/docs/handbook/2/objects.html
- Prometheus, Instrumentation best practices: https://prometheus.io/docs/practices/instrumentation/
