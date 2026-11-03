# Managed Metrics Dashboard API: Startup Shipment Incident Reconstruction Explained

Choose a managed metrics API only if you can follow a dashboard anomaly back to the evidence needed to explain a shipment incident. For a small Node.js startup serving Europe and the US, that is a more useful acceptance test than the number of dashboard panels. TL;DR: keep aggregate metrics small, retain shipment evidence separately, and make the connection between them explicit.

The instrumentation guidance in References warns against high-cardinality labels and recommends investigating other processing approaches when a metric could have substantial cardinality. That makes a shipment identifier a poor default metric label. The design below keeps identifiers in an incident evidence store and uses bounded regional metrics to find the interval worth investigating. It is a proposed implementation, with synthetic data, rather than a report of a customer incident.

## What must survive a customer incident?

Start with a concrete support question: why did a shipment assistant tell a customer that a delivery was confirmed? A total of successful requests cannot answer it. For this workflow, require a record of the delivery-state evidence the assistant used, the version of the decision policy, the resulting decision, and the correlation identifier connecting that decision to its request. If those records are unavailable, write down that limit before choosing a dashboard.

Keep the evidence.

The proposed data flow is short: the Node.js application records a decision event in the appropriate regional evidence store, then emits an aggregate metric with bounded labels. The dashboard selects a region and time window; an authorized support lookup uses that same window to find candidate decision records and then follows their correlation identifiers. A managed API can handle the aggregate side of this flow. Evaluate the evidence lookup separately, including its retention and export behavior.

## A minimal implementation with explicit evidence boundaries

The following TypeScript example runs without external dependencies in a TypeScript-capable Node.js environment. Its in-memory sinks make the data flow inspectable; they are demonstration adapters, and their contents disappear when the process ends. Replace the evidence adapter with durable regional storage before using this pattern for customer support. The shipment references and policy version below are arbitrary example values.

```ts
type Region = "eu" | "us";
type Decision = "confirmed" | "needs_review";
type Evidence = {
  eventId: string;
  requestId: string;
  shipmentRef: string;
  region: Region;
  occurredAt: string;
  policyVersion: string;
  sourceEventId: string;
  sourceState: "delivered" | "pending";
  decision: Decision;
};
type Metric = {
  name: "shipment_decisions_total";
  region: Region;
  decision: Decision;
  value: 1;
};
type Sinks = {
  saveEvidence: (event: Evidence) => Promise<void>;
  emitMetric: (metric: Metric) => Promise<void>;
};

async function recordDecision(event: Evidence, sinks: Sinks) {
  await sinks.saveEvidence(event);
  await sinks.emitMetric({
    name: "shipment_decisions_total",
    region: event.region,
    decision: event.decision,
    value: 1,
  });
}

async function main() {
  const records = new Map<string, Evidence>();
  const totals = new Map<string, number>();
  const sinks: Sinks = {
    async saveEvidence(event) {
      records.set(event.eventId, { ...event });
    },
    async emitMetric(metric) {
      const key = `${metric.region}:${metric.decision}`;
      totals.set(key, (totals.get(key) ?? 0) + metric.value);
    },
  };

  await recordDecision({
    eventId: "decision-demo-1",
    requestId: "request-demo-1",
    shipmentRef: "shipment-demo-1",
    region: "eu",
    occurredAt: new Date().toISOString(),
    policyVersion: "delivery-policy-v1",
    sourceEventId: "carrier-event-demo-1",
    sourceState: "pending",
    decision: "needs_review",
  }, sinks);

  console.log([...totals.entries()]);
  console.log(records.get("decision-demo-1"));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

There are 2 region values and 2 decision values in this example: 4 possible label combinations for the counter. Neither shipment references nor request identifiers enter the metric. That fixed vocabulary is deliberate. The cited instrumentation guidance explains why labels need care; the precise regional and decision vocabulary here is an application design choice.

The example records the observed source state alongside the resulting decision. For an AI-assisted delivery workflow, extend the evidence schema with the model identifier and prompt-template version if support needs them to explain a response. Retain an approved response record or a reference to it when exact wording matters. Avoid collecting entire prompts by default: decide which fields are necessary for reconstruction and who may retrieve them before adding customer text to the store.

This is a small starting point.

It doesn't implement a carrier integration, a dashboard, or an LLM call. Its useful boundary is the pair of interfaces: the aggregate transport can change without changing which evidence a delivery decision must retain.

## Decide what a successful write means

The order of operations is intentional. If saving evidence fails, this example stops before emitting the metric. If emitting the metric fails after evidence is saved, the function rejects with evidence already present. A caller that retries the whole operation can therefore produce a duplicate metric increment; the demonstration map replaces a matching event record, but the counter increments on every accepted call.

That is the trap.

Don't interpret the counter as an exact ledger of unique delivery decisions under retries. For a deployment that needs that guarantee, require a durable event identifier, deduplication rules, and a delivery process whose retry behavior is tested. One possible design is to commit the evidence and a pending metric-delivery record together, then let a worker send the aggregate update. That proposal adds storage and worker ownership. It is justified only if the required accounting semantics outweigh the extra operational work.

The simpler path is to treat the dashboard as a signal and reconstruct the incident from the durable evidence store.

Document that choice.

A founder shipping one assistant feature may reasonably accept approximate operational counts while requiring complete evidence for decisions that change a customer-visible delivery status. The trade-off is explicit: this counter is not suitable for exact settlement or billing. Its limitation is duplicate counting under retries; use a deduplicated decision ledger for those jobs. The evidence map has a separate limitation: it cannot survive a process restart.

Also decide whether evidence persistence belongs in the customer request path. Waiting for a durable acknowledgement spends part of the latency budget. Moving the write elsewhere requires a durable handoff and a defined response when that handoff fails. The code chooses to wait; measure that choice with your own workload before adding an asynchronous path. No latency improvement is assumed here.

## How should a startup evaluate a managed metrics dashboard API?

A simpler metrics dashboard can reduce the amount of monitoring infrastructure a startup chooses to operate. It still needs to pass the actual support workflow. Ask a candidate service to demonstrate ingestion, bounded-label filtering, time-window queries, and export using your synthetic dataset. Keep the same evidence interface during the exercise so that dashboard convenience does not quietly redefine the evidence contract.

For the Europe and US deployment, define the permitted destination for each kind of data before testing endpoints. The `eu` label in the example is a routing input you must enforce in the storage adapter; its presence does not establish where anything is stored. Verify processing locations, access controls, retention, and exports against the service's documented contract. A regional chart alone is insufficient evidence for that decision.

Use a specific drill: create a pending carrier event, record a `needs_review` decision, locate its regional aggregate, and retrieve the decision record by the dashboard's time window. Then repeat with a delayed source event and a repeated request. Preserve both the source event identity and the decision identity so that the investigator can distinguish a new observation from another attempt to process an existing one. Decide how event time and ingestion time appear in production records; a single timestamp in this minimal example is not enough to analyze delayed ingestion.

Test an exporter failure too.

Establish whether the application blocks, buffers, retries, or drops an aggregate update, and check that the evidence remains retrievable under the chosen policy. Require a clear error signal for evidence-write failures. A clean graph should never be your only acceptance criterion.

Cost belongs in this exercise as a constraint. Estimate aggregate volume, evidence retention, query frequency, and the effort required to operate the adapters. Adding shipment identifiers to labels works against the bounded-label design; retaining unrestricted response text adds a different storage and access burden. Neither is a substitute for deciding what support actually needs. Pricing alone cannot tell you whether a disputed delivery can be reconstructed.

## Ship with a support lookup that you can test

Before rollout, assign the evidence schema a version and verify that support can retrieve a synthetic incident with the permissions it will actually have. Choose a retention window from the period during which your business must investigate customer disputes, then test deletion and document what becomes unavailable after expiry. Keep raw shipment references out of dashboard labels and restrict their lookup to the evidence workflow.

Next, deploy the metric adapter with a bounded label vocabulary and exercise its failure path. Re-run the same reconstruction drill after changing a prompt, decision policy, storage adapter, or export destination. Include a rollback test: a record written by the newer deployment must remain interpretable by the support workflow after the application rolls back. Store the versions needed to make that possible.

Finish by exporting a synthetic incident and reconstructing it without the dashboard. If that fails, the evidence contract still needs work. Accept an aggregate service only after it passes the regional and operational checks. Retain enough independent evidence to explain what the assistant told the customer and why.

## References

- Instrumentation best practices, including label cardinality guidance: https://prometheus.io/docs/practices/instrumentation/
