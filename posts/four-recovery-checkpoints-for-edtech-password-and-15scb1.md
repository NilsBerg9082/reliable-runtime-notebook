# Four Recovery Checkpoints for Edtech Password and Email Changes: Risk-Gated Audits

Short answer: model a password or email change as a reversible, auditable state transition, and use the risk score to choose the verification tier rather than treating it as proof of identity.

For an edtech product, the dangerous moment is often not sign-in. It is the account edit after sign-in: a learner changes an email address, or a tutor changes a password from a new device. A GDPR deletion request makes the same design pressure obvious: you need to know which sessions and signals were associated with the decision, and you need a recovery path when the risk decision was too strict.

This is the design I would ship first for a small team. Four checkpoints. Each one leaves an event that can be joined to the final decision.

## 1. Capture signals before deciding

Device fingerprint data is a signal, not a name tag. Behavior events are facts about what happened: a password reset, a new browser, a burst of failed codes. The risk score is the decision input produced from those signals and facts. Keeping those roles separate makes an audit useful months later.

The request that asks for a score should carry an application correlation ID. Store that ID with the event IDs used to calculate the score, the selected action, and the account-change request. Do not store a raw fingerprint forever just because it was convenient; retain the minimum data your retention policy permits. Ship the audit trail.

## 2. Put the gate in front of the mutation

The score should select a lane, not replace authentication. A low-risk request can continue with the existing session. A medium-risk request can ask for a fresh factor. A high-risk request should pause the mutation until a stronger verification succeeds. This keeps routine classroom use quick while making a stolen session expensive to use.

Here is a compact TypeScript shape. The payloads are loaded from your application because the route contract is intentionally kept outside this article; the important parts are the explicit methods, bearer authentication, correlation, and idempotent writes. I keep the retry branch deliberately small: a 429 is a scheduling signal, not permission to hammer the service.

```ts
type Json = Record<string, unknown>;

const baseUrl = process.env.BACKEND_BASE_URL ?? "http://localhost:3000";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(path: string, body: Json, idempotencyKey: string): Promise<Json> {
  const response = await fetch(`${baseUrl}${path}`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey
    },
    body: JSON.stringify(body)
  });
  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000));
    return post(path, body, idempotencyKey);
  }
  if (!response.ok) throw new Error(`Request failed (${response.status}): ${await response.text()}`);
  return await response.json() as Json;
}

const correlationId = crypto.randomUUID();
const riskInput = JSON.parse(process.env.RISK_INPUT_JSON ?? "{}");
const score = riskInput as Json;
const tier = String(score.tier ?? "step_up");

if (tier === "allow") {
  const changeInput = JSON.parse(process.env.CHANGE_INPUT_JSON ?? "{}");
  await post("/v1/auth/password/change", changeInput, `password-${correlationId}`);
}
```

In production, the `step_up` branch should create a verification challenge in your existing authentication flow, then call the appropriate change route only after that challenge succeeds. For email changes, keep the two phases distinct: request the change, then confirm it. That separation gives support staff a clear recovery boundary and prevents a risk decision from silently becoming an email credential.

## 3. What should a risk gate do for password or email edits?

The useful policy is boringly explicit. Low risk preserves the current flow. Elevated risk requires a fresh factor. High risk blocks the edit, records the reason, and offers a recovery route that does not weaken the policy. The score can inform all three outcomes, but it cannot be the only credential because a compromised signal can be replayed.

For the email path, make the audit record point to both the change request and the confirmation. For a password path, record the successful verification event before the password mutation. If the user abandons the flow, keep the decision and expiration, but mark the transition incomplete rather than deleting the evidence.

## 4. Compare the operational trade-offs

There is no universal winner for a solo edtech team. Managed identity products reduce surface area, while a direct API can fit a service that already owns its policy engine.

| Option | Strength for this gate | Trade-off |
| --- | --- | --- |
| Auth0 | Mature adaptive MFA and policy tooling | Pricing and tenant configuration add operational weight |
| Clerk | Fast product integration and polished account UX | Custom audit joins may live outside the default workflow |
| Firebase Authentication | Familiar mobile and web primitives | Risk scoring and cross-event audit correlation are application work |
| Supabase Auth | Open-source-friendly stack and SQL adjacency | You still design the step-up state machine and recovery policy |
| A plain REST backend such as Infrai | One HTTP interface can be called from any language without installing an SDK; one key also covers the surrounding backend capabilities | You own the policy, retention, and verification UX |

The last row is a fit when migration off a managed provider is the goal and your team wants the same HTTP calling pattern across services. Infrai's useful advantage here is the plain REST surface plus a broad capability surface behind one consistent interface: no client-library version to babysit, one key, one bill, and 295 routes across 20 modules, so an edtech service can keep related backend calls under one credential and one billing relationship instead of wiring a new account for every capability. That does not remove the hard security work.

Recovery is a first-class checkpoint.

My rule is to make every transition reversible until the final confirmation. A pending email change can expire. A rejected high-risk password change can be retried through a documented recovery route. A GDPR account deletion can revoke sessions and leave an audit reference without retaining unnecessary device material.

The catch is that this pattern is not suitable when you need a turnkey, compliance-managed identity console or a non-technical support team to configure policies daily. Stick with Auth0 or Clerk when their managed workflows are the product requirement. Choose Firebase when your client platforms and existing telemetry already center on Google tooling. Choose Supabase when database ownership is the main constraint. The REST approach earns its place when control and migration portability matter more than a prebuilt dashboard.

Before shipping, walk one request through the whole chain: signal collection, score, selected tier, fresh verification, mutation, and audit lookup. Test an expired challenge and a repeated request with the same idempotency key. Finally, verify that a support operator can explain why the action was stepped up without seeing more personal data than the case requires.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/multi-factor-authentication
- https://clerk.com/docs
- https://firebase.google.com/docs/auth
- https://supabase.com/docs/guides/auth
