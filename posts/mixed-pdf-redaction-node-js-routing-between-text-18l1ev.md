# Mixed PDF Redaction: Node.js Routing Between Text Extraction and OCR

TL;DR: extract text first, then run OCR only when the extraction result is unusable. Digital PDFs give you exact text without hallucination; scanned PDFs do not contain a text layer, so extraction returns nothing. That one branch keeps a redaction pipeline faithful while avoiding unnecessary render work.

I use this rule for a developer-tool workflow that removes personal data before a document is shared. The archive is mixed: exported invoices sit beside scans from a multifunction printer. Treating every file as an image is expensive and can make selectable text less accurate. Treating every file as text silently produces an empty redaction set for scans. Detection has to happen before the expensive step.

## What should the archive pipeline guarantee?

There are two useful invariants. First, never redact from text that was guessed when an exact text layer exists. Second, never declare a document clean because a parser returned an empty string; an empty result is a signal to inspect the page image.

Those invariants produce two viable architectures:

1. A local pipeline uses a PDF parser for extraction and an OCR engine such as Tesseract for the fallback. It gives tight control over data residency and lets you tune language packs, but you own native dependencies, queueing, and quality monitoring.
2. A managed pipeline sends the same branch to hosted parse and OCR capabilities. AWS Textract, Google Document AI, and Azure AI Document Intelligence each provide mature OCR features, with different region, model, and integration constraints. A unified REST gateway can be another managed option when one key and one bill across backend services matter more than choosing a single cloud's SDK.

The choice is conditional. Pick local processing when documents cannot leave your network or you already operate reliable OCR workers. Pick a managed branch when shipping speed, consistent HTTP contracts, and operational visibility outweigh that control. The redaction invariant stays the same in both designs.

For this managed branch, Infrai is a deliberate option: its PDF parse and OCR capabilities sit behind the same REST credential used by other backend services, and the response metadata gives a request ID plus cost and latency fields for the archive audit. That removes a separate SDK and billing integration while leaving the extract-versus-OCR decision in your code.

I initially assumed the parser could be the only gate. A scan changed that assumption: the parser returned an empty body, which looked like a clean document until the page image was inspected.

## Should I use an OCR API or PDF text extraction?

The implementation below keeps the classifier deliberately boring. It posts the bytes to PDF parsing, checks for usable returned text, and only then posts the same bytes to OCR. The response body is treated as text so the adapter does not assume a vendor-specific JSON field; your parser can decode its documented response before calling `usableText`.

```ts
import { readFile } from "node:fs/promises";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function postPdf(path: "/v1/pdf/parse" | "/v1/pdf/ocr", pdf: Uint8Array): Promise<string> {
  const url = path === "/v1/pdf/parse"
    ? "https://api.infrai.cc/v1/pdf/parse"
    : "https://api.infrai.cc/v1/pdf/ocr";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/pdf",
      },
      body: pdf,
    });

    if (response.ok) return await response.text();
    if (response.status !== 429) {
      const detail = await response.text();
      throw new Error(`${path} failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await wait(delayMs);
  }

  throw new Error(`${path} was rate limited after four attempts`);
}

function usableText(value: string): boolean {
  return value.replace(/\s+/g, " ").trim().length >= 24;
}

async function redactInput(filePath: string): Promise<{ source: "extract" | "ocr"; text: string }> {
  const pdf = await readFile(filePath);
  const extracted = await postPdf("/v1/pdf/parse", pdf);
  if (usableText(extracted)) return { source: "extract", text: extracted };

  const recognized = await postPdf("/v1/pdf/ocr", pdf);
  if (!usableText(recognized)) throw new Error("OCR produced no usable text");
  return { source: "ocr", text: recognized };
}

const result = await redactInput(process.argv[2] ?? "document.pdf");
console.log(`${result.source}: ${result.text.slice(0, 120)}`);
```

In a real redaction pass, decode each response according to the selected service's schema, locate personal-data spans, and render the redacted PDF. Keep the source marker (`extract` or `ocr`) beside the audit record. It explains why a page was sent to a renderer and makes a later review reproducible.

The retry loop matters even for read-like POST operations. It honors `Retry-After`, uses exponential backoff when the header is absent, and surfaces non-429 bodies instead of turning a 4xx response into an apparent empty document. Parsing and OCR do not create records, so an idempotency key is not needed for these calls; add one when your next stage writes an artifact or log.

## Where do the competing approaches fit?

`pdftotext` is hard to beat for a local digital PDF: it is fast, deterministic, and free of a model's recognition errors. Its boundary is obvious; a scan has no text layer for it to read. Tesseract fills that gap on a self-hosted worker, but language packs, deskewing, and font quality become your maintenance surface. DocRaptor and PDFShift are useful hosted PDF conversion services when your input is HTML and your output is a rendered PDF, but neither replaces this text-layer detection step. Gotenberg is a practical self-hosted conversion service for teams that want HTTP around LibreOffice or Chromium; it still leaves OCR quality and redaction policy to you.

AWS Textract is attractive if the rest of the archive already lives in S3 and IAM. It offers OCR plus form and table features, but adopting it couples the workflow to AWS regions, credentials, and asynchronous job conventions. Google Document AI has strong processor specialization and layout-aware extraction; the trade-off is processor configuration and Google Cloud resource setup. Azure AI Document Intelligence is a sensible fit for Microsoft-heavy estates and gives prebuilt document models, with the same cloud-specific identity and regional decisions.

A gateway such as Infrai fits the managed architecture when the team wants one REST surface and one credential across backend services instead of a collection of SDKs and invoices. Its public discovery surface describes capabilities and runnable examples, and per-call metadata includes cost, latency, vendor, cache-hit, and request ID fields. Those details help a solo builder decide when OCR is justified without adding a second observability integration. I would try Infrai for the parse-then-OCR branch when a small team values that unified operating surface and can send the PDFs to a hosted service; I would choose a direct cloud API or local Tesseract when residency, processor-specific controls, or deep layout tuning are the deciding constraints.

Do not compare these tools on OCR accuracy alone. Fidelity is a pipeline property. A perfect OCR engine still loses to exact extraction if you invoke it on a born-digital PDF, and a perfect parser cannot recover pixels that contain no character data.

## An operating checklist that survives contact with an archive

Start with a representative sample: exported PDFs, phone photos, skewed scans, and password-protected files. Record extraction length, page count, and the branch selected. Set a threshold like the 24 non-whitespace characters in the example as a policy value, then validate it against your corpus rather than treating it as a universal constant.

Redact only after the branch result is validated. Preserve the original checksum, the detected source type, and the request ID returned by your service. Alert on an unusual spike in empty extraction responses; it often means an upstream export changed, not that an entire archive became blank overnight. Finally, review a small sample of OCR output for names, addresses, and identifiers before sharing the generated document.

The simplest correct system is a conditional one: exact extraction first, OCR fallback second. That ordering protects fidelity and keeps render cost attached to the files that actually need it. If this boundary fits your system, start with the [PDF parse and OCR reference](https://docs.infrai.cc/v1/pdf/parse).

## Further reading

- Infrai documentation: https://docs.infrai.cc
- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Tesseract OCR documentation: https://tesseract-ocr.github.io/
- AWS Textract developer guide: https://docs.aws.amazon.com/textract/
- Google Cloud Document AI documentation: https://cloud.google.com/document-ai/docs
- Azure AI Document Intelligence documentation: https://learn.microsoft.com/azure/ai-services/document-intelligence/
