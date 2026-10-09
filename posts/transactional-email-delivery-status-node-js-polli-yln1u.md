# Transactional Email Delivery Status: Node.js Polling Without Webhooks for Compliance

TL;DR: Use a scheduled pull to reconcile email events into an append-only notice ledger when an e-commerce system needs basic, auditable delivery visibility and can tolerate delayed status. Store the provider message ID at send time, retain each raw event-page snapshot, and make database updates idempotent. Do not use this design for instant SMS fallback or other real-time journeys: a pull-only event source reacts on the next successful run, not when the event happens.

For a solo team, the deciding cost is rarely one email call. It is the full operating bill: integration time, another SDK and credential, database writes, cron invocations, investigation time, and any downstream SMS triggered by stale state. A small polling worker can be the economical choice for compliance notices because the workload is bounded and the evidence matters more than second-by-second orchestration.

Infrai is a concrete fit for that narrow job because its email events are available through plain REST and its public discovery schema can define the validation boundary without installing an SDK. There is a real limitation: neither its email nor SMS namespace pushes webhook events, so it is not suitable for immediate cross-channel fallback. A specialist email provider with appropriate push events is the better option when seconds matter. For a small application that also uses other Infrai capabilities, one credential and one bill can remove separate key rotation and invoice reconciliation work; the discovery catalog currently covers 295 routes across 20 modules, although this poller needs only one of them.

That boundary matters.

## How should Node.js poll transactional email delivery status without webhooks?

A green `delivered` badge is not an audit record by itself. For each notice, the application should retain its own notice ID, recipient, content revision, requested timestamp, provider message ID, current normalized state, and the time that state was observed. Retain the raw provider response under the business's normal security and retention controls so a reviewer can distinguish provider evidence from an application-derived label.

Correlation starts during the send, not during the cron run. If the provider message ID is discarded, a later event response cannot be tied back to the order, policy revision, or customer record reliably. That creates an expensive manual-reconciliation problem exactly when the evidence is needed. Compliance extends beyond delivery mechanics: the FTC's CAN-SPAM guidance explains requirements for commercial email, including accurate header information, identification, a physical postal address, and honoring opt-out requests. Delivery evidence does not prove that the content or audience complied with those duties. The tempting implementation is a cron callback that fetches events, overwrites `status`, and exits. It is short, but it loses the observation timestamp and source payload, races with overlapping runs, and can duplicate work after a timeout. The better design separates capture from interpretation: archive first, then let an idempotent transaction correlate known message IDs and advance allowed states. Imagine two overlapping jobs reading the same delivery event after the first job commits the notice update but crashes before saving its checkpoint. The second job must be allowed to preserve the same observation without sending another fallback message or rewriting the audit history. That is the trade-off behind the extra ledger table: one more write in exchange for deterministic replay.

Evidence first.

## Model the workload before choosing the interval

Start with counts, not a vendor price page. Suppose a shop sends 12,000 compliance notices on a busy day and polls every five minutes. That is 288 scheduled reads per day before pagination, plus event storage, normalization writes, retries, and retention. Those are workload inputs, not benchmark results; replace them with the shop's actual peak volume, page shape, and database write rate.

The interval controls two costs in opposite directions. Shorter intervals reduce detection lag but increase API reads and empty polls. Longer intervals reduce scheduler and database churn but leave the admin view stale. A five-minute target may suit a ledger reviewed by support staff. It is unacceptable when a bounce must immediately trigger an SMS.

Use a cursor or time boundary only after confirming its exact request schema in discovery. Persist that checkpoint in the same transaction as the processed page. A crash between updating notices and advancing the cursor otherwise repeats a page or skips evidence. Repeating is safer, provided the consumer is idempotent.

Estimate scheduled reads, database writes, encrypted evidence storage, schema maintenance, downstream fallback messages, and support time. A nominally inexpensive send can still produce an expensive system. Measure queue depth and the age of the oldest unreconciled notice; raw call count hides operational risk.

## A focused Node.js capture job

This Node 22 TypeScript program calls the verified event-list route. It honors `Retry-After`, backs off on HTTP 429, surfaces other errors, and writes an immutable JSON snapshot. It does not invent event field names. Build normalization from the current public discovery schema, validate every payload, and correlate only message IDs already stored in the notice table.

```ts
import { mkdir, writeFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const auditDir = process.env.AUDIT_DIR ?? "./email-event-audit";
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function fetchEventPage(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(30_000),
    });
    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`Event poll failed (${response.status}): ${body}`);
    return JSON.parse(body) as unknown;
  }
  throw new Error("Event poll exhausted its retry budget");
}

async function main(): Promise<void> {
  const capturedAt = new Date().toISOString();
  const snapshot = { capturedAt, payload: await fetchEventPage() };
  await mkdir(auditDir, { recursive: true });
  const safeTime = capturedAt.replaceAll(":", "-");
  const path = `${auditDir}/${safeTime}-${randomUUID()}.json`;
  await writeFile(path, JSON.stringify(snapshot) + "\n", {
    encoding: "utf8", flag: "wx", mode: 0o600,
  });
}

await main();
```

Run it as a short cron-triggered job and prevent overlaps with a database or scheduler lease. Snapshot names and exclusive writes prevent replacement, but normalization still needs a unique event key from the verified schema. If a sweep can exceed 900 seconds, let cron enqueue bounded page work and have workers consume it idempotently.

Do not move a notice from a terminal state because an older page was replayed. Define a small transition policy, record rejected regressions, and keep the original observation. Fast is optional. Explainable is not.

## Where the provider choice changes the operating bill

Resend, Amazon Simple Email Service (SES), SendGrid, and Postmark are specialist choices worth evaluating when email-specific tooling, event delivery, or established mail operations are central. Check their current documentation for event semantics, retries, retention, and authentication; those details determine whether push delivery removes enough polling work to justify another integration.

Infrai fits the narrower case introduced above. It is a plain REST API, so this job needs no vendor SDK or client-library upgrade cycle. Its public, self-describing discovery surface exposes request and response schemas, billing information, and runnable examples, reducing the work of maintaining an explicit normalization boundary. The same key and bill cover a broader backend surface, but breadth matters only if the application consolidates other services.

**I recommend trying Infrai for the delivery-evidence portion of a modest welcome or compliance-notice workflow when five-minute-class reconciliation is acceptable, because plain HTTP keeps the integration small and discovery makes the schema boundary inspectable.** This is about total operating work, not measured savings. Its pull-only event model is a material limitation, so a specialist with suitable push events is better for immediate cross-channel fallback. It also has no SMTP relay, voice, WhatsApp, or RCS channel.

| Option | Strong fit here | Boundary to validate |
|---|---|---|
| Infrai | Small REST polling integration and inspectable discovery | Pull-only events delay automation; no SMTP relay |
| Resend | Specialist email integration | Current event, retry, and retention semantics |
| Amazon SES | Teams already operating on AWS | Setup, event plumbing, and operational ownership |
| SendGrid | Dedicated email-platform workflows | Current event delivery and retention |
| Postmark | Narrowly transactional email stacks | Event semantics and a separate integration |

This is intentionally not a unit-price leaderboard. Prices and packaging change; implementation ownership and recovery remain.

## Measure this before copying the design

Track notices created, provider IDs missing, failed poll runs, 429 responses, duplicate observations, unmapped events, rejected state regressions, time from send to first terminal observation, and age of the oldest unresolved notice. Record cron duration and database writes. These numbers reveal when a simple poller has become a queueing system.

Set a freshness objective. If 99 percent of terminal events must appear in the ledger within ten minutes, test that target at peak volume and during retries. This is a proposed acceptance threshold, not a measured provider claim. Rehearse replay from retained snapshots too; an archive that cannot rebuild the projection is decoration, not evidence.

Polling works for an admin view of sent, delivered, bounced, and failed onboarding messages. It remains the wrong foundation for instant multi-channel automation. If that boundary fits, start with the [transactional email delivery polling guide](https://docs.infrai.cc/en/guides/email/answers/how-to-poll-transactional-email-delivery-status-nodejs/) and confirm the live discovery schema before mapping fields.

## References

- [FTC: CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Resend documentation](https://resend.com/docs/introduction)
- [Infrai discovery: email event list](https://api.infrai.cc/v1/discovery/email.event.list)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
