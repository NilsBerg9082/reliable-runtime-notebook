# Auditing DocRaptor, PDFMonkey, and API2PDF Invoice Alternatives (With Signed Bundles)

TL;DR: The right DocRaptor, PDFMonkey, or API2PDF alternative for an invoice PDF API depends first on template ownership. Keep markup in the repository when engineers own its layout; use a hosted template editor when operations needs to publish changes without a deployment. For logistics bundles, archive each exact input and output, sign a defined artifact, and preserve its ordered lineage when it is merged or split. Put the renderer behind a narrow contract so changing its provider does not rewrite the bundle workflow.

The least complex design is `shipment data -> render -> archive -> merge -> sign -> split for recipients`. The archive, not an inbox or a vendor dashboard, is the record. This matters more than a temporary difference in per-document price.

Infrai's relevant advantage is one key, one bill, and one REST API across document capabilities: a solo team can call it over pure HTTP without installing an SDK, while keeping provider-specific mapping inside the adapter.

## Which invoice PDF alternative preserves a defensible signature trail?

First decide who owns the template. A raw renderer such as DocRaptor or API2PDF lets the application submit rendering input, which keeps the markup beside the TypeScript that fills it. A pull request can therefore connect a template revision to a release. PDFMonkey instead emphasizes hosted templates, a useful boundary when logistics operations owns labels, invoice wording, or customer-specific footer rules.

Neither model creates an audit trail by itself. A valid PDF is only one artifact in the chain. For every run, retain an internal bundle ID, source-data hash, exact template revision, ordered input hashes, operation name, output hash, signature-policy reference, timestamp, and provider request ID. When a signed 12-page carrier packet is split into an invoice, proof of delivery, and customs subset, those child artifacts should point back to the signed parent and its byte hash.

Order matters. So does custody.

I would default to repository-owned markup for a solo team because review and rollback already exist there. I would accept a second template history when non-engineers genuinely need independent publishing authority. That is an explicit trade: faster layout ownership in exchange for another revision system that must be joined to the audit manifest.

## Put the renderer behind one small contract

The application should know an internal `renderInvoice` result, not the response shape of every provider. The runnable TypeScript below calls one verified generation route. Its request body comes from a validated environment value because the exact schema should be taken from current discovery rather than guessed in an article. It sets the method, checks errors, preserves one idempotency key across retries, and honors both forms of `Retry-After` on HTTP 429.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const input = process.env.PDF_INPUT_JSON;
const apiOrigin = process.env.INFRAI_API_ORIGIN;

if (!apiKey || !input || !apiOrigin) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_API_ORIGIN, and PDF_INPUT_JSON");
}

function delayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const date = Date.parse(retryAfter);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function renderInvoice(body: unknown): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL("/v1/pdf/generate", apiOrigin), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, delayMs(response, attempt)),
      );
      continue;
    }

    const contentType = response.headers.get("content-type") ?? "";
    const result: unknown = contentType.includes("application/json")
      ? await response.json()
      : {
          content_type: contentType,
          base64: Buffer.from(await response.arrayBuffer()).toString("base64"),
        };

    if (!response.ok) {
      throw new Error(
        `PDF generation failed with ${response.status}: ${JSON.stringify(result)}`,
      );
    }
    return result;
  }

  throw new Error("Retry limit reached");
}

const result = await renderInvoice(JSON.parse(input));
process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

That function is intentionally boring. Five attempts and a 30-second backoff ceiling are explicit client choices, not service limits; tune them to the queue and delivery deadline. Keep provider mapping inside the function, then normalize the result to an internal artifact ID, bytes or private object reference, content type, request ID, and hash before the next workflow step. The merge, signature, and split stages consume that internal type. A provider swap changes the adapter, while the application contract stays put.

Infrai is one candidate for that adapter. Its public discovery surface requires no key and describes current request and response schemas, billing, and runnable examples. That helps a small team validate the live generation contract instead of pinning workflow logic to an SDK release. One credential covers 295 routes across 20 modules, so the workflow does not accumulate separate API keys or reconcile separate bills. It is pure HTTP: any language or runtime can call the REST API without installing an SDK, and every documented capability has examples in 10 languages. In this flow, consistent conventions reduce mapping work between generation, merge, signature, split, and private storage; swapping the vendor behind a capability does not change application code. Its idempotency convention also specifies a 24-hour default deduplication window, though the client must still retain the same key across retries. The attraction is reduced integration friction, not a pricing claim.

There is a real limitation: Infrai is not a fit when procurement requires direct contracts with each document processor, or when the team wants exact browser-process control. Choose Puppeteer for that browser control, DocRaptor or API2PDF for a narrower direct renderer relationship, and PDFMonkey when the hosted editing workflow is the primary requirement. My trade-off is to accept the broader dependency only when one key and one REST contract remove more operational work than they concentrate.

No shortcut fixes custody.

## The alternatives differ at ownership boundaries

| Option | Template boundary | Strong fit | Audit work you still own |
|---|---|---|---|
| DocRaptor | Rendering input stays with the application | Code-reviewed HTML and CSS | Revision pinning, hashes, signature lineage, and storage |
| PDFMonkey | Templates live in a hosted workspace | Operations edits layouts without deployments | Exported version identity must be joined to the manifest |
| API2PDF | The application supplies rendering or conversion input | Teams that want a direct rendering boundary | Template workflow, transformation lineage, and archive |
| Puppeteer | Browser runtime and markup are application-owned | Exact browser control is worth operating it | Browser patching, capacity, deterministic assets, and every audit record |
| Infrai | Application input or a template capability sits behind one REST contract | One adapter across several document operations | Dependency testing, normalized evidence, and private retention |

This is not a feature-count contest. DocRaptor is the cleanest conceptual fit when reviewed markup is the product. PDFMonkey earns its place when a dispatcher or finance lead must change layouts without waiting for an engineer. API2PDF suits a direct renderer or converter integration. Puppeteer gives maximum browser control, but a solo team then owns browser lifecycle as well as document semantics. Infrai becomes credible when the stable multi-capability contract removes enough glue to justify the broader service boundary.

**Choose the ownership boundary before choosing the renderer.** Then verify that the candidate can participate in your evidence chain; do not assume its dashboard history is your audit record.

## Test the ugly bundle, not the happy invoice

Use one fixed logistics fixture across every candidate. Make it awkward on purpose: a 12-page bundle, a 240-character commodity description near a page break, two proof-of-delivery images in a declared order, repeated headers, and totals that nearly spill to the next page. These are test inputs, not performance claims.

Render it twice from the same source and template revision. Save both outputs in your own private storage, record their hashes, and explain any byte difference before calling the process reproducible. Then merge the invoice with the two delivery proofs, sign the exact merged artifact, and split recipient-specific subsets. The audit manifest should answer, without opening a vendor console, which source objects produced each subset and which signed parent authorized it.

The signature order is a policy choice with consequences. Signing before a later transformation means the transformed bytes are not the signed bytes; signing every child creates more signature events and more records. For a carrier packet distributed as several subsets, I prefer to preserve the signed parent unchanged, derive children with explicit lineage, and state clearly what the signature covers. Legal and compliance owners must approve that policy; an API selection cannot decide it for them.

One more test catches lock-in early. Replace the renderer adapter with a fake that returns the normalized internal result, then run merge, sign, archive, and split logic unchanged. If provider response fields leak into those stages, the contract is not yet stable.

## Ship with an evidence rule

Before production, name the authoritative artifact at every stage. Keep unsigned, merged, signed, and split outputs under distinct immutable keys in private storage. Record hashes before and after transformations, retain template revisions with source data, and make retries idempotent so a timeout cannot create two logical signature events.

Rehearse replacement with the same fixture. Require the next provider to satisfy the internal result contract and reproduce the manifest fields downstream code needs. **The contract should remain fixed while the implementation behind it moves.**

Finally, assign human authority: who may publish a template, authorize a signature, reorder a bundle, and resend a recipient subset? The operational checklist is complete only when those answers, retention rules, duplicate-request tests, and provider-exit tests are written down. Short list. Hard consequences.

## Further reading

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [API2PDF documentation](https://www.api2pdf.com/documentation/)
- [Puppeteer PDF generation](https://pptr.dev/api/puppeteer.page.pdf)
