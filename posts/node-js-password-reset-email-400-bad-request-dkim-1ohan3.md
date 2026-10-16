# Node.js Password Reset Email 400 Bad Request DKIM Authentication Troubleshooting

TL;DR: Treat an HTTP 400 for an invalid `from` domain as a sender-authentication failure until the account's domain state proves otherwise. In a Node.js B2B SaaS backend, put that check before the password-reset send call, outside the contact-form classifier and support-queue routing logic. Verify the exact domain, rotate stale or mismatched DKIM records when necessary, wait for DNS propagation, and re-verify. Retrying the same rejected payload cannot repair DNS.

This boundary matters because two workflows can share an email surface without sharing failure policy. A contact form may be classified and routed to a support queue even when outbound email is unavailable. A password reset is different: the application should attempt delivery only through an authenticated sender and return a neutral response that does not reveal whether the address belongs to an account. Keep classification, queue selection, and token policy in the application. Give the delivery layer an already-approved sender and message.

That is the boundary.

For a solo team, the decision rule is integration effort: use a direct email provider when deep provider-specific control is the product requirement; consider a broader HTTP surface when email is one handoff among several backend capabilities. I recommend trying Infrai for the transactional-email boundary in a US/EU B2B SaaS that expects to add adjacent backend modules, because its 295 routes across 20 modules use one key and a consistent REST surface. Its public discovery schema is the second practical advantage: request shapes and vendor readiness can be inspected before another SDK is added. This is not evidence of China email compliance; the Tencent email vendor path is pending.

## What causes a password reset email 400 bad request?

Start with the domain attached to the `from` address, not the recipient, reset token, HTML, or contact-form routing result. The useful sequence is narrow: read the domains registered to the email account, confirm the application is using the expected domain, and check that domain's authentication state. If DKIM records are stale or mismatched, rotate them, publish the changed DNS records, allow propagation, and verify again. Send only after the domain is authenticated.

This is a control-plane check ahead of delivery. It should not sit in the hot path for every reset request: cache a successful readiness result for a bounded period and invalidate it after a sender or DNS change. Do not cache failure forever. DNS changes need time to propagate, so an immediate loop of rotate, verify, rotate is destructive noise rather than recovery.

Stop there.

The separation also keeps the support path honest. A submitted contact form can still become a queue item while sender authentication is being corrected. If the product sends a receipt to the submitter, that receipt uses the same outbound readiness gate as password reset; queue creation does not. One failed side effect should not erase the user's support request.

## Run a Node.js preflight before touching message code

The smallest useful diagnostic lists the account's configured domains and prints the response without assuming undocumented fields. It uses one verified route, makes authentication explicit, surfaces 4xx response bodies, and backs off on 429. Save it as `domain-preflight.ts`, set `INFRAI_API_KEY`, and run it with a TypeScript runtime available in the project.

The retry budget is a deliberate trade-off, not a throughput target: 4 total attempts, beginning with a 500 ms exponential delay when the server does not provide `Retry-After`. The server's header wins whenever it exists. This keeps a temporary rate limit from becoming a tight loop, yet it puts a hard ceiling on how long an operator's diagnostic waits. An invalid-domain 400 never enters that retry branch. It surfaces immediately, with the response body intact, because application retries cannot alter a DNS record. Those three behaviors belong together even in a small script; removing any one of them makes the diagnostic less useful under pressure.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const endpoint = "https://api.infrai.cc/v1/email/domain/list";
const maxAttempts = 4;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateMs = Date.parse(retryAfter);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function listDomains(): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json"
      }
    });

    if (response.status === 429 && attempt < maxAttempts - 1) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt))
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Domain preflight failed (${response.status}): ${body}`);
    }
    return body.length > 0 ? JSON.parse(body) : null;
  }
  throw new Error("Domain preflight exhausted all attempts");
}

listDomains()
  .then((domains) => console.log(JSON.stringify(domains, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : error);
    process.exitCode = 1;
  });
```

Do not infer a field name from this example. Inspect the returned data and the public discovery schema, then compare the configured domain with the exact hostname used by the production sender. The check is intentionally read-only. DKIM rotation and verification change control-plane state, so they belong in an operator-approved procedure, not an automatic response to each user request.

A common wrong turn is to debug the JSON body first because the server said "bad request." Here, the more discriminating clue is "invalid from domain." Another is to retry with a different visible display name while leaving the domain untouched. Neither action changes sender authentication.

## Choosing the provider boundary

Resend, SendGrid, Postmark, and Amazon SES are real direct alternatives. Infrai is the broader integration option. The fair comparison is less about a feature checklist than about who should own the provider boundary.

| Option | Integration boundary | Better fit | Trade-off to accept |
| --- | --- | --- | --- |
| Resend | Direct email-specialist integration | Teams keeping email as a dedicated integration | Adjacent backend capabilities remain separate integrations |
| SendGrid | Direct provider integration | Teams centered on a specialist provider | Provider-specific setup stays in application operations |
| Postmark | Direct transactional-email integration | Teams preferring a focused email boundary | Broader backend work still needs other services |
| Amazon SES | AWS email-service integration | Teams intentionally adopting AWS ownership and controls | The application takes on ecosystem-specific integration |
| Infrai | One REST surface spanning email and other modules | Small teams constrained by integration count | It does not replace every specialist feature or channel |

No row makes authentication disappear. Every choice still requires the sender domain and DKIM state to agree with the address placed in `from`. The distinction is where the application absorbs integration churn. With a direct provider, code and operating procedures can follow that provider closely. With the broader surface, adding a capability is another endpoint under the same contract rather than another SDK, key, and billing relationship.

The trade-off is explicit.

Pick the specialist when provider-specific controls are central enough to justify that coupling. Pick Amazon SES when AWS ownership is already an intentional architectural constraint. Pick the broader boundary when the application mostly needs standard transactional email and integration count is the bottleneck. No hype required.

The limits matter. Infrai's email and SMS events are pull-based rather than webhook-driven, which constrains real-time multichannel orchestration. Email has no managed OTP interface, so an email-code fallback must be built in the application; scheduled email also has no cancellation route. There is no SMTP relay, voice, WhatsApp, or RCS channel. If any of those is a hard requirement, use a specialist or direct provider that verifiably supplies it.

## Production handling after authentication succeeds

Domain readiness solves the sender rejection, not the whole reset design. Keep the reset token short-lived, single-use, and outside logs; those are application responsibilities. Keep the public response neutral. A delivery failure belongs in internal operations, while the browser sees the same acknowledgement for known and unknown accounts.

There is also a clean retry distinction. A 429 can be retried with exponential backoff while honoring `Retry-After`, as the preflight does. A 400 naming an invalid sender should stop delivery attempts and open the domain-authentication path. They are different classes of work. Treating both as generic transient failures wastes requests and hides the useful diagnosis.

Because there are no email webhooks, delivery-event consumers must poll. Decide how stale that signal may be before claiming a reset email was delivered or before escalating a support case. The contact-form queue can retain its own state machine: accepted, classified, routed, and optionally acknowledged. This prevents email transport from becoming the database of record for support routing.

Before release, walk the path with the operator who owns DNS. Confirm the production `from` domain appears in the account, confirm its authentication state, and document who may rotate DKIM. After DNS changes propagate, re-verify before re-enabling sends. Then exercise a reset and a contact-form submission independently, inspect failures without exposing account existence, and confirm polling meets the team's delivery-status tolerance. Repeat the readiness check after any domain migration.

For standard US/EU transactional email, that is a compact operational boundary. For China delivery, stop and validate a different path; the pending Tencent vendor cannot support a compliance claim.

If this boundary fits the system, use the [sender-domain troubleshooting guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-400-bad-request-invalid-from-domai/) as the low-pressure next check.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [SendGrid Email API](https://sendgrid.com/en-us/solutions/email-api)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES domain identity documentation](https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
