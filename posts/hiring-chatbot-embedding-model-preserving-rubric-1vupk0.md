# Hiring Chatbot Embedding Model: Preserving Rubric Evidence Across Retrieval Rebuilds

Choose a chatbot RAG embedding model by testing whether a replacement index preserves the evidence needed to apply your hiring rubric. For a media company scoring candidates, the deciding constraint should be evidence coverage across a provider change: the chatbot needs to retrieve the right passages before anyone can assess whether its proposed scores are justified.

Short answer: there is no best embedding model established here. Use a fixed candidate corpus, rubric, and set of evidence questions to compare eligible models, then ship the smallest retrieval pipeline that meets your declared coverage and latency requirements. Keep each embedding configuration in its own index generation so a replacement can be tested and rolled back.

This engineering note proposes that experiment. It reports no benchmark winner. The intended result is a defensible migration decision, with a reviewable link between retrieved evidence and rubric criteria.

## How should an embedding model support chatbot retrieval for hiring?

Start with a deliberately small, synthetic example: a media hiring team has two rubric criteria, source verification and deadline coordination. A candidate packet contains an interview transcript and a work sample. One passage describes checking a disputed quotation against a recording; another describes handing an article to an editor before a scheduled publication window. These are illustrative inputs, not observations about an actual applicant.

Write an evidence question for each criterion. For source verification, ask which passage documents a verification step, and have a reviewer mark the passages that would support an assessment. Keep that reference set separate from the text sent to the embedding model. Otherwise, an experiment can accidentally test whether the model finds an answer you already inserted into its input.

Retrieve evidence first.

A passage containing the words “verification” or “deadline” is not automatically sufficient evidence. The review question is whether the passage supports the particular criterion and whether its source is inspectable. For this experiment, count a criterion as covered only when the retrieved set includes reviewer-accepted evidence. Report missing evidence separately from evidence that was retrieved but interpreted incorrectly. That distinction tells you whether to investigate retrieval or the scoring step.

The relevant failure is concrete: a chatbot assigns a weak source-verification score because the supporting passage never reached its context. A polished explanation cannot repair that omission. Treat a score without cited support as a review item, and preserve an explicit “insufficient evidence” outcome rather than converting every missing passage into a low candidate score.

This costs reviewer time. It also gives the embedding comparison an application-level target beyond similarity scores.

## Make the corpus rebuildable before comparing models

The simple approach is to change the embedding client behind an existing search interface and continue using the same index. Reject that approach as an experimental design: it leaves the relationship between document vectors, query vectors, and the configuration that produced them unclear. Even equal vector dimensions should not qualify two configurations as interchangeable in this workflow. Require a matching generation identifier instead.

Store the source passages and their stable identifiers independently of the embedding output. Each index generation should record its embedding configuration, document preprocessing version, chunking version, vector dimensions, and similarity metric. When any relevant setting changes, build another generation from the preserved passages. This is a proposed application contract; it is not a claim that providers expose identical options.

The portability trade-off is additional storage and rebuild work in exchange for a replacement you can inspect before switching traffic. For a solo builder, that is a more bounded commitment than maintaining several live retrieval paths indefinitely. Start with one active generation and one candidate generation, and retain a prior generation only for the rollback window you actually need.

Hold chunking and the source snapshot fixed during the first model comparison. Then test chunking changes separately. If the model, passage boundaries, and query wording all change together, a coverage improvement does not tell you which change deserves credit. Conversely, preserving a poor chunking policy forever just to simplify comparisons would make portability an obstacle to improving the application.

A generation is the unit of replacement.

## One boundary that keeps the experiment honest

The following TypeScript example expresses a local contract for a candidate-evidence retriever. The identifiers and criterion names belong to the synthetic example above. An adapter must implement the actual storage lookup and apply the packet restriction before returning results; the validation here provides an additional check at the application boundary.

```ts
type Generation = {
  id: string;
  dimensions: number;
};

type Evidence = {
  packetId: string;
  passageId: string;
  text: string;
};

interface EvidenceIndex {
  generation: Generation;
  search(input: {
    packetId: string;
    vector: number[];
    limit: number;
  }): Promise<Evidence[]>;
}

async function retrieveEvidence(input: {
  index: EvidenceIndex;
  queryGenerationId: string;
  vector: number[];
  authorizedPacketId: string;
}): Promise<Evidence[]> {
  const { index, vector } = input;
  if (input.queryGenerationId !== index.generation.id) {
    throw new Error("Query and index generations differ");
  }
  if (vector.length !== index.generation.dimensions ||
      !vector.every(Number.isFinite)) {
    throw new Error("Invalid query vector");
  }

  const hits = await index.search({
    packetId: input.authorizedPacketId,
    vector,
    limit: 5,
  });
  if (hits.some(hit =>
    hit.packetId !== input.authorizedPacketId)) {
    throw new Error("Evidence crossed the packet boundary");
  }
  return hits;
}
```

The limit of 5 is an illustrative experimental setting, not a recommendation or measured optimum. Keep it fixed for the initial comparison and inspect the returned passages. Increase it only as a separate experiment, because returning more material also changes the context passed to the chatbot and the work required to review its citations.

Generation identity catches configuration mismatches only if the calling code attaches the correct identity to vectors. Create that identity where query embedding happens, and pass the vector and identity together. The function cannot prove how a vector was produced. Nor does a packet identifier establish authorization: the caller must obtain the allowed packet from an authenticated application context.

That boundary matters for hiring data. The OWASP Top 10 for LLM Applications identifies risks including sensitive information disclosure and prompt injection. Treat candidate documents as untrusted evidence in this design. Retrieved text should not grant access, change the rubric, or authorize an action just because it contains instructions addressed to the chatbot. Those are application policy decisions.

## Measure the replacement before switching traffic

Use the same evidence questions against both generations and record which reviewer-accepted passages appear in the first 5 results. Include cases with no supporting evidence. A useful review set should distinguish an unsupported criterion from an overlooked passage; otherwise, a model that always retrieves something can look better than it deserves. Keep final candidate scoring in a separate review stage, where each proposed score must identify its supporting passages and the applicable rubric version.

Then measure retrieval latency, query-embedding latency, full chatbot response latency, and the amount of retrieved text passed into generation. Declare your acceptable limits before choosing a winner. There are no measured thresholds in this note, so an embedding choice remains provisional until your corpus supplies those results.

Count the rebuild too.

Record the document volume processed, embedding input consumed, rebuild duration, storage overlap, and failed work that needed another attempt. These are proposed measurements, not estimates of a particular service's behavior. A shorter vector or faster query is only one part of the operational decision; a replacement that repeatedly exceeds your rebuild window may be impractical even if its retrieval results are acceptable. Keep pricing out of the quality verdict and use your actual workload to budget the experiment.

For deployment, prepare the candidate generation away from live traffic. Make the active generation an explicit setting, check that the query embedder matches it, and preserve enough versioned configuration to switch back. Log generation identifiers, passage identifiers, timings, and error categories; avoid putting candidate document text into routine operational logs. On a retrieval failure, return an explicit inability to assess rather than quietly producing a score with missing evidence.

Before copying this design, verify three things with your own review set: rubric evidence survives the replacement, candidate boundaries hold, and the complete request stays inside your latency and resource budgets. If one fails, keep the current generation while investigating that failure. Choose the embedding model only after the retrieval experiment supports the hiring workflow it is meant to serve.

## References

- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
