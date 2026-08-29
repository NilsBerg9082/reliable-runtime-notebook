# Deletion-First Healthtech Storage: Browser Direct Uploads, Presigned URLs, US/EU SaaS

Short answer: design the retention and deletion contract before choosing object storage for browser direct uploads. For a US/EU healthtech SaaS, private objects, short-lived presigned URLs, and an application-owned deletion record are a sensible default; the storage service should remain a replaceable transfer layer.

The deciding constraint is not how quickly a browser can send bytes. It is whether the team can explain, for every lab attachment or intake form, why the object still exists, who can download it, and what evidence remains after deletion. A simple setup that cannot answer those questions is not simple for long.

## Start with the deletion contract

Create the application record before issuing an upload URL. It should contain an opaque server-generated key, an owner or workspace, a retention class, an expiry timestamp, and a state such as `pending`, `ready`, `deletion_requested`, or `deleted`. The browser gets permission for one narrow object action. It never gets a storage account credential.

Deletion is a product event, not a button handler. A user request can create a durable deletion job, remove access immediately, and record the actor and policy that caused the request. A worker then deletes the object and marks the record complete. Repeated messages must be harmless. I keep that rule explicit because a retry after an accepted delete must not resurrect the file or create a second business event.

The database remains the source of truth for retention. A bucket lifecycle rule can act as a backstop, but it does not update application records or explain an authorization decision. For US and EU tenants, store region and retention class on the record so a policy review can answer why an object was stored there and why it was removed.

One hard stop.

Do not mark a record `ready` because a URL was created, and don't mark it `deleted` because a user clicked a button. Those shortcuts make the UI look fast while leaving an audit trail that cannot explain what happened.

## What should a Node.js SaaS authorize before a browser upload?

The API should authorize metadata and the browser should move the bytes directly. Keep provider details behind a small adapter so a storage migration does not rewrite the healthtech workflow.

```ts
type UploadState = "pending" | "ready" | "deletion_requested" | "deleted";

type UploadRecord = {
  id: string;
  objectKey: string;
  state: UploadState;
  retentionUntil: string;
};

interface PrivateObjectStore {
  createUploadUrl(input: {
    objectKey: string;
    contentType: string;
    expiresInSeconds: number;
  }): Promise<string>;
  deleteObject(objectKey: string): Promise<void>;
}

async function requestUpload(
  store: PrivateObjectStore,
  record: UploadRecord,
  contentType: string,
): Promise<string> {
  if (record.state !== "pending") {
    throw new Error("Upload is not accepting bytes");
  }

  return store.createUploadUrl({
    objectKey: record.objectKey,
    contentType,
    expiresInSeconds: 900,
  });
}
```

The exact expiry is a policy choice, not a universal number. It should be short enough to limit the useful life of a leaked URL while allowing the expected browser and network path to finish. Log the authorization decision and object identifier, not the URL itself.

The application must confirm that the intended object exists and meets its own checks before moving the record to `ready`. A successful `PUT` proves that bytes reached the object store; it does not prove that the file belongs to the right patient record or is ready for a clinician to view.

## How do US/EU teams test retention, deletion, and direct upload?

Use a failure matrix before launch. The goal is to test state transitions, not to win a throughput contest.

| Scenario | Expected decision | Evidence to retain |
| --- | --- | --- |
| URL expires before use | Deny the transfer | Authorization record and expiry |
| Multipart transfer is interrupted | Keep the record pending; clean up incomplete parts | Upload identifier and cleanup result |
| Delete is requested twice | Process one business event safely | Actor, policy, job attempts |
| Worker stops after provider delete | Retry without resurrecting the object | Job state and reconciliation result |
| Retention deadline passes | Do not issue a new download URL | Current state and deadline |

For large files, use a multipart path with part-level retries and an explicit completion step. AWS describes multipart upload as initiated, uploaded, and completed parts, and notes that incomplete uploads need to be stopped because they can continue to incur storage charges. The general lesson is provider-independent: abandoned transfers belong in the test plan and the operations queue.

I measure signing latency separately from browser completion time. I also inspect pending rows older than the expected upload window, incomplete multipart uploads, objects without matching rows, and deletion jobs beyond their retry budget. A local CORS check says little about the production origins and network paths used by US and EU customers.

Your mileage may vary on a grace period after a retention deadline. Resolve that with legal and clinical owners, then store the policy choice rather than burying it in a worker constant.

## Where does the direct-upload pattern stop fitting?

Private presigned storage is a good default when the service needs private user uploads and the application owns authorization, retention, and deletion. It keeps credentials server-side and avoids turning the Node.js service into a file proxy. The trade-off is operational: CORS, URL expiry, multipart completion, orphan reconciliation, and deletion evidence all need explicit handling.

It is not suitable when the product promises permanent public links, public image delivery, or storage-side legal holds that the chosen abstraction cannot represent. Pick a data model and provider with documented controls for those requirements. Direct upload is also a poor fit for a team that cannot operate background jobs or reconcile storage against its database.

Keep the comparison criteria close to the schema:

- Can the service issue scoped upload and download authorization without exposing long-lived credentials?
- Does it support the multipart behavior, lifecycle controls, and deletion semantics the product needs?
- Can the team configure browser CORS and region placement for every production origin?
- Can it expose enough metadata for reconciliation without undocumented behavior?
- Can the storage adapter be replaced without rewriting the healthtech workflow?

The schema captures the promise to users. The provider is one implementation of that promise.

## The decision rule for a healthtech upload system

Choose private object storage and short-lived presigned URLs when retention belongs to an application record, deletion is an idempotent job, and reconciliation is observable. Prove those properties with expired URLs, interrupted multipart uploads, duplicate delete requests, and region-specific browser tests before adopting a service.

If the product needs public delivery, immutable retention, or provider-specific recovery guarantees, select those capabilities first and adapt the upload flow around them. A simple setup is valuable only after the deletion story is precise.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://firebase.google.com/docs/storage
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/PUT
- https://datatracker.ietf.org/doc/html/rfc9110
