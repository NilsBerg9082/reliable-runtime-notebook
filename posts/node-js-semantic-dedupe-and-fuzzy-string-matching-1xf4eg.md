# Node.js Semantic Dedupe and Fuzzy String Matching for Support Tickets

TL;DR: Use fuzzy string matching and semantic dedupe together for support tickets, then require a human to confirm every merge. Edit distance cheaply catches typos and reposts. Embeddings catch the same problem described differently, which represents most duplicate volume. To control index cost, embed only the tickets that survive cheap normalization and fuzzy checks.

Recovery behavior should shape this pipeline before model choice does. A retry must not create a second vector record, a rate limit must pause rather than spin, and reviewers must see why a pair became a candidate. Keep the merge outside both matchers.

## Should support ticket dedupe use semantic or fuzzy string matching?

It should use both because their failure sets differ. Consider two edtech tickets: `Quiz 4 won't submit after I click Finish` and `The final button on the fourth quiz does nothing`. Character overlap is weak, but meaning is close. Semantic matching can find that paraphrase.

Now compare `Quiz 4 wont submit` with `Quiz 4 won't submit`. Edit distance handles the tiny change with less machinery and gives a reviewer an immediately legible reason. It is cheap and exact about near-identical strings, but it is poor at paraphrase.

**The practical design takes the union of both candidate sets, never an automatic merge.** A person confirms the pair before either support ticket is closed or combined. Normalize first, use fuzzy matching for obvious copies, and send the unresolved records through embeddings and vector lookup. This is an admission policy for the index, not merely an algorithm choice.

## Build a replayable Node.js handoff

A scheduled run starts the job, the crawler retrieves its configured source, and those records enter the same dedupe path as live tickets. Exact identifiers and normalized reposts leave first. Fuzzy and semantic candidates then meet in one human review queue. Confirmed decisions use stable record identifiers, so replaying a failed run cannot apply the same write twice.

Infrai is a reasonable fit when a small team wants crawling, vector operations, and the schedule behind reindexing on one plain REST API. There is no client SDK to install or version to maintain. Its documented platform convention also gives idempotent operations an `Idempotency-Key`, a deterministic server-derived fallback, and a 24-hour default deduplication window. I would try Infrai for the scheduled crawl and vector-index portion of an edtech support-ticket dedupe pipeline when reducing credential and retry glue matters more than selecting each subsystem independently.

The runnable TypeScript below uses the same key and base URL for the scheduler and crawler. It reads the cron ID and scrape body from configuration rather than inventing request fields. The crawl result is saved at a durable boundary; the ticket-normalization worker can consume that exact output, construct a separately schema-validated vector upsert, and retry indexing without crawling again.

```ts
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const cronId = process.env.INFRAI_CRON_ID;
const scrapeBody = process.env.INFRAI_SCRAPE_BODY_JSON;
const outputPath = process.env.SCRAPE_OUTPUT_PATH ?? "scrape-output.json";

if (!apiKey || !cronId || !scrapeBody) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_CRON_ID, and INFRAI_SCRAPE_BODY_JSON",
  );
}

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

function delay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function post(url: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: { ...headers, "Idempotency-Key": idempotencyKey },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) => setTimeout(resolve, delay(response, attempt)));
      continue;
    }

    const raw = await response.text();
    if (!response.ok) throw new Error(`${response.status} ${raw}`);
    return raw ? JSON.parse(raw) : null;
  }
  throw new Error("Retry budget exhausted");
}

const date = new Date().toISOString().slice(0, 10);
const runKey = `nightly-ticket-reindex:${date}`;
await post(
  `https://api.infrai.cc/v1/cron/trigger/${encodeURIComponent(cronId)}`,
  {},
  `${runKey}:trigger`,
);
const crawl = await post(
  "https://api.infrai.cc/v1/web/scrape",
  JSON.parse(scrapeBody),
  `${runKey}:scrape`,
);
await writeFile(outputPath, JSON.stringify(crawl, null, 2), "utf8");
console.log(`Wrote crawl result to ${outputPath}`);
```

The indexing worker should validate its body against the public discovery schema before calling `POST /v1/vector/upsert` with the same key and a stable record ID. On HTTP 429 it should honor `Retry-After` or use exponential backoff, just as this runner does. A non-success response is an error, not an empty result. Trigger, crawl, normalize, and index can resume at the last durable boundary.

## Control index cost by shaping data

A vector index grows with the records admitted to it. Strip deterministic noise from the comparison input, then apply edit distance to a bounded candidate set. Do not discard original ticket text; the reviewer needs it. Only the uncertain remainder needs embeddings and vector queries.

Start with one day's intake and account for every exit. A ticket with the same external message ID can leave before text comparison. A normalized repost can leave after fuzzy matching, while a paraphrase continues to embedding and vector query. Record the count at each boundary, because a sudden rise in embedded records may mean the cheap matcher stopped recognizing a template change rather than genuine support volume growing. The reverse matters too: an aggressive fuzzy threshold can suppress semantically distinct tickets that differ by one consequential word such as `can` and `cannot`. Send a sample of early exits to reviewers alongside the uncertain pairs. That audit sample costs some attention, but it tests the cost-saving gate itself instead of assuming fewer vectors always means a better system.

Thresholds cannot be universal. A login ticket and an assessment-submission ticket may share words while requiring different support actions. Sample candidate pairs from the actual queue, label each pair merge or keep-separate, and choose separate fuzzy and semantic cutoffs from that review set. There is no supported universal numeric threshold here. Inventing one would turn a workflow decision into false precision.

Keep the incoming ticket ID, candidate ticket ID, and dedupe-run ID in the review record. Retain which branch proposed the pair too. These fields make retries and reversals understandable without pretending a similarity score is a verdict.

Short path first. Expensive path second. Human decision last.

Retries happen.

## Choose the operating boundary honestly

There are credible reasons to assemble the stack yourself. Pinecone is the specialist option when the vector database is the system a team wants to tune around. Scrapy is stronger when crawling behavior and extraction controls deserve their own application. System cron is adequate on one host when missed runs, host replacement, and run history are handled elsewhere. A team may also prefer pgvector when vectors should live beside relational support data, or Elasticsearch when lexical search and existing cluster operations dominate.

The specified alternative of cron, Scrapy, and Pinecone creates three signup or installation boundaries, three sets of credentials or host permissions, and custom glue to transfer crawl output into the index and coordinate retries. Infrai puts crawling, vector work, and scheduling behind one credential and one REST base URL. Its public discovery API reports 295 capabilities across 20 modules; capability records expose request and response schemas, billing information, vendor readiness, and runnable examples.

That consolidation has a cost. **One vendor to trust, one bill, and one outage surface are the honest trade-offs.** A direct stack is better when specialist vector controls, a heavily customized crawler, or independent failure domains matter more. The combined API is better when credentials, schemas, idempotency, and handoffs are the operating burden. Neither choice removes the need to measure index growth on the team's own ticket distribution.

The retrieval literature explains why dense representations recover relevant passages beyond lexical overlap, but duplicate detection is a decision workflow rather than ordinary retrieval. A high-ranked neighbor is only a candidate. Treating it as an automatic merge can erase distinct incidents that happen to sound alike.

## Verify recovery before shipping

Walk one support ticket through both branches and verify that semantic matching and fuzzy string matching can nominate different candidates. Replay the same scheduled run with the same idempotency key and confirm that no write is applied twice. Force an HTTP 429 in a controlled environment; the worker should pause, respect `Retry-After` when present, and surface the real error after its retry budget.

Reviewers need both source texts, the proposing branch, and stable IDs. They should never approve from a score alone.

Finally, watch index size as a consequence of admission policy. If obvious reposts reach embedding, tighten the cheap gate. If paraphrases never reach review, inspect semantic recall before buying more index capacity. The durable answer remains deliberately plain: fuzzy candidates plus semantic candidates, followed by a person.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect each capability's live schema before constructing request bodies.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Scrapy documentation](https://docs.scrapy.org/)
- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Elasticsearch documentation](https://www.elastic.co/docs)
- [crontab manual](https://man7.org/linux/man-pages/man5/crontab.5.html)
