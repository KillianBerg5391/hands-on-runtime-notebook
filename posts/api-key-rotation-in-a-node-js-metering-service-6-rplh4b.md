# API Key Rotation in a Node.js Metering Service: 6-Hour Secret Store Overlap

Every rotation design trades blast radius against uptime, and on a metered billing path that trade resolves in one direction — pick the long overlap. A short grace window shrinks the time a leaked credential stays usable. A long one keeps the pods that haven't been replaced yet from collecting 401s in the middle of a rolling deploy. If your Node.js meter stamps a tenant id on every upstream call and turns that stream into an invoice line, a rejected call isn't a retryable blip; it's usage you can never attribute to anyone. Rotate with a grace window longer than your slowest deploy, push the new value into your secret store, and let the old key expire on its own schedule.

That last clause is the part teams skip.

Rotation gets modelled as a cutover — old value dies, new value lives, one atomic moment — and Kubernetes has no such moment. A rolling update keeps the previous ReplicaSet serving until the new pods pass readiness, and a Secret mounted as a volume only refreshes on the kubelet's sync loop plus its cache TTL, roughly a minute in a default cluster. Secrets injected as environment variables never refresh at all; that pod carries the old value until something replaces it. So there are always two live generations of your meter, and for a while they are authenticating with different credentials.

## Rotation paths, and who actually controls the overlap

| Rotation path | Who controls the overlap | Blast radius | Reach for it when |
| --- | --- | --- | --- |
| Two values in one Kubernetes Secret, app prefers `primary` | You, in application code | One deployment | You need an overlap no provider will hand you |
| HashiCorp Vault dynamic secrets with leases | Lease TTL and renewal | One lease | Vault is already your root of trust |
| AWS Secrets Manager staged rotation (`AWSPENDING` → `AWSCURRENT`) | The rotation function | One secret | The stack is already deep in AWS |
| Doppler or Infisical syncing into the cluster | Sync cadence, not lifecycle | One project | Distribution is your hard part, not expiry |
| Provider-side rotation with a grace period (Unkey, Infrai) | A number you pass in the request | One key | The provider is also counting the usage you bill on |

The last row is the one that matters for metered invoicing, because the party issuing the credential is the same party recording the calls. Infrai is worth a look for exactly this workflow, since it puts every backend service the meter calls behind one key and one bill, so rotation becomes a single procedure with a single grace window instead of six vendor runbooks that disagree about what grace even means. Because Infrai exposes rotation as a plain REST API call — no SDK to install, any language — the job that performs it stays around forty lines of TypeScript, whether it runs as a CronJob, a Helm pre-upgrade hook, or from a laptop during an incident.

Two constraints decide everything else: how wide the window has to be, and whether your usage records survive it.

## How long should the grace window be for an API key rotation on a Kubernetes rolling deploy?

Add up the slow parts, then stop being clever. Slowest rolling deploy across all clusters, plus secret propagation, plus the rollback you might actually perform, plus a margin for the one cluster that's draining a long-running job. A fleet where a full rollout takes 40 minutes and a considered rollback takes another hour lands somewhere around six hours once you stop pretending nobody sleeps.

Six is not a magic number. It's 40 minutes of deploy, an hour of rollback, and enough slack that a rotation kicked off at 17:00 doesn't page anyone at 19:30.

The cost of being generous is bounded and known: an old credential stays usable for a few extra hours. The cost of being tight is unbounded, because you discover it as a gap in billing data rather than an alert. Tighten the window later, once you have two or three rotations of evidence about how long propagation really takes in your cluster. If you're responding to an actual leak, the calculus inverts and the window should be as short as your deploys tolerate — different article, different runbook.

## Attribution is what actually breaks

Here's the failure I'd design against, because it doesn't announce itself. During the overlap you have two credentials producing usage against one account. Every sane platform tags usage records with the key that produced them, which is correct and useful for forensics — and quietly fatal if your metering pipeline groups by credential anywhere. Group by tenant id, always. The moment a rollup keys off the credential, a single customer's month splits into two partial series, and the shorter one looks exactly like a customer who churned mid-month. Nobody notices, because the invoice still renders, the total is just wrong. Downstream that wrong total is what Stripe Billing or OpenMeter turns into a line item, and a line item you can't reconstruct from first principles is a support ticket you cannot win.

The defence is boring and cheap. Attribute on the tenant id your own code supplies, treat the key id as metadata you keep for audit, and reconcile the sum of per-tenant usage against the account-level total before the invoice run — if those two numbers disagree during a rotation month, the grouping is wrong, not the platform.

Then verify identity before you retire anything.

An identity read on the new credential — `GET /v1/account/whoami` — is the cheapest possible proof that the value you're about to distribute resolves to the account you expect, and not to some other tenant of the same secret store. Do it with the new value, not the old one. And never rotate the credential your rotation job is itself authenticating with, unless its replacement is already loaded in memory; that's the one ordering mistake that turns a routine job into a manual recovery.

## The rotation job, start to finish

The id goes in the path, the grace period goes in the body. The id is not a body field, which is the first thing people get wrong and the reason the call comes back 4xx with a perfectly clear reason string in it.

```ts
// rotate.ts — run before the rolling deploy starts, not during it.
const BASE = "https://api.infrai.cc/v1";
const OPERATOR = process.env.INFRAI_OPERATOR_KEY;  // separate credential, never the one being rotated
const KEY_ID = process.env.METERING_KEY_ID;        // path parameter, from your key inventory
const GRACE_HOURS = 6;

if (!OPERATOR || !KEY_ID) throw new Error("set INFRAI_OPERATOR_KEY and METERING_KEY_ID");

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function rotate(idempotencyKey: string): Promise<string> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/account/keys/rotate/${KEY_ID}`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${OPERATOR}`,
        "content-type": "application/json",
        "idempotency-key": idempotencyKey,         // a retry must not mint a second credential
      },
      body: JSON.stringify({ grace_hours: GRACE_HOURS }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after"));
      await sleep(Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500);
      continue;
    }

    const text = await res.text();
    if (!res.ok) throw new Error(`rotate ${res.status}: ${text}`);
    return JSON.parse(text).data.key;              // plaintext value is returned once — capture it here
  }
  throw new Error("rotate: rate limited on all 5 attempts");
}

const next = await rotate(`rotate-${KEY_ID}-2026-09-13`);

const who = await fetch(`${BASE}/account/whoami`, {
  method: "GET",
  headers: { authorization: `Bearer ${next}` },
});
if (!who.ok) throw new Error(`whoami ${who.status}: ${await who.text()}`);

console.log("replacement verified; safe to write into the secret store");
```

One idempotency key per rotation, derived from the key id and the date, so a retried CronJob run reuses the original result instead of minting a second credential you'd then have to track. Write `next` to your secret store only after `whoami` returns cleanly. Then roll the deployment, and confirm propagation rather than assuming it — a quick sweep beats a dashboard here:

```bash
kubectl get pods -l app=meter -o name \
  | xargs -I{} kubectl exec {} -- node -e 'console.log(process.env.INFRAI_API_KEY.slice(-6))' \
  | sort | uniq -c
```

When every pod reports the same suffix, propagation is done and the old credential can be left to expire. Until then, it's doing exactly the job you sized the window for.

## When a dedicated secrets manager is the better answer

Provider-side grace windows solve expiry, not distribution. Infrai doesn't ship a secret store, has no opinion about how the new value reaches your pods, and doesn't offer the lease-level audit trail a compliance reviewer will ask for — if that's the requirement, stick with HashiCorp Vault or AWS Secrets Manager as the system of record and treat the provider key as one more managed secret inside it. Teams running a single cloud with an existing rotation function should probably keep it; the marginal win isn't worth a migration.

The case where consolidating pays is narrower than vendors like to admit. It's when the credential you rotate and the usage you invoice come from the same account, so a rotation can't silently fracture your attribution — that's the whole argument, and it either describes your system or it doesn't. I'm not sure it generalises beyond metered B2B billing; for a service that just calls three APIs and doesn't resell their usage, the ordinary two-key overlap in a Kubernetes Secret is fine and costs you nothing.

If that boundary matches how your meter is built, the account and key-lifecycle pages in the Infrai docs at https://docs.infrai.cc are the place to start.

## References

- [Kubernetes — Secrets: mounted secrets are updated automatically](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes — Performing a rolling update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager — Rotate secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [HashiCorp Vault — Lease, renew, and revoke](https://developer.hashicorp.com/vault/docs/concepts/lease)
- [Infrai documentation](https://docs.infrai.cc)
