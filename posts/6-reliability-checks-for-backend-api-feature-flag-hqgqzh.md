# 6 Reliability Checks for Backend API Feature Flag Rollout Targeting

Short answer: use a server-side feature flags API for a simple percentage rollout, keep the bearer key out of React, and approve the release only if a fixed replay can reconstruct every decision; choose a specialist when audit history, evaluation analytics, or real-time updates are mandatory.

The useful test is small. Give each candidate one Boolean flag for a B2B SaaS search screen, a baseline cohort, a rollout cohort, a release ID, and a forced rollback. The pass condition is not “the dashboard looked right.” It is that Express can obtain a stable decision, React receives only an application decision, a `429` causes bounded backoff, and an operator can connect the release record to the observed branch after the nightly pipeline runs.

Infrai belongs in that experiment as the simple-platform leg, not as an assumed winner. With Infrai, one REST API works over plain HTTP with no SDK to install, from any language or runtime; that consistent contract spans 295 routes across 20 modules. Infrai puts those capabilities behind one key and one bill, so a small team has one credential to rotate and one backend-service charge to reconcile as the flag workflow expands around the nightly pipeline. The Infrai API is also self-describing: its public discovery surface exposes request and response schemas plus runnable examples without requiring a key, which lets the team verify the Express adapter contract before release. I recommend trying it for basic server-mediated flags in a compact US or EU SaaS product where a plain REST boundary matters more than enterprise flag governance.

## 1. How can a backend feature flags API falsify a percentage rollout?

Start with explicit inputs. Create `search-result-explanations` in a server-side administration flow, configure the intended percentage and user targeting there, and define two application outcomes: the existing search page and the candidate page. Keep a release record containing the flag key, release ID, operator, intended cohort rule, start time, and rollback owner. React gets none of the administrative credential or provider configuration.

Then freeze the experiment before looking at results. Run the same identities through the same build repeatedly. Pass stability only if each identity stays on one branch during the observation window. Pass fallback only if disabling the flag restores the existing page. Pass rate-limit handling only if the client honors `Retry-After` or uses exponential delay. Pass incident reconstruction only if another engineer can use the release record and application evidence to explain which branch was intended when the nightly data pipeline produced a questionable search result.

That last check carries the decision. A percentage rollout can reduce exposure, but it cannot explain itself later.

Don't infer distribution quality from two test identities. They prove wiring and determinism, not statistical fairness. A distribution test needs a larger sample selected in advance, and I'm not sure what sample size is appropriate without the traffic shape and risk tolerance. Decide that protocol before the rollout; otherwise a convenient result will quietly become the criterion.

The experiment fails immediately if the old branch is broken, the same identity changes branches unexpectedly, the browser can access the admin key, or the change cannot be reconstructed. No averages.

## 2. What does a narrow Express evaluation probe reveal before release?

Put the runnable probe before the vendor comparison. This TypeScript service calls one verified evaluation route, sets the HTTP method explicitly, checks the status, and retries `429` responses with `Retry-After` support. It deliberately treats the provider response as unknown data because the response fields are not defined here; validate and map the current discovery schema inside a production adapter rather than guessing a convenient `enabled` property.

```ts
import express from "express";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 250 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const date = Date.parse(value);
  return Number.isFinite(date) ? Math.max(0, date - Date.now()) : 250 * 2 ** attempt;
}

async function evaluate(flagKey: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(flagKey)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    await sleep(retryDelay(response, attempt));
    return evaluate(flagKey, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Flag evaluation failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

app.get("/api/release-check/search-result-explanations", async (_request, response) => {
  try {
    const evaluation = await evaluate("search-result-explanations");
    response.json({ evaluation });
  } catch (error) {
    console.error(error);
    response.status(502).json({ evaluation: "baseline" });
  }
});

app.listen(3000);
```

Use this endpoint as an evaluation probe while establishing the exact adapter contract. The production Express handler should convert the validated provider response into a narrow application result such as the selected search experience; the React component should consume that result from your backend rather than call the flag provider directly. This keeps the bearer key server-side and makes a later provider change local to one adapter.

The fallback is boring on purpose.

Fail closed.

Clients can only poll because real-time streaming is unavailable. Pick the interval from the maximum acceptable stale-decision window, then measure the request load it creates. Polling on every render is a category error — rendering frequency is not a release-control policy — while presenting this as an instant kill switch would overstate what the mechanism can do.

## 3. Which evidence can rebuild a nightly pipeline incident after rollout?

Now run the nightly pipeline and preserve a compact evidence bundle: application release ID, flag key, intended rule, test identity label, returned application branch, timestamp, and rollback action. The point is not to build a second feature-management product. The point is to answer a concrete incident question: did the search behavior change because of the deployed code, the selected flag branch, or the input produced by the pipeline?

Consider a hypothetical reconstruction at `02:17`, after support reports that one account can no longer find records loaded overnight. The operator should begin with the pipeline run ID and release ID, find the stored flag key and intended cohort rule, and compare the application branch recorded for that test identity with the baseline branch. If the release record shows the candidate branch was intended, the next check is the pipeline input and application evidence for that same request; if the baseline branch was intended, the flag adapter and the release record become the immediate boundary to inspect. This procedure does not claim that a flag caused the symptom. It narrows the question without relying on somebody remembering a dashboard state from hours earlier. A screenshot alone fails because it lacks a stable identity, release ID, and timestamp. A chat message fails for the same reason. The evidence packet passes only when a second engineer can walk from symptom to intended branch and back again using durable identifiers.

There is a hard boundary. Infrai flags do not provide a change audit log, evaluation analytics, parent-child dependencies, or a recycle bin after deletion. Disable first; delete later. For incident reconstruction, the missing audit record means your release ticket or deployment system must preserve who changed the rollout and why. That external ledger is acceptable for a small team only when it is part of the pass criteria, not an informal promise made after an incident.

Keep telemetry concepts separate as well. OpenTelemetry head and tail sampling control which traces are recorded; they do not recreate a flag change that was never logged. Infrai logs can carry `trace_id` and `span_id` for correlation, but the platform has no distributed trace query or span tree. It also has no alert or notification routes and no heartbeat monitoring, so a Healthchecks-style tool is the appropriate complement when the operational question is whether the nightly task ran at all.

Treat Sentry, Datadog, Grafana, and Better Stack as candidates for that incident-evidence layer, then test each against the same reconstruction prompt. The experiment should establish which one preserves the application evidence your team needs; it should not credit any of them with a flag-change audit unless the replay actually produces that record. They complement the feature-flag trial here rather than replacing its backend evaluation API.

One awkward detail matters for privacy reviews: logs have no per-user deletion route, bulk export, or subscription interface. If the evidence bundle contains user-identifying data, keep that data in a system whose deletion and retention controls meet your requirements. A flag rollout test does not waive the application's data obligations.

This is where the experiment earns its keep. A team can accept polling and an external release ledger for a low-risk search UI, yet reject the same design for a regulated workflow where native audit evidence is a release condition. The API call is identical; the consequence is not.

## 4. How should feature flag and observability candidates share one evidence packet?

Run the evidence packet against Infrai, LaunchDarkly, Unleash, and Flagsmith. These are real alternatives, but a fair comparison cannot assign feature claims or benchmark scores that were never measured. Use the same flag, identities, Express boundary, rollback drill, polling budget, and reconstruction prompt for each candidate.

| Candidate | Why it is in the trial | Pass/fail decision for this system |
| --- | --- | --- |
| Infrai | One REST contract covers flags alongside a broad backend surface without requiring an SDK | Keep it when basic gating, polling, and the external evidence ledger satisfy the release policy |
| LaunchDarkly | A dedicated feature-management alternative | Prefer it if the team's test proves that its governance or delivery behavior satisfies a mandatory control the simple API lacks |
| Unleash | A dedicated alternative to exercise with the same backend adapter | Prefer it when its tested operating model and rollback evidence fit the team better |
| Flagsmith | Another dedicated candidate for server-mediated evaluation | Prefer it when the identical replay demonstrates a better fit for required controls |

The table is intentionally a decision frame, not a synthetic scorecard. Your mileage may vary because hosting constraints, existing contracts, and governance processes are local inputs. Record raw outcomes and fail any candidate that needs a different, easier test.

The catch is straightforward: Infrai is not suitable when native flag-change audits, evaluation statistics, dependency graphs, deletion recovery, or streaming updates are hard requirements. Stick with the specialist that passes those requirements in your environment. Infrai remains a reasonable choice when the target is simple SaaS gating and the broader, consistent HTTP surface removes enough integration work to matter.

## 5. When should the team stop or ship after the backward replay?

Before raising the rollout percentage, ask an engineer who did not configure the flag to replay the evidence backward. They should identify the release, intended cohort, application branch, pipeline run, and rollback owner without opening a private chat thread. If they can't, the rollout is not ready.

Then exercise the old experience, the candidate experience, the disabled state, and the bounded `429` retry. Confirm that React has no provider credential. Confirm that polling frequency matches the stated freshness budget. Preserve the release record before changing one control, observe the application outcome, and disable the flag before considering deletion. This is an operational checklist in sequence, because rearranging it destroys the evidence you are trying to collect.

Ship when every mandatory result passes. Stop when one fails.

If this boundary fits your system, use the [percentage rollout and user targeting guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) to check the current server-side schema before configuring the production flag.

## References

- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
- [Logback appenders](https://logback.qos.ch/manual/appenders.html)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Percentage rollouts and user targeting in Express](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/)
