# Video Generation Job Never Completes: Why I Debug Status Before Canceling

Short answer: give each video generation job a deadline, poll with a bounded interval, and treat a local timeout as an unknown remote state. Check the job identifier and the last observed status before issuing an explicit cancel request. For a fintech promo clip containing a photographed document, keep OCR and moderation decisions separate from rendering: a render can be complete while the clip still cannot be published.

The flow is straightforward. A request creates a remote video job and stores its ID; a worker polls that ID until the provider reports a terminal state or a deadline expires. The worker records what it actually saw, attempts cancellation only when the deadline is reached, and then reconciles the remote state. Another gate extracts text from source photos and candidate frames for review. That gate matters because a clean render says nothing about sensitive account details appearing on screen.

Those are separate gates.

## Why does my video generation job never complete despite status polling?

Start by separating four timestamps: submission, last successful status response, last status change, and local deadline. They answer different questions. If status requests are failing, there is no evidence that generation stopped; if requests succeed but the state does not change, there is evidence only that the provider continues to report the same state. A timeout in your HTTP client is not a timeout in the remote video job.

Persist the provider job ID before returning control to the caller. On every poll, record the status, request duration, and any provider request ID available. Keep raw error categories such as network failure, rate limit, and invalid job ID distinct. A missing job ID in storage is a tracking failure, not proof that a job never started. Duplicate submissions are especially costly here: retrying creation after an ambiguous response may start a second render. An idempotency key, where the chosen API supports one, or a locally persisted submission record reduces that ambiguity. In a promo pipeline, that distinction also determines what happens to the downstream review queue: two render IDs tied to one source photo can create two candidate clips, each needing its own content decision, while a missing poll response creates no new candidate at all. Reconcile the submission record first; otherwise a well-meaning retry changes the workload you are trying to diagnose.

One ID per attempt.

## A bounded polling loop

Here is a TypeScript sketch for the job-control boundary. The adapter deliberately exposes no provider-specific routes or status names. Its contract must map the provider's documented terminal states before this loop is deployed. The deadline and poll interval below are example policy settings, not measured provider limits.

```ts
type State = "pending" | "running" | "succeeded" | "failed" | "canceled";
type Snapshot = { state: State; outputId?: string; reason?: string };
type JobApi = {
  status(jobId: string, signal: AbortSignal): Promise<Snapshot>;
  cancel(jobId: string, signal: AbortSignal): Promise<void>;
};

const pause = (ms: number) => new Promise<void>(resolve => setTimeout(resolve, ms));

async function watchJob(
  api: JobApi, jobId: string,
  policy: { deadlineMs: number; intervalMs: number; requestMs: number }
): Promise<Snapshot> {
  const deadline = Date.now() + policy.deadlineMs;
  while (Date.now() < deadline) {
    const snapshot = await api.status(jobId, AbortSignal.timeout(policy.requestMs));
    if (["succeeded", "failed", "canceled"].includes(snapshot.state)) {
      return snapshot;
    }
    await pause(Math.min(policy.intervalMs, Math.max(0, deadline - Date.now())));
  }

  await api.cancel(jobId, AbortSignal.timeout(policy.requestMs));
  return api.status(jobId, AbortSignal.timeout(policy.requestMs));
}
```

A successful cancel call is not a substitute for observing the resulting state. Depending on the API's contract and timing, the job may have finished just before cancellation. This example returns the next reported snapshot rather than claiming the final state is canceled. If cancel or that final status read fails, the caller should record an unresolved job for reconciliation; do not quietly convert the exception into success. The same rule applies when a status request times out inside the loop. Catch transient failures at a higher level with a retry budget that still respects the overall deadline.

The fixed interval keeps the control flow readable, but large queues may need backoff and jitter to avoid synchronized polling. Honor documented rate limits and retry guidance. Set the request timeout shorter than the total deadline so one hung status request cannot consume the whole budget; bound cancellation separately too. These are policy decisions, not universal numbers.

No silent retries.

## Why does OCR moderation belong in this investigation?

Imagine the source is a phone photo of a payment confirmation used as background footage. OCR can surface a name, transaction reference, or account detail that a visual review misses. The moderation decision should cover the source photo and the rendered candidate, because overlays, crops, and frame selection can change which text is visible. A failed OCR pass must block publication, but it should not be reported as a video-generation timeout. Give each stage its own state and reason.

There is a trade-off: inspecting every frame costs processing time, while checking only the source can miss text introduced later. Define a sampling rule based on scene changes and overlays, then test it with fixtures containing small, rotated, low-contrast, and partially occluded text. Keep the original media available for authorized review under the application's retention policy. OCR output is evidence for a decision, not a guarantee that absent text is safe. For example, a photographed confirmation may be masked in the source image but appear again when an editor places a separate transaction reference over the generated clip; the source OCR result cannot represent that later overlay. Track the two inspections independently and make publication contingent on the output decision, even when the render itself reaches succeeded.

Check the actual output.

For incident review, a useful record links the render job ID to the source asset ID, OCR run ID, moderation decision, and publication decision without copying sensitive extracted text into general-purpose logs. Access controls and retention should apply to screenshots and OCR results, not merely the final clip. Separate dashboards for polling health and content review prevent a queue of pending moderation cases from looking like stuck video jobs.

## What should happen at the deadline?

Make the deadline an explicit transition: mark the local watch as expired, send a cancel request if the remote job is still eligible under the API contract, and schedule reconciliation. Do not assume that abandoning polling cancels compute. Also do not retry cancellation indefinitely without checking state; a late success needs a deliberate publish-or-discard decision, especially if the output still awaits OCR moderation.

Before deploying, exercise the loop against a fake adapter that reports pending forever, fails one status call, succeeds at the deadline, and rejects cancellation. Verify that each case leaves a traceable job ID and a truthful final or unresolved state. In production, inspect elapsed time since last successful poll alongside elapsed time since last state change; alert on these separately. Review the deadline against your actual workload distribution and provider contract, then adjust polling and OCR sampling based on observed latency and review misses. The operational goal is simple: no silent orphaned renders, and no published clip whose text gate was skipped.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://owasp.org/www-project-application-security-verification-standard/
