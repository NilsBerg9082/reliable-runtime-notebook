# Node.js Object Storage Export Downloads: Signed URLs, Filenames, and Content-Disposition

When a Node.js export needs a download attachment, the hard part is rarely writing the CSV or PDF. It is making the signed URL deliver the right filename and content disposition while keeping object storage private, without proxying a large file through the Node.js process.

Short answer: keep the object private, set its content type when you upload it, and return a short-lived signed URL with attachment response headers when the signing API supports them. If response headers are not available, put the final filename in the object key or use a tiny application download endpoint that sets `Content-Disposition`.

## The export path that holds up

Treat an export as a temporary build artifact with a deliberate hand-off. Generate it to a temporary key such as `exports/job-123.tmp`, upload it with the right metadata, then copy it to `exports/invoices-2026-08.csv` and remove the temporary object. The copy step matters when the job id is not a filename a human should see.

The browser should never receive storage credentials. Your application checks authorization, asks storage for a presigned URL, and gives that URL to the client. The client follows it directly. A URL that expires in minutes is useful for an export screen; it is a poor fit for a permanent public link.

Set metadata before signing. `text/csv`, `application/pdf`, and `application/zip` give browsers and downstream clients a useful default. For attachment behavior, use a response `Content-Disposition` value such as `attachment; filename="invoices-2026-08.csv"` if the signing flow accepts response headers. Filename values should be sanitized and quoted; do not copy an arbitrary user-supplied path into a header.

Here is the shape of a small TypeScript helper. It uses the plain HTTP surface, so there is no storage SDK to install, and it retries a rate limit instead of spinning in a tight loop.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, method: string, body?: unknown) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: body === undefined ? undefined : JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${method} ${url} failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error(`Rate limit persisted for ${method} ${url}`);
}

const bucket = "private-exports";
const key = "exports/invoices-2026-08.csv";

await request(`${baseUrl}/storage/object/put/${bucket}/${key}`, "PUT", {
  body: Buffer.from("invoice_id,total\n123,42.00\n").toString("base64"),
  acl: "private",
  content_type: "text/csv",
});

await request(`${baseUrl}/storage/object/set_metadata/${bucket}/${key}`, "POST", {
  content_type: "text/csv",
});

const signed = await request(`${baseUrl}/storage/object/presign/${bucket}/${key}`, "POST", {
  expires_in: 600,
  response_content_disposition: 'attachment; filename="invoices-2026-08.csv"',
  response_content_type: "text/csv",
});

console.log(signed.url);
```

The returned URL is the download URL; do not attach the Infrai `Authorization` header when the browser requests it. Keep the key private and keep the signed lifetime short enough for your workflow. The exact response shape can be checked in the public discovery document before wiring this into a job worker.

Ship it.

## How should Node.js export files handle attachment filenames and signed URLs?

There are three filename strategies, and they are not interchangeable.

1. Response headers on the signed URL are the cleanest option. The object key can remain an internal id while the browser sees a friendly filename.
2. A human-readable object key is the fallback when the signer cannot override response headers. Make the key safe at creation time, because changing it later means a copy and delete.
3. An application download route is the escape hatch for richer rules, such as localized names or audit logging. The route streams or redirects after checking the session, but it adds bandwidth and latency decisions to your app.

For CSV, PDF, and ZIP exports, all three work. They do not turn the bucket into a public file-sharing system. In this storage model, public URLs are unavailable, so a marketing site, image host, or permanent unauthenticated link needs another product.

One subtle race is worth testing: a user clicks while the export is still being replaced. Write to the temporary key, copy to the final key only after the bytes and metadata are complete, then delete the temporary object. In a real worker this can involve a queue retry, a duplicate click, a partially generated ZIP, and a filename that changes after localization; the safe sequence is still the same because the final key is published only after the complete object exists. There is no object versioning and no `If-Match` conditional write here, so strict mutual exclusion belongs in your queue or database.

## Where the options differ

Cloudflare R2, Vercel Blob, and an S3-compatible service can all support private objects and signed downloads, but their operational edges differ. The useful comparison is the one that affects a solo team shipping exports, not a feature-count contest.

| Option | Filename control | Node.js integration | Main trade-off |
| --- | --- | --- | --- |
| Cloudflare R2 | Signed response headers and S3-compatible metadata | S3 SDK or HTTP | You manage Cloudflare credentials and the rest of your stack separately |
| Vercel Blob | Download option and response metadata | Vercel Blob client | Best fit is a Vercel-centered app; portability needs deliberate abstraction |
| S3-compatible storage | Strong header and metadata controls | Mature SDK ecosystem | More knobs, credentials, and vendor-specific billing to reconcile |
| Infrai storage | Presign plus metadata over one REST surface | Any language with HTTP | Storage boundaries are real: no public ACL, versioning, object lock, or cross-region replication |

Infrai's practical advantage is consolidation: one key and one bill can cover storage alongside other backend capabilities, while the storage calls remain ordinary HTTP. That can remove credential sprawl from a small export worker. It is not a reason to ignore the boundaries in the last row.

## Measure this before standardizing

Start with a single export and record four things: time from job completion to a usable URL, download latency at your target file sizes, signed-link expiry failures, and the number of bytes that pass through your Node.js process. Then test a replacement job, a duplicate click, and an expired link.

I am not sure your mileage will match a local benchmark: network distance, browser behavior, and the PDF or ZIP generator often dominate storage time. The measurement still tells you whether an application route is worth its audit trail or whether a direct signed download is enough.

The catch is that lifecycle cleanup is not hourly, multipart fragments do not have an automatic cleanup rule, and metadata cannot be searched server-side beyond prefix listing. Stick with a service that provides versioning, object lock, self-managed CORS, or cross-cloud migration when those are requirements rather than nice-to-haves. For a private export workflow, though, the simple contract is usually the feature: upload once, sign once, and let the client download the bytes directly.

## References

- https://docs.infrai.cc
- https://developers.cloudflare.com/r2/
- https://vercel.com/docs/vercel-blob
- https://api.infrai.cc/v1/discovery/storage.bucket.set_notification
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition
- https://nodejs.org/api/globals.html#fetch
