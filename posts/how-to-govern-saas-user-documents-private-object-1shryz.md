# How to Govern SaaS User Documents: Private Object Storage Retention

Short answer: for SaaS user document storage, keep every marketplace file private, send its bytes directly from the browser to object storage with a narrowly scoped signed upload, and make the application database own retention, holds, and deletion evidence. A low storage rate cannot rescue a design that loses track of deletion.

The useful split is between a credential that moves bytes and a record that explains why those bytes still exist. A seller uploads a product video without routing a multi-gigabyte body through the application. The API issues the signed operation, the browser transfers the object, and the API later confirms it. From then on, a deletion worker follows the marketplace's policy record rather than guessing from an object key or bucket age.

Keep those jobs separate.

## How should SaaS object storage protect private user document links?

Start with one invariant: no signed link is an authorization system. The application authorizes the seller or buyer first, selects exactly one opaque object key, and only then asks the storage adapter for a short-lived operation. AWS describes presigned URLs as bearer tokens, which is the right mental model even when the implementation is hidden behind a generic adapter. Whoever receives the URL can use the permission it carries while that permission remains valid, so don't put it in analytics events, durable logs, support transcripts, or HTML that a third party can inspect.

S3, Cloudflare R2, DigitalOcean Spaces, and Backblaze B2 appear in many comparison searches, but those names do not answer the exposure question. The behavior to verify is narrower: the browser receives permission for one operation on one opaque key for a bounded time, while the application retains the authorization decision.

The code should return the same external `file not found` result for a missing file and a tenant mismatch. That avoids turning sequential IDs or timing differences into a catalog of another marketplace's media. Internally, log the reason using the file ID and request ID, but never the signed URL.

## Rehearse upload, hold, and deletion races in one state machine

The following TypeScript is deliberately provider-free. It can run under a TypeScript test runner once a repository and storage adapter are supplied, and its interfaces form the acceptance contract for any candidate. The important part isn't the class names; it is the order: reserve metadata, sign one key, confirm storage, authorize every read, claim deletion, remove the bytes, then record completion.

```ts
type FileState = "reserved" | "available" | "deleting" | "deleted";

type MediaFile = {
  id: string;
  marketplaceId: string;
  objectKey: string;
  state: FileState;
  deleteAfter: Date;
  held: boolean;
};

interface ObjectStore {
  signUpload(input: {
    key: string;
    contentType: string;
    expiresInSeconds: number;
  }): Promise<string>;
  signDownload(input: { key: string; expiresInSeconds: number }): Promise<string>;
  exists(key: string): Promise<boolean>;
  remove(key: string): Promise<void>;
}

interface MediaRepository {
  reserve(file: MediaFile): Promise<void>;
  get(id: string): Promise<MediaFile | null>;
  markAvailable(id: string): Promise<void>;
  findDeletionCandidates(now: Date, limit: number): Promise<MediaFile[]>;
  claimDeletion(id: string): Promise<boolean>;
  markDeleted(id: string, deletedAt: Date): Promise<void>;
}

export async function beginDirectUpload(
  store: ObjectStore,
  repo: MediaRepository,
  input: {
    fileId: string;
    marketplaceId: string;
    contentType: string;
    deleteAfter: Date;
  },
) {
  const objectKey = `marketplaces/${input.marketplaceId}/media/${input.fileId}`;
  await repo.reserve({
    id: input.fileId,
    marketplaceId: input.marketplaceId,
    objectKey,
    state: "reserved",
    deleteAfter: input.deleteAfter,
    held: false,
  });

  const uploadUrl = await store.signUpload({
    key: objectKey,
    contentType: input.contentType,
    expiresInSeconds: 300,
  });
  return { fileId: input.fileId, uploadUrl };
}

export async function confirmDirectUpload(
  store: ObjectStore,
  repo: MediaRepository,
  fileId: string,
) {
  const file = await repo.get(fileId);
  if (!file || file.state !== "reserved") throw new Error("invalid upload state");
  if (!(await store.exists(file.objectKey))) throw new Error("upload not present");
  await repo.markAvailable(file.id);
}

export async function createDownload(
  store: ObjectStore,
  repo: MediaRepository,
  input: { fileId: string; marketplaceId: string },
) {
  const file = await repo.get(input.fileId);
  if (!file || file.marketplaceId !== input.marketplaceId) {
    throw new Error("file not found");
  }
  if (file.state !== "available") throw new Error("file unavailable");
  return store.signDownload({ key: file.objectKey, expiresInSeconds: 60 });
}

export async function deleteDueMedia(
  store: ObjectStore,
  repo: MediaRepository,
  now = new Date(),
) {
  const files = await repo.findDeletionCandidates(now, 100);
  for (const file of files) {
    if (file.held || file.deleteAfter > now) continue;
    if (!(await repo.claimDeletion(file.id))) continue;
    await store.remove(file.objectKey);
    await repo.markDeleted(file.id, now);
  }
}
```

The five-minute upload and one-minute download windows are application choices in this example, not universal recommendations. Tune them to the largest expected media object, client connection quality, and the damage caused by a leaked link. A large upload may need a multipart protocol or refreshed authorization; that belongs behind `ObjectStore`, while the retention record stays unchanged.

For a marketplace, the metadata row needs more than `ownerId` and `key`. A catalog video can be deleted when its listing closes, a dispute attachment can be retained while the case is active, and a seller export may have a deadline set by a separate policy. Store the policy class, planned deletion time, hold state, upload state, and final deletion evidence. The bucket holds media. The row holds intent.

There is one subtle race worth dwelling on. Suppose a listing closes and its video becomes due at 02:00, while a buyer opens a dispute at 01:59:59. The dispute transaction must set the hold before the worker can claim deletion, or the repository must serialize those two state changes. Checking `held` in memory and deleting later is not enough by itself, because the value can change between the check and the claim. Define `claimDeletion` as one conditional database update that succeeds only when the file is due, available, and still unheld. Then define hold creation to reject a file already claimed for deletion and send that case to an explicit product rule. I don't pretend storage can resolve that conflict. It is marketplace state, and the database transaction is where the decision belongs.

A green upload demo proves very little. Build a conformance suite around the adapter and run it in an isolated private bucket. The suite should upload a known byte sequence, confirm it, obtain a signed download, compare the bytes, and delete the object. It should also assert that a second marketplace cannot obtain a link through the application, an unconfirmed upload cannot be downloaded, a held object survives a deletion pass, and two workers cannot both claim the same row.

A green upload demo proves very little. Build a conformance suite around the adapter and run it in an isolated private bucket. The suite should upload a known byte sequence, confirm it, obtain a signed download, compare the bytes, and delete the object. It should also assert that a second marketplace cannot obtain a link through the application, an unconfirmed upload cannot be downloaded, a held object survives a deletion pass, and two workers cannot both claim the same row.

Then test time.

Signed URL expiry is especially easy to test badly. Don't assume every provider reports an expired credential with the same status body; assert that access is denied after the documented window, and record the status plus request identifier for diagnosis. I'm not sure a synthetic test can establish the exact behavior of every browser, intermediary, and clock-skew combination. A controlled integration test resolves the provider side, while a browser test from the production origin resolves the CORS side. MDN explains that cross-origin browser requests can trigger a preflight and that the server's CORS response controls which origins, methods, and headers are allowed. Configure the narrow set the upload UI actually sends, then verify it rather than copying a broad policy from a tutorial.

The test output should make the lifecycle extractable instead of burying it in logs:

| Event | Application evidence | Storage evidence |
| --- | --- | --- |
| Upload reserved | tenant, key, policy, deadline | none yet |
| Upload confirmed | state becomes `available` | object exists at the exact key |
| Hold applied | hold transaction and reason | object remains private |
| Deletion claimed | conditional state transition | object still exists before removal |
| Deletion completed | timestamp and operation ID | object is absent |

## Prove absence, then calculate operating cost

Deletion needs two pieces of evidence: storage no longer exposes the object, and the application has a durable completion event tied to the policy decision. A successful delete response alone doesn't prove that the right tenant key was selected. Have the worker record the immutable file ID, opaque key, policy class, deletion timestamp, and operation request identifier when the adapter exposes one. A reconciliation job should revisit rows stuck in `deleting`, check the object, and complete or retry the same idempotent transition. It should also sample deleted rows and verify absence. This is not glamorous work -- it is the difference between a retention setting and an operable retention system.

Private object storage with signed links is a good fit when the application can decide access before each link is issued and large media can move directly between browser and store. **The catch is that this pattern is not suitable when every read requires synchronous byte-level transformation, content inspection, or application-generated output.** In those cases, keep an application proxy or a dedicated media pipeline in the data path even though it costs more compute and operational attention.

The storage boundary has limits too. Keep it only when documented signing, CORS, deletion, and retention behavior passes the conformance suite and the billing model matches the measured operation mix. Change the boundary when retention controls cannot express the required holds or the delivery model cannot meet geographic needs. Compatibility labels don't erase semantic differences, and no neutral article can pick a winner without the retention classes, request distribution, object sizes, and download locations.

Only after the absence checks pass does “cheapest” become meaningful. Capture monthly bytes stored, median and large-object size, upload count, download count, metadata checks, multipart operations, deletion volume, and download geography. Replay that mix and apply the current public billing rules. Your mileage may vary sharply with marketplace media size and buyer location; a single storage-per-gigabyte figure omits request classes and data transfer.

Before release, trace one seller video from reservation through browser upload, confirmation, signed buyer access, listing closure, hold evaluation, deletion claim, object removal, audit event, and reconciliation. Run the same trace with a dispute arriving at the deletion boundary. If both paths are explainable from application records and repeatable in tests, the architecture is ready for a pricing comparison. If they aren't, keep working on the boundary.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
