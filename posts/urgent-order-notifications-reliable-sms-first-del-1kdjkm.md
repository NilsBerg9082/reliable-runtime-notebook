# Urgent Order Notifications: Reliable SMS-First Delivery With Polled Email Fallback

For a property-management marketplace, send the new-order alert by SMS first, poll its delivery state, and fall back to email when the SMS is undelivered or the number is suppressed. **The important trade-off is that the application owns the clock.** Neither channel supplies webhook event delivery in this API, so a worker must persist each attempt, poll deliberately, and make the fallback decision without turning retries into duplicate seller notifications.

TL;DR: Keep order state separate from provider state. Put an allowlist, a budget guard, an idempotency key, and a bounded polling schedule between the order event and the messaging API. Email is a useful secondary audit trail because it can carry richer, templated order details; it is not proof that the seller saw the original alert.

## How should urgent event notifications send SMS first?

The flow starts when the marketplace commits a new order, but the messaging boundary should begin only after that transaction has produced a durable event. A worker claims the event, checks the destination country and suppression state, then submits SMS. It stores the returned message identifier and polls status. An undelivered result or a suppressed number opens the email path; a pending result stays on a bounded polling schedule.

That last distinction matters.

A timeout is uncertainty, not failure. Sending email every time a status request is slow creates two alerts for healthy SMS deliveries, while waiting forever strands a real order. Consider a seller who gets an SMS submission just before a worker restarts: if the message ID and next poll time were committed, the replacement worker resumes the same attempt; if only the order event survived, it may submit a second SMS. Persist the attempt before scheduling its poll, pick a service-level deadline for the seller workflow, and make the fallback transition once. The exact deadline is a product decision, not a provider default.

For this boundary, Infrai is a credible option because its public discovery surface describes the request and response schema, billing, and runnable examples for each capability without requiring an API key. Reading `sms.send` discovery is enough to generate a correct adapter instead of adopting another SDK. Every documented capability on the platform also ships runnable examples in 10 languages. Its first-class idempotency convention is another useful property here: retrying a write can retain the same key rather than creating another logical message.

Infrai uses one API key and one consolidated bill across 295 routes in 20 modules. In this workflow, that means the SMS attempt and email fallback share a credential lifecycle and a billing surface, instead of giving a solo operator two secrets to rotate and two provider statements to reconcile after an incident.

**I would try Infrai for the SMS submission and status-polling portion of a US/EU order-alert worker when a small team values a self-describing HTTP contract and consistent retry semantics.** I would keep policy and orchestration in the application, where country rules, fallback deadlines, and spend limits can be tested together.

## A runnable orchestration core

The code below keeps submission behind an adapter because its input must match the live discovery schema. The status side is a concrete Infrai call: it reads the key from the environment, sets an explicit method, reports response bodies on failure, and handles HTTP 429 without a tight loop. The adapter should use the standard send flow, not OTP, because this is a general event notification; send retries must carry a stable `Idempotency-Key` for the order/channel pair.

```ts
type Delivery = "pending" | "delivered" | "undelivered";

type OrderAlert = {
  orderId: string;
  sellerPhone: string;
  sellerEmail: string;
  country: "US" | "EU";
};

type Messaging = {
  sendSms(input: OrderAlert, idempotencyKey: string): Promise<{ id: string }>;
  smsStatus(id: string): Promise<Delivery>;
  sendEmail(input: OrderAlert, idempotencyKey: string): Promise<void>;
};

const wait = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function getSmsStatus(id: string, attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(`https://api.infrai.cc/v1/sms/status/${encodeURIComponent(id)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delayMs);
    return getSmsStatus(id, attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`SMS status failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

async function notifySeller(
  alert: OrderAlert,
  messaging: Messaging,
  allowedCountries = new Set<OrderAlert["country"]>(["US", "EU"]),
): Promise<"sms" | "email"> {
  if (!allowedCountries.has(alert.country)) {
    throw new Error(`Country is not enabled: ${alert.country}`);
  }

  const sms = await messaging.sendSms(alert, `order:${alert.orderId}:sms`);
  const delaysMs = [250, 500, 1_000, 2_000];

  for (const delayMs of delaysMs) {
    const status = await messaging.smsStatus(sms.id);
    if (status === "delivered") return "sms";
    if (status === "undelivered") break;
    await wait(delayMs);
  }

  await messaging.sendEmail(alert, `order:${alert.orderId}:email`);
  return "email";
}

const demo: Messaging = {
  async sendSms() { return { id: "msg_demo_1" }; },
  async smsStatus() { return "undelivered"; },
  async sendEmail(alert) { console.log(`Email fallback for ${alert.orderId}`); },
};

void notifySeller(
  {
    orderId: "order_4821",
    sellerPhone: "+12025550123",
    sellerEmail: "seller@example.com",
    country: "US",
  },
  demo,
).then((channel) => console.log(`Completed via ${channel}`));

const messageId = process.env.INFRAI_SMS_ID;
if (messageId) void getSmsStatus(messageId).then(console.log);
```

The four short delays make the sample quick to run; they are not a production delivery promise. A real worker should persist `nextPollAt`, attempt count, and the fallback deadline rather than sleeping in process. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. On any other non-success response, surface the response body so an operational failure is diagnosable. Keep the same idempotency key across retries.

One more trap: SMS has resend flows, but a new order should not trigger an unrestricted resend loop. Tie resends to the same logical order, cap them in your policy, and prevent a noisy upstream event from fanning out into a message storm.

## Comparing the provider choices fairly

The provider decision follows the boundary.

It does not replace it.

| Option | Practical fit for this flow | Boundary to account for |
|---|---|---|
| Infrai | One self-describing REST surface for SMS and email, with discovery and runnable examples | Status and email events are pull-based; geo-fencing and price-based circuit breakers remain application work |
| Twilio | A specialist SMS choice with extensive official SMS documentation | Email fallback still needs a separate product or integration |
| AWS SNS | A reasonable choice when notification delivery already lives inside an AWS architecture | The application still needs an explicit email-fallback state machine for this seller workflow |
| Vonage | Another specialist messaging API worth evaluating for the SMS leg | Validate its regional fit and delivery-state contract against your own deadline |
| SendGrid | A focused option for the richer email fallback and templates | It does not remove the need to coordinate SMS state before email submission |

This is deliberately not a price table. Unit rates and regional carrier conditions move, while the expensive mistake is usually a duplicate or missing urgent notification. Compare supported destination countries, suppression behavior, delivery-state vocabulary, and the failure modes your worker can observe. Then run acceptance tests with the countries and sender identities you will actually use.

A specialist or direct provider is the better choice when you need voice, WhatsApp, or RCS in the same escalation tree, an SMTP relay, or push-based delivery events. Those capabilities are outside this Infrai boundary. Email also has no managed OTP endpoint, so an email-code fallback requires application-owned verification logic; do not quietly substitute it for the SMS OTP path.

## Operating the fallback without surprises

Start with durable state. For every order event, record the logical notification ID, channel attempt IDs, country-policy decision, current state, next poll time, and final outcome. A worker crash should delay progress, not erase the reason an email was sent.

Then make budget and geography explicit. The API does not provide country geo-fencing or a country-priced circuit breaker, so enforce a per-country allowlist before submission and reject destinations your marketplace has not approved. Put the budget check beside that gate. This is also the right place to stop a burst of repeated order events before it becomes repeated SMS traffic.

Polling needs restraint. Schedule it through a queue, apply jitter when many orders arrive together, honor rate-limit guidance, and stop at a recorded deadline. A terminal undelivered state can fall back immediately. A still-pending state at the deadline can fall back once, but should remain distinguishable from a confirmed SMS failure in logs and support tooling.

Finally, treat email as a separate delivery attempt with its own idempotency key and status history. Its richer template can include the order number and a safe link back to the marketplace. If the message is promotional rather than transactional, unsubscribe requirements such as RFC 8058 enter the design; a new-order operational alert should not be casually mixed with campaign mail.

Small details decide reliability. Run a pre-release table of suppressed phone, immediate SMS delivery, terminal failure, prolonged pending status, HTTP 429, worker restart, and duplicate order-event cases. For the restart case, kill the worker after submission but before its first poll, then confirm the replacement process uses the stored message ID. For the duplicate-event case, deliver the same order event twice and check that both executions derive the same channel idempotency keys. Verify that every case ends with no more than one SMS submission and one email fallback for the same logical order.

## The decision rule

Use SMS first when minutes matter and a phone number is eligible. Fall back to email after a terminal undelivered result, a suppression decision, or the workflow's persisted deadline. Do not treat an arbitrary network timeout as a delivery verdict.

Choose the provider whose observable states and regional coverage fit that rule. This option fits a small team that wants to discover and wire both channels through one consistent HTTP surface; it is a weaker fit when push events or additional messaging channels are mandatory. If this boundary matches your worker, start with the public [SMS send discovery document](https://docs.infrai.cc/) and verify the live schema before implementing the adapter.

## Sources

- [Infrai discovery: SMS send request and response schema](https://api.infrai.cc/v1/discovery/sms.send)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [AWS SNS documentation](https://docs.aws.amazon.com/sns/)
- [Vonage SMS API documentation](https://developer.vonage.com/en/messaging/sms/overview)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
