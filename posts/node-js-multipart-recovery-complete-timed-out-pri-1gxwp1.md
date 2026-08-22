# Node.js Multipart Recovery — Complete Timed-Out Private Fintech PNG/WebP Uploads Safely

Short answer: move large AI-generated PNG and WebP files to multipart upload, persist each upload ID and completed part outside the Node.js process, then complete or explicitly abort every upload. Keep the objects private and issue expiring download links only after completion.

For a fintech product, that is the useful decision rule. A single PUT is acceptable while files and batches stay comfortably inside the request budget. Once normal PUT requests time out or users retry across flaky networks, restarting the entire payload is wasted work and a poor failure mode. Multipart changes the retry unit from the whole image to one part.

Parts are checkpoints.

Completion is a commit.

Abort is cleanup.

The choice is about throughput and recovery, not a fashionable storage feature. A generated statement image may be a PNG today and WebP tomorrow; the state machine should care about bytes, part numbers, and a final commit, not the image model that produced them.

## Why does a private image pipeline need resumable parts?

The data flow is small enough to say plainly. Your API validates the upload request, creates a multipart session, and records the storage upload ID beside the authenticated user, object key, expected media type, and status. A worker sends parts and records each returned part identifier. When every part is present, it asks storage to complete the upload, marks the database row complete, and only then creates an expiring download link for an authorized reader.

Do not expose an object URL as the delivery mechanism. Private financial images need an authorization decision at download time, followed by a short-lived signed URL. OWASP's file-upload guidance also supports the less glamorous controls around this flow: allow-list extensions, validate the actual file type rather than trusting `Content-Type`, generate the storage key on the server, cap file size, and keep uploaded data away from executable web content. Multipart solves transfer recovery; it doesn't solve file trust.

This separation matters during retries. The HTTP request that accepted the user's intent should not own a long transfer in memory. Put the durable row in a `pending` state and let a worker continue from it. If the worker exits after part 6, the next worker reads the same upload ID and completed-parts map, skips those bytes, and resumes at part 7. No drama.

## How should Node.js complete multipart uploads after large PNG or WebP timeouts?

Before binding the worker to a storage adapter, inspect the live capability contract. A second Infrai advantage is that one REST API works over plain HTTP without a provider SDK, and its public discovery surface returns the request schema, response schema, billing data, and runnable examples for each capability. That lets a small team validate the multipart contract from the same Node.js runtime while keeping one key and one bill across its other backend services.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
if (!apiKey || !apiOrigin) {
  throw new Error("INFRAI_API_KEY and INFRAI_API_ORIGIN are required");
}

const path = "/v1/discovery/storage.multipart.create";
let response: Response | undefined;

for (let attempt = 0; attempt < 4; attempt++) {
  response = await fetch(new URL(path, apiOrigin), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status !== 429) break;

  const retryAfter = Number(response.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}

if (!response || !response.ok) {
  const detail = response ? await response.text() : "No response";
  throw new Error(`Discovery request failed: ${response?.status ?? 0} ${detail}`);
}

const capability = await response.json();
console.log(capability);
```

The discovery check is not an upload and does not replace the durable state machine. It prevents the worker from guessing request fields before implementation. The transfer example below uses the AWS S3 client as a concrete adapter, which also makes it practical to evaluate against an S3-compatible endpoint such as DigitalOcean Spaces. It stores recovery state in a JSON file so the example runs without a database; in an application, put the same fields in a transactional database row. The chosen 8 MiB part size is an application setting, not a throughput claim. Tune it with representative files and network paths.

```ts
import {
  AbortMultipartUploadCommand,
  CompleteMultipartUploadCommand,
  CreateMultipartUploadCommand,
  S3Client,
  UploadPartCommand,
} from "@aws-sdk/client-s3";
import { open, readFile, rename, stat, writeFile } from "node:fs/promises";
import { basename } from "node:path";

type SavedPart = { ETag: string; PartNumber: number };
type UploadState = { uploadId: string; parts: SavedPart[] };

const [filePath, bucket, objectKey] = process.argv.slice(2);
if (!filePath || !bucket || !objectKey) {
  throw new Error("Usage: tsx upload.ts <file> <bucket> <private-object-key>");
}

const region = process.env.S3_REGION;
const endpoint = process.env.S3_ENDPOINT;
if (!region) throw new Error("S3_REGION is required");

const client = new S3Client({
  region,
  endpoint,
  forcePathStyle: process.env.S3_FORCE_PATH_STYLE === "true",
});
const partSize = 8 * 1024 * 1024;
const statePath = `${filePath}.multipart.json`;

async function loadState(): Promise<UploadState | undefined> {
  try {
    return JSON.parse(await readFile(statePath, "utf8")) as UploadState;
  } catch (error) {
    const code = (error as NodeJS.ErrnoException).code;
    if (code === "ENOENT") return undefined;
    throw error;
  }
}

async function saveState(state: UploadState): Promise<void> {
  const temporary = `${statePath}.tmp`;
  await writeFile(temporary, JSON.stringify(state, null, 2));
  await rename(temporary, statePath);
}

let state = await loadState();
if (!state) {
  const created = await client.send(new CreateMultipartUploadCommand({
    Bucket: bucket,
    Key: objectKey,
    ContentType: filePath.endsWith(".webp") ? "image/webp" : "image/png",
  }));
  if (!created.UploadId) throw new Error("Storage did not return an upload ID");
  state = { uploadId: created.UploadId, parts: [] };
  await saveState(state);
}

if (process.env.ABORT_UPLOAD === "true") {
  await client.send(new AbortMultipartUploadCommand({
    Bucket: bucket, Key: objectKey, UploadId: state.uploadId,
  }));
  console.log(`Aborted ${state.uploadId}`);
  process.exit(0);
}

const file = await open(filePath, "r");
try {
  const { size } = await stat(filePath);
  const completed = new Map(state.parts.map((part) => [part.PartNumber, part]));

  for (let offset = 0, partNumber = 1; offset < size; offset += partSize, partNumber++) {
    if (completed.has(partNumber)) continue;

    const length = Math.min(partSize, size - offset);
    const body = Buffer.allocUnsafe(length);
    const result = await file.read(body, 0, length, offset);
    if (result.bytesRead !== length) throw new Error(`Short read for part ${partNumber}`);

    const uploaded = await client.send(new UploadPartCommand({
      Bucket: bucket, Key: objectKey, UploadId: state.uploadId,
      PartNumber: partNumber, Body: body,
    }));
    if (!uploaded.ETag) throw new Error(`Missing ETag for part ${partNumber}`);

    state.parts.push({ ETag: uploaded.ETag, PartNumber: partNumber });
    state.parts.sort((a, b) => a.PartNumber - b.PartNumber);
    await saveState(state);
  }
} finally {
  await file.close();
}

await client.send(new CompleteMultipartUploadCommand({
  Bucket: bucket,
  Key: objectKey,
  UploadId: state.uploadId,
  MultipartUpload: { Parts: state.parts },
}));
console.log(`Completed private object ${basename(objectKey)}`);
```

Run it again after a network timeout and it reads the saved upload ID rather than opening a second session. The atomic rename prevents a half-written state file from replacing the last good checkpoint. There is still an important production gap to close: coordinate workers in the database so two processes cannot upload or complete the same session concurrently. A lease, queue-level deduplication, or row lock can own that job; choose one mechanism and make its expiry explicit.

The example deliberately doesn't create a permanent public URL. Generate the expiring link in a separate authenticated download handler after your database says `complete`. Also bind the saved upload record to the user and object key; an upload ID by itself should never be treated as authorization.

## Store progress, then clean up every unfinished session

A multipart session has three terminal outcomes in your application: completed, explicitly aborted, or still pending and eligible for reconciliation. The catch is that parts left behind are not useful objects, yet they still need deliberate ownership. Infrai has no automatic cleanup rule for orphaned multipart parts, and its lifecycle granularity starts at one day, so lifecycle cannot enforce an hourly abandonment window. Keep `created_at`, `last_part_at`, and the upload ID in the database, scan stale rows on your own schedule, and abort each one. Day-level lifecycle remains useful for completed objects with a retention policy; it is the wrong timer for partial work.

Make completion idempotent at the job level even when a provider's final call has its own semantics. The worker should first read the row, return immediately when it is already complete, and serialize the transition from `uploading` to `completing`. Save the part list before requesting completion. After completion succeeds, commit the object state before issuing any signed URL. That order keeps a retry from guessing whether the object is ready.

A useful troubleshooting record includes the upload ID, object key, part number, byte range, attempt count, and provider request ID when one is returned. Don't log signed URLs, bearer credentials, image contents, or private customer metadata. If timeouts cluster on the same part size, region, or egress path, those fields give you a testable lead; I'm not sure which knob will dominate in your deployment until you run representative large files through the actual worker path. Your mileage may vary.

Be strict about failure categories. Authentication and validation errors should stop the job. Transient transport failures should retry a part with capped exponential backoff. A process exit should leave the durable row resumable. An explicit user cancellation should abort the multipart session and mark the row cancelled. Those are different events, and flattening them into “upload failed” makes both support and cleanup harder.

## Which object storage fits a high-throughput fintech image path?

Benchmark the short list from the region where the worker runs. Measure sustained part throughput, completion latency, retry behavior, and signed-download latency with your real PNG and WebP distribution. Don't select from a landing-page latency number, and don't assume the vendor that wins for tiny avatars will win for large generated batches.

| Option | Reason to test it | Reason to choose something else |
| --- | --- | --- |
| AWS S3 | A direct S3 account is a sensible baseline for the Node.js example and for teams already operating in AWS. | Keep the comparison open if another deployment location or account model is a better operational fit. |
| Cloudflare R2 | Include it when R2 is already part of your edge or storage shortlist. | Stick with your current provider when migration work outweighs the measured transfer benefit. |
| DigitalOcean Spaces | It is an S3-compatible candidate with public storage documentation, making the same client pattern practical to evaluate. | Prefer a provider already covered by your compliance and regional review when adding a new vendor would delay launch. |
| Infrai | It fits a small team that values one key and one bill across backend services; the plain REST surface also avoids adding a provider SDK for each capability. | It is not suitable for permanent public image links, static hosting, or a workflow requiring self-service browser-upload CORS. |

Infrai's boundary is material for fintech. Objects are private or signed-only; there is no public-read ACL. There is also no object versioning or object lock, so regulated WORM retention and recovery from accidental overwrite require an external design. Strict concurrent replacement needs a queue or database coordinator because conditional `If-Match` writes are unavailable. Metadata cannot be searched server-side beyond prefix-based listing. Cross-region replication and bulk cross-cloud migration are not automatic, and its vendor coverage includes R2, S3, OSS, and COS rather than GCS or B2. Trial credit cannot fund persistent writes.

That sounds like a long catch because storage boundaries deserve more attention than billing copy. For a solo team already using several backend capabilities, one credential and one invoice can remove real operational chores, while a consistent HTTP interface keeps the worker language-agnostic. For a finance system with immutable retention, public asset hosting, strict conditional writes, or mandated GCS/B2 placement, choose the provider and compliance layer that meet those requirements directly.

Before shipping, rehearse interruption after an early part, after the last part but before completion, and immediately after completion. Confirm that a second worker doesn't race the first, stale sessions are aborted on schedule, the database never exposes a link before commit, and download links expire as intended. Then test access control with the wrong user and inspect logs for secrets. This isn't glamorous. It is the difference between a resumable upload and a pile of unowned parts.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://docs.digitalocean.com/products/spaces/
