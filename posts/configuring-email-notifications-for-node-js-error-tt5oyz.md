# Configuring Email Notifications for Node.js Errors in Healthtech Observability

Short answer: configure Node.js error notifications through a separate worker that evaluates durable import events, then sends deduplicated email without coupling delivery to ingestion. The deciding constraint is cost attribution: an alert must identify the work that consumed time and retries.

Keep it separate.

Send email from that notification worker after a scheduled healthtech import has produced an explicit result event. The importer should not depend on email delivery.

That means treating “completed with zero results” as a policy decision rather than assuming every zero is an error. Give each run a stable import ID, record the expected delivery window, and carry bounded dimensions such as source, tenant, environment, and job class. Then let a worker evaluate the event and deliver one deduplicated notification.

The simple approach is a process-exit hook that emails on a nonzero exit code. It misses silent empty imports, repeats messages during retries, and cannot explain which feed or tenant owns the investigation. The chosen design costs a queue or outbox and a small evaluator, but it keeps ingestion, policy, transport, and accounting separate. That is the trade.

## What should count as a failed scheduled import?

An importer can exit normally after receiving an empty page, rejecting every record, or failing to advance an upstream cursor. None of those states is adequately described by `exitCode === 0`.

That is the trap.

Record one structured result event per run. Useful fields include `importId`, `source`, `tenant`, `startedAt`, `finishedAt`, `status`, `resultCount`, `expectedBy`, `retryCount`, and `jobClass`. A zero-result run becomes alertable only when it is late relative to a feed-specific expectation. A 15-minute grace period can be a starting policy for a feed with a short schedule; it is not a universal healthtech default.

Consider a feed scheduled for 06:00. If its policy says results are expected by 06:15, a completed event at 06:10 with zero rows is still pending, while the same event at 06:16 is a candidate alert. If the source normally delivers every other day, the policy should say so rather than forcing the evaluator to infer it from a stream of false alarms. The event should preserve the schedule, the observed result count, the retry count, and the reason the evaluator selected the state. That extra context is cheap to retain and prevents an operator from reconstructing the decision from scattered logs after the email arrives.

The evaluator should distinguish at least four states: a normal result, an expected quiet window, a late result, and an import failure. That distinction matters for cost. A failed import that retries three times has a different owner and investigation burden from a feed that is legitimately quiet, even if both produce zero rows.

Use a metric name that describes the domain and unit, such as `healthtech_import_results_total`, with stable labels for low-cardinality dimensions. Prometheus recommends names that communicate meaning and labels that represent dimensions rather than multiplying metric names. Patient IDs, exception text, and arbitrary URLs belong in a protected event store, not metric labels or an email subject.

## How should a Node.js service separate email from error tracking?

The import job owns facts. The evaluator owns policy. The mail adapter owns transport and retry behavior.

That boundary lets an SMTP relay or an HTTPS mail service sit behind the same application interface. It also prevents an outage in the mail path from changing the import result. SMTP is specified by RFC 5321; the implementation detail of the transport should remain outside the code that decides whether a result is overdue.

```ts
type ImportResult = {
  importId: string;
  source: string;
  tenant: string;
  finishedAt: string;
  resultCount: number;
  expectedBy: string;
  status: "completed" | "failed";
  retryCount: number;
  jobClass: string;
};

type Alert = {
  key: string;
  subject: string;
  text: string;
};

interface Mailer {
  send(alert: Alert): Promise<void>;
}

function evaluateResult(result: ImportResult, now = Date.now()): Alert | null {
  const overdue = Date.parse(result.expectedBy) <= now;
  const noResult = result.status === "completed" && result.resultCount === 0;

  if (!overdue || !noResult) return null;

  return {
    key: `import-no-result:${result.source}:${result.tenant}`,
    subject: `Scheduled import produced no results: ${result.source}`,
    text: [
      `Tenant: ${result.tenant}`,
      `Import ID: ${result.importId}`,
      `Finished: ${result.finishedAt}`,
      `Retry count: ${result.retryCount}`,
      `Job class: ${result.jobClass}`,
      "Result count: 0",
      "Action: inspect the import event and upstream delivery window.",
    ].join("\n"),
  };
}
```

The example deliberately keeps cost dimensions in the event rather than inventing a dollar estimate. `jobClass`, retry count, execution duration, and an LLM token estimate, when an LLM step exists, can support a later report. An email should answer “what needs attention?” It should not pretend to be the accounting system.

The idempotency key is coarser than the import ID. It groups repeated evaluations for the same source and tenant, so a queue retry does not send one message per attempt. Store notification state with an expiry or a recovery transition. A key that never expires suppresses future incidents; a key that is never stored turns one upstream problem into a burst of mail.

There is a real ordering trade-off. Marking a key before sending avoids duplicates but can lose an alert when the transport rejects it. Marking it after sending permits a duplicate if the worker crashes between the send and the database write. An outbox record with delivery status, retry count, and last error makes that uncertainty inspectable. It does not make delivery exactly once; it gives the team a recoverable record of what happened.

## What should the alert measure besides failure?

Measure the alert path separately from the import path. Track result count, import duration, event age, notification attempts, notification failures, and time from the expected window to acknowledgement. Keep dimensions bounded: source, tenant tier, environment, and pipeline are usually more useful than unrestricted identifiers.

Cost attribution belongs at the same event boundary. For each run, retain a job class, retry count, execution duration, and any token estimate produced by an LLM step. Aggregate these fields into a daily operational report. The alert can stay short while the report explains which scheduled imports consumed work without producing a result.

This is where a tempting optimization fails. A global “email on error” hook is cheap to add, but it hides silent failures and makes the cost of investigation impossible to allocate. A feed-level expectation plus a durable event costs more storage and one policy decision per source. That is a reasonable trade when an unattended import can hide missing clinical data.

The correction is straightforward: alert on a violated expectation, not on a noisy implementation symptom.

Before copying the threshold, measure the normal quiet-feed rate, duplicate-alert rate, and acknowledgement delay for two weeks. If quiet feeds are common, change the expectation window or suppress by source policy. If duplicates are common, fix idempotency before adding another notification channel. If acknowledgement is slow, add event context before increasing email volume.

## How should the implementation be tested and operated?

Run the evaluator in dry-run mode first, recording candidate alerts without sending them. Test a normal result, an expected zero-result window, a late result, a failed import, a queue retry, and a mail timeout. Verify that an import can complete while the mailer is unavailable and that the worker retries without replaying the same incident forever.

Keep recipient policy outside the importer. A source owner, on-call group, or tenant contact should be changeable without redeploying ingestion. Redact sensitive fields from the subject and body, link to an access-controlled event view, and preserve the import ID for tracing across logs, metrics, and the event store. OWASP's logging guidance is useful here because an operationally useful record still needs deliberate treatment of sensitive data.

One sentence is enough for the stopping rule.

Ship result events, bounded metrics, a policy evaluator, and an idempotent outbox. Add dashboards or escalation after those signals are trustworthy. Email is only the delivery mechanism; the durable result event is what makes a silent import observable and its operating cost attributable.

## References

- Prometheus, “Metric and label naming”: https://prometheus.io/docs/practices/naming/
- Node.js documentation, “Environment variables”: https://nodejs.org/api/environment_variables.html
- RFC 5321, “Simple Mail Transfer Protocol”: https://www.rfc-editor.org/rfc/rfc5321
- OWASP, “Logging Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
