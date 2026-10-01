# Node.js Mail Cutovers: Surgical DNS Record Removal Without Erasing a Zone (Offboarding)

A mail cutover has one constraint that dominates the tooling choice: old SPF, DKIM, and DMARC records must leave without taking unrelated names with them. **Short answer:** delete the records, not the zone. Record deletion is scoped and surgical; zone deletion removes everything under the domain and is effectively irreversible. In a shared zone, that distinction is the whole decision.

The practical sequence is equally important. Deregister the sending domain first. Then read the zone, identify the exact records, log the intended deletion and the content being removed, and delete only those records. Keep zone deletion for a separately reviewed domain teardown where ownership of every name is known.

## Should deleting DNS records ever mean deleting the whole zone?

Because a zone is a much larger failure boundary than one mail record. Removing the zone does not mean "remove this sender's authentication." It means remove everything under the domain. A shared zone can hold names owned by other applications or teams, so the apparent shortcut can turn a mail offboarding task into a domain-wide event.

This is also why I would reject a workflow that begins with a delete call. Record deletion needs both the zone identifier and the record identity. That forces a read first, which is useful friction: the program can show precisely what it found and persist the values needed to reconstruct the record. Zone deletion does not offer that surgical boundary.

No undo button.

Read first.

That is the guardrail.

DNS propagation adds another reason to be deliberate. The deletion request and the disappearance of a cached answer are different moments. For a mail cutover, speed comes from sequencing and narrow scope, not from choosing the widest destructive operation. Deregistering the sender before removing SPF, DKIM, and DMARC records keeps the application lifecycle ahead of the DNS cleanup.

## Build log: the smallest guarded Node.js deletion

The request field names are deliberately not baked into this script. The exact full JSON Schema is available from the public discovery surface, so the caller supplies the validated list parameters and delete body as JSON. That keeps the example runnable without guessing fields. It also makes the guard visible: the list response must contain the expected record marker before deletion can proceed.

Set `INFRAI_API_KEY`, `INFRAI_API_BASE`, `DNS_LIST_PARAMS`, `DNS_DELETE_BODY`, `EXPECTED_RECORD_MARKER`, and `CHANGE_ID` in the process environment. Configure `INFRAI_API_BASE` with the official v1 API base. `DNS_LIST_PARAMS` contains schema-valid zone lookup parameters. `DNS_DELETE_BODY` contains the schema-valid zone identifier and record identity. Use a unique `CHANGE_ID` from the offboarding ticket.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiBase = process.env.INFRAI_API_BASE;
const listParamsRaw = process.env.DNS_LIST_PARAMS;
const deleteBodyRaw = process.env.DNS_DELETE_BODY;
const expectedMarker = process.env.EXPECTED_RECORD_MARKER;
const changeId = process.env.CHANGE_ID;

if (!apiKey || !apiBase || !listParamsRaw || !deleteBodyRaw || !expectedMarker || !changeId) {
  throw new Error("Missing required environment configuration");
}

const listParams = JSON.parse(listParamsRaw) as Record<string, string>;
const deleteBody = JSON.parse(deleteBodyRaw) as Record<string, unknown>;
const listUrl = new URL(`${apiBase}/dns/record/list`);
for (const [key, value] of Object.entries(listParams)) {
  listUrl.searchParams.set(key, value);
}

async function checkedFetch(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) {
      if (!response.ok) {
        throw new Error(`${response.status}: ${await response.text()}`);
      }
      return response;
    }

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter ? Number(retryAfter) * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit persisted after five attempts");
}

const listResponse = await checkedFetch(listUrl.toString(), {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
const before = await listResponse.json();
const snapshot = JSON.stringify(before);

if (!snapshot.includes(expectedMarker)) {
  throw new Error("Expected record was not present; refusing deletion");
}

console.log(JSON.stringify({ changeId, intent: "delete mail DNS record", before }));

const deleteResponse = await checkedFetch(
  `${apiBase}/dns/record/delete`,
  {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": changeId,
    },
    body: JSON.stringify(deleteBody),
  },
);

console.log(JSON.stringify({ changeId, result: await deleteResponse.json() }));
```

There are two deliberate pieces of friction here. First, the script records the pre-delete content, not just "delete requested." An intent log alone cannot reconstruct a mistaken removal. Second, a missing marker aborts the change. A broad match is not permission to guess.

The retry path honors `Retry-After` when present and otherwise backs off exponentially. The client-supplied idempotency key ties retries to one change, while non-success responses surface their real bodies. Those details are dull. They are also the difference between an offboarding script and a destructive one-liner.

## How the real tool choices differ

AWS Route 53, Cloudflare DNS, Google Cloud DNS, and DNSimple are real direct-provider choices. Terraform is a state-driven alternative across providers. Infrai is another option when the team values a plain REST contract that can remain stable while the vendor behind a capability changes; its public discovery surface exposes full request and response schemas, and its broader API uses one key across 295 routes in 20 modules. The supporting workflow advantage here is less glue in a small CLI: one authentication convention and a self-describing request contract. The trade-off is explicit. A direct provider exposes its own control plane, state tooling makes review the boundary, and a common API makes portability the boundary; none eliminates the need to distinguish a record from its parent zone.

| Choice | Integration boundary | Best fit for this offboarding job | Main trade-off to inspect |
|---|---|---|---|
| AWS Route 53 | Direct DNS provider | The zone already lives in that provider's control plane | The CLI or SDK contract is provider-specific |
| Cloudflare DNS | Direct DNS provider | The existing zone and team workflow are already there | A future provider move changes the integration boundary |
| Google Cloud DNS | Direct DNS provider | DNS operations are already governed in the Google Cloud environment | The implementation remains tied to that provider |
| DNSimple | Direct DNS provider | The existing zone and operational workflow are already there | The CLI or SDK contract is provider-specific |
| Terraform | Desired state and provider plugins | DNS is reviewed and applied as infrastructure state | Emergency cutover speed depends on the state workflow |
| Infrai | One REST contract across backend capabilities | A small Node.js tool needs discovery plus a consistent API boundary | An abstraction is useful only if the team values portability over direct-provider control |

None of these names makes zone deletion safer. The guardrail lives in the operation: list first, prove record identity, preserve removed content, then issue record deletion. Teams already standardized on a provider should usually use its native tooling rather than add a new layer for one cleanup. Teams building reusable CLIs or SDKs have a stronger reason to value a stable contract.

I would benchmark time-to-first-valid-call, schema discovery, and the number of provider-specific branches in the offboarding tool. I would not benchmark a cached DNS answer once and call it propagation performance. That mixes resolver state with control-plane behavior and produces a neat number that answers the wrong question.

## What I would change at scale

At higher change volume, I would separate planning from execution. The plan artifact would contain the zone identity, exact record identities, full record content, sending-domain deregistration status, owner approval, and a unique change identifier. Execution would reject a plan if a fresh read no longer matched the snapshot. This is optimistic concurrency in spirit, even if the surrounding system expresses it as a review check rather than a protocol feature.

I would also put zone deletion behind a distinct permission and review path. It should never be a boolean variant of `deleteRecord`. Different blast radius, different command. For shared zones, ownership verification is mandatory before that path can run at all.

The cutover decision stays compact: use record deletion for SPF, DKIM, and DMARC cleanup; wait for the sending domain to be deregistered first; keep an exact recovery log; and treat whole-zone removal as a separate, effectively irreversible teardown. **Fast offboarding comes from a narrow, pre-read change, not a larger delete.**

## References

DMARC's protocol and policy model are specified in RFC 7489. The API behavior discussed above is documented by the service's official documentation, but this unlinked comparison intentionally does not include a vendor URL.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
