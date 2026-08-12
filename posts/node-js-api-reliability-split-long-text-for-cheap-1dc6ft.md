# Node.js API Reliability: Split Long Text for Cheap SaaS Summarization

Short answer: treat a sales-call summary as a budgeted workflow, not a single cheap API call. Store the source, split it by meaning, count tokens before dispatch, and turn the result into validated CRM actions only after a separate extraction step. This keeps a Node.js SaaS feature portable across providers and makes an unexpectedly long transcript a product decision instead of a surprise invoice.

The first useful question is not “which API has the lowest advertised rate?” It is “what happens when the transcript is three times longer than the one in the demo?” A call summary that loses the renewal date is a quality failure. A summary that invents a next step is a CRM data failure. A request that crosses a context limit is an admission-control failure. Different failures need different counters, owners, and recovery paths.

Keep the pipeline explicit. It doesn't need to be clever.

## Where does a sales-call pipeline lose reliability?

For sales calls, the input should have a stable envelope before it reaches a model: call ID, account ID, participants, transcript text, language, and a source timestamp. Keep that envelope outside the text sent for summarization where possible. It gives retries and audits an identity, while the model sees the content it actually has to summarize. The envelope is also the join key for a later CRM write.

Before choosing an API, classify the failure you need to contain:

| Failure | What the user sees | What the service should record |
| --- | --- | --- |
| Context overflow | A transcript is too long to process now | input size, rejected job, and next action |
| Lost attribution | A commitment has no clear owner | speaker or evidence span missing |
| Invalid action | The result cannot safely update the CRM | schema error and review state |
| Duplicate retry | One call creates two activities | idempotency key and write outcome |
| Cost drift | The estimate differs from actual usage | estimate version, usage, and variance |

This changes the design decision. The cheap API is not the winner if it leaves the application unable to distinguish a rejected transcript from a generated summary that failed validation. A small, inspectable state machine is more useful than a provider-specific promise about “smart” summarization, especially when a solo team has to debug a customer report with only a job ID and a timestamp.

Then make two passes. The first pass produces bounded notes for each chunk: objections, commitments, dates, risks, and unresolved questions. The second pass combines those notes and emits CRM actions in a schema. Chunking is not a free quality upgrade; it is a way to fit an input into a known budget. A chunk can preserve a sentence while losing who said it, so carry a small amount of speaker and section context into every request.

Character count is a useful rough gate, not a token count. Different languages, punctuation, and transcript formatting change the relationship. I would use characters to form candidates, then use the selected runtime's tokenizer or counting endpoint as the authority. I'm not sure a universal chunk size exists for this workload. Measure a representative transcript set, including calls with long monologues and calls with many short turns.

Here is a deliberately provider-neutral first pass. It does not claim that its character target is safe for a particular model; it preserves speaker turns and leaves the final token decision to the adapter.

```ts
export type Turn = {
  speaker: string;
  text: string;
};

export function makeTranscriptChunks(
  turns: Turn[],
  targetChars: number,
): string[] {
  if (!Number.isInteger(targetChars) || targetChars < 1) {
    throw new RangeError("targetChars must be a positive integer");
  }

  const chunks: string[] = [];
  let current = "";

  const flush = (): void => {
    if (current) chunks.push(current);
    current = "";
  };

  for (const turn of turns) {
    const text = `${turn.speaker}: ${turn.text.trim()}`;
    if (!text.trim()) continue;

    if (current && current.length + text.length + 1 > targetChars) {
      flush();
    }

    if (text.length > targetChars) {
      for (let offset = 0; offset < text.length; offset += targetChars) {
        const piece = text.slice(offset, offset + targetChars);
        if (piece.length === targetChars) chunks.push(piece);
        else current = piece;
      }
    } else {
      current = current ? `${current}\n${text}` : text;
    }
  }

  flush();
  return chunks;
}
```

That last-resort slice can split a speaker turn. It is still preferable to silently dropping the tail, but it should be visible in telemetry and reviewed in evaluation. If a single turn regularly exceeds the target, preserve a turn ID and use an adapter-specific strategy rather than pretending the generic helper understands every transcript format.

## What must the CRM action contract reject before a write?

Put admission control between candidate chunking and model dispatch. For each chunk, count input tokens with the same model configuration that will run the request. Reserve space for instructions and the expected output. Reject, re-chunk, or queue work when the total cannot fit the tenant's configured budget. The exact budget belongs in configuration because model context and pricing contracts change; hard-coding a number into the controller turns a current assumption into a hidden outage risk.

An estimate should be a ledger entry, not a promise. Record the input count, reserved output count, number of chunks, selected model route, and an estimate version. After the call, record actual usage and reconcile the difference. Generation length is variable, and a retry can consume more work even when the final CRM update is written once. An idempotency key based on the call ID and pipeline version keeps that retry from creating duplicate activities.

The user-facing decision can stay simple: summarize now, queue for later, or ask for a shorter recording. The backend should be less simple. A request that passes the count gate can still fail validation, time out, or return incomplete structured data. Keep those states separate so “not admitted,” “not generated,” and “generated but not applied” do not collapse into one red error. If a CRM action has no evidence span, the safe result is review-required, not a best guess.

The ledger also gives a solo team a portability boundary. Define an adapter with operations for count, estimate, generate, and usage extraction. Keep prompt construction, chunk ordering, schema validation, retry policy, and CRM writes above it. A provider switch should replace an adapter and its contract tests, not rewrite the sales workflow.

## How should a Node.js API split long text for cheap summarization?

Use ordinary HTTP semantics at the boundary: explicit methods, bounded timeouts, authentication from environment configuration, status inspection, and retry rules that distinguish throttling from invalid input. A `429` can be retried according to the service's guidance; a malformed schema should go to a review path, not an exponential-backoff loop. The transport is not the workflow.

For progress updates, Server-Sent Events are a reasonable fit when the browser needs a one-way stream of states such as `queued`, `counted`, `summarizing`, `validating`, and `applied`. The browser can reconnect, but the server still needs a durable job record; an event stream is not a substitute for state. MDN documents the `EventSource` model and the event-stream format, which is enough to keep this part based on a web standard rather than a provider SDK. Your mileage may vary if the product needs bidirectional live editing; that is a different transport decision.

The output contract deserves more attention than the prose prompt. A useful action object might contain an action type, owner, due date, evidence span, confidence, and “needs review” flag. A date with no evidence span is not ready to write into a CRM. A missing due date can be a valid unknown, while an invented due date is data corruption. Validate types and allowed values, then require a human or product rule for high-impact updates.

Here is the shape I would keep above any provider adapter:

```ts
export type SummaryJob = {
  jobId: string;
  callId: string;
  chunks: string[];
  tokenBudget: number;
  pipelineVersion: string;
};

export type RuntimeAdapter = {
  countTokens(input: string): Promise<number>;
  estimateCost(inputTokens: number, outputTokens: number): Promise<number>;
  summarize(input: string, outputSchema: unknown): Promise<unknown>;
};

export async function admitJob(
  job: SummaryJob,
  runtime: RuntimeAdapter,
  outputReserve: number,
): Promise<{ admitted: boolean; estimatedCost: number }> {
  let inputTokens = 0;
  for (const chunk of job.chunks) {
    inputTokens += await runtime.countTokens(chunk);
  }

  if (inputTokens + outputReserve > job.tokenBudget) {
    return { admitted: false, estimatedCost: 0 };
  }

  const estimatedCost = await runtime.estimateCost(inputTokens, outputReserve);
  return { admitted: true, estimatedCost };
}
```

The adapter contract intentionally says nothing about a vendor's model names or request syntax. That is the point. A direct API, a hosted gateway, or a self-managed gateway can sit behind it, provided the implementation can make its counting and usage semantics precise. The contract test should verify that estimates, actual usage, timeout behavior, and structured output are mapped consistently.

## When is this reliability design the wrong fit for a SaaS feature?

The catch is that a two-pass, token-aware pipeline adds latency, storage, and operational work. It is not suitable when the feature needs an instant one-line preview for a tiny input; a single bounded request may be the better product choice. It is also not suitable for high-stakes CRM automation without domain evaluation and review. Stick with a human approval step when an incorrect owner, amount, or renewal date has material consequences.

Portability has a trade-off too. A common adapter reduces application coupling, but the lowest common denominator can hide useful provider features and make quality differences harder to see. Keep provider-specific options behind explicit capability flags, and test the default path without them. Otherwise the abstraction becomes a second undocumented API that the team has to maintain.

Before shipping, measure p50 and p95 input tokens, chunk count, admission rejections, estimate-to-actual variance, end-to-end latency, retry rate, schema-validation failures, and human corrections to CRM actions. Slice by language and transcript shape. The cheap path is only cheap if it does not create a cleanup queue.

Start with a small replay set of de-identified calls. Check names, commitments, dates, objections, and speaker attribution. Include a call where the final agreement changes an earlier proposal. A map-reduce summary can preserve both statements without knowing which one supersedes the other, so the evaluator must score temporal and conversational context, not just fluent prose.

Ship the ledger with the feature. The API choice can change later; an unmeasured data contract is harder to replace.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/BerriAI/litellm
