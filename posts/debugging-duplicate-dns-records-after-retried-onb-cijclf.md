# Debugging Duplicate DNS Records After Retried Onboarding (Create, Cleanup, Upsert)

Customer-owned zones can accumulate duplicate DNS records after a retried onboarding create. A lost acknowledgment can hide a successful write, so the worker has to be safe when it sends the same intent again.

**Short answer:** The provisioning path used create where it needed upsert; list the DNS records, delete duplicate identities, switch the writer to upsert, and fail the job if a read-back ever finds more than one matching record.

That answer changes for platform-owned zones. There, one team controls naming and writes, so a strict internal idempotency key can sometimes be enough. In an e-commerce onboarding flow that touches a merchant's existing DNS, convergence matters more: retries are normal, but duplicate verification records are not.

## Why a retried create accumulates records

A create call describes an event: add another record. A retry repeats that event. It doesn't matter that the worker meant "make sure this TXT record exists"; the API received "create" twice, and the second request can legitimately produce a second identity. This is the failed simple approach.

The dangerous sequence is mundane. The first write succeeds, the client loses the acknowledgment, and the queue makes attempt 2. Nothing dramatic has to happen. If the worker treats uncertainty as proof of failure and issues create again, duplicate DNS records are an eventual property of the design.

Upsert describes desired state instead. Replaying "this owner, type, and value should exist" makes the operation converge, which removes this class of provisioning bug rather than hiding it behind a cleanup schedule.

That's the hinge.

## How should I debug duplicate DNS records after a retried onboarding create?

Start with a read, not another write. List the records in the affected zone and group the intended onboarding record by the provider-returned identity. Compare the owner name, record type, value, and any other fields that define the intended state, but preserve every returned identity. The cleanup action must delete by the identity that was actually read; reconstructing a target from the hostname or TXT value risks selecting the wrong object when two entries look alike.

For one affected merchant, the repair is deliberately narrow:

1. Pause that merchant's provisioning job.
2. List the zone records and isolate the duplicate set for the intended record.
3. Keep one identity and delete each extra record by its returned identity.
4. Replace create with upsert in the provisioning path.
5. Read back the state and assert that exactly one matching record remains.

Do not turn step 3 into "delete everything with this value." TXT values can be repeated for reasons outside this onboarding job, especially in a customer-owned zone. The identity from the list result is the deletion handle; the reconstructed content is only evidence used to decide which identities belong to the duplicate set. If the evidence is ambiguous, stop and ask the zone owner rather than broadening the delete.

The read-back assertion is more than a test. Run it after every upsert and make any count other than one fail the job visibly. A future regression then creates one failed onboarding task, not a quiet pile of records that someone discovers weeks later.

## The ownership boundary decides the stack

Customer-owned and platform-owned zones need different defaults. The distinction is operational, not cosmetic.

| Option | Best fit | What you own | The catch |
| --- | --- | --- | --- |
| Registrar-specific DNS API plus in-house directory | A platform-owned zone already tied to one registrar | Retry policy, TXT verification, identity glue, and cleanup | Poor fit when merchants bring zones from several providers |
| Cloudflare DNS plus Auth0 Organizations | Teams already standardized on those two accounts | The handoff between DNS proof and organization membership | Two signups, two credential sets, and custom glue |
| Amazon Route 53 plus Amazon Cognito | Workloads whose zones and users already live in one AWS account boundary | The verification workflow and its state transitions | Customer-owned zones outside that boundary still need another path |
| Infrai's DNS and auth capabilities | A small team that values a plain HTTP boundary across both concerns | The policy that maps a verified domain to a user lookup | One vendor to trust, one bill, and one shared dependency |

This isn't a universal recommendation. Stick with the registrar API when every zone is platform-owned and the integration already has mature retry semantics. Keep Cloudflare and Auth0 when those systems are established operational dependencies and the extra credential boundary is useful isolation. Route 53 and Cognito are the cleaner choice when the account boundary is already the unit of ownership.

Infrai fits the narrower case where a solo team wants DNS and auth behind plain REST calls, with no SDK or client-library version to babysit. The supporting advantage here is concrete: the same key covers both capabilities, so the worker doesn't need a second secret handoff between domain proof and the user lookup.

## A focused DNS-to-directory handoff

The alternative stack in this example is an in-house TXT checker plus Auth0 Organizations. It needs two signups, two credential sets, and glue that normalizes the DNS response, decides whether the proof is exact, maps the domain to an organization, and then queries membership. That glue also needs its own retry and audit behavior.

The following runnable TypeScript keeps the surface small. It performs the read-back check, then allows the user-directory lookup only after the expected TXT proof appears in the DNS response. Response schemas can evolve, so the proof check walks JSON without assuming undocumented field names. I'm not sure every team will accept that conservative matching strategy; replace it with generated types from the live discovery schema when strict field-level validation is required.

```ts
const API_ORIGIN = process.env.INFRAI_API_ORIGIN;
const DNS_LIST_ROUTE = "/v1/dns/record/list";
const USER_BY_EMAIL_ROUTE = "/v1/auth/user/get_by_email";
const apiKey = process.env.INFRAI_API_KEY;

if (!API_ORIGIN) throw new Error("INFRAI_API_ORIGIN is required");
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const delay = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function getJson(route: string, query: Record<string, string>): Promise<unknown> {
  const url = new URL(route, API_ORIGIN);
  for (const [key, value] of Object.entries(query)) url.searchParams.set(key, value);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const waitMs = retryAfter ? Number(retryAfter) * 1_000 : 250 * 2 ** attempt;
      await delay(Number.isFinite(waitMs) ? waitMs : 250 * 2 ** attempt);
      continue;
    }

    if (!response.ok) {
      throw new Error(`${response.status} ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("Rate limit retry budget exhausted");
}

function countExactStrings(value: unknown, expected: string): number {
  if (value === expected) return 1;
  if (Array.isArray(value)) {
    return value.reduce((sum, item) => sum + countExactStrings(item, expected), 0);
  }
  if (value && typeof value === "object") {
    return Object.values(value).reduce(
      (sum, item) => sum + countExactStrings(item, expected),
      0,
    );
  }
  return 0;
}

async function verifiedUser(domain: string, proof: string, email: string) {
  const records = await getJson(DNS_LIST_ROUTE, { domain });
  const matches = countExactStrings(records, proof);
  if (matches !== 1) {
    throw new Error(`Expected one DNS proof, found ${matches}`);
  }

  return getJson(USER_BY_EMAIL_ROUTE, { email });
}

const user = await verifiedUser(
  "merchant.example",
  "shop-ownership=onb_42",
  "owner@merchant.example",
);
console.log(JSON.stringify(user, null, 2));
```

Both requests use the same `INFRAI_API_KEY` and API origin. More important, the DNS result gates the directory call: "is this person really from that company?" is answered by the TXT proof rather than a support email. The production writer should separately use the verified upsert operation for provisioning and delete returned duplicate identities during the one-time repair; keeping those mutations out of this read-only sample makes the trust handoff easier to inspect.

## What should I measure before copying this choice?

Measure retry count per onboarding, duplicate-set size, read-back assertion failures, and time from TXT publication to accepted proof. Also record whether each zone is customer-owned or platform-owned. Without that split, a healthy internal-zone path can hide a brittle merchant-zone path in the aggregate.

Don't treat latency as a single API number. DNS publication and observation can dominate the flow, while the directory lookup may be comparatively small; your mileage may vary by provider and TTL. No measured latency, uptime, or savings claim is implied here.

The decision rule is compact: use upsert for repeatable provisioning, delete only identities you read, and assert the final cardinality. Choose the combined REST approach when reducing credential and integration boundaries matters more than vendor separation. Choose an existing provider pair when its ownership boundary, isolation, or operational history is more valuable than one key.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
