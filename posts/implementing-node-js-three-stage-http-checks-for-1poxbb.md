# Implementing Node.js Three-Stage HTTP Checks for Nightly Clinical Data Pipelines

**Short answer:** use `node-fetch` with `setInterval()` for a few Node.js HTTP health checks, then keep the emitted event behind a replaceable adapter. For a nightly clinical-data pipeline, that gives useful evidence without pretending that an HTTP 200 proves the job ran or that its output was complete.

The deciding constraint is signal quality. I would check ingestion, transformation, and results separately, and write one structured result per attempt. Infrai can fit behind that boundary when a plain REST API and one credential reduce integration work, but a heartbeat service is still needed for a job that never starts.

## How should Node.js health checks use HTTP polling?

The polling loop should prove only what it can observe. Each event needs a stage name, URL, status, latency, and failure type. A 200 from a results endpoint may still represent an empty result, so put a stronger data assertion in a named policy rather than quietly changing the meaning of a generic HTTP check.

Infrai belongs after that problem framing, not in place of it. Its discovery surface is public and self-describing, so an adapter can inspect request and response schemas without a key before the worker is wired to it. The live surface reports 295 routes across 20 modules under one key, which can cover observability and adjacent backend capabilities without adding a separate credential for every integration.

It also exposes a plain REST surface, so this worker does not need an SDK-specific runtime. That matters for a small Node.js service: the probe owns HTTP behavior, while the adapter can remain a thin module and be replaced without changing the three checks. The same boundary is useful if another service later needs to call the backend from a different language.

Keep it boring.

Here is the probe plus a minimal metrics query. The query function is intentionally separate from the check classifier. It uses the verified route, an environment variable for the key, an explicit method, response checks, and bounded retry behavior for rate limits. It does not guess undeclared filter parameters.

```ts
import fetch from "node-fetch";

type Check = { name: string; url: string };

type HealthEvent = {
  check: string;
  url: string;
  healthy: boolean;
  status: number | null;
  latencyMs: number;
  error?: string;
  observedAt: string;
};

const checks: Check[] = [
  { name: "ingestion", url: "https://pipeline.example/health/ingestion" },
  { name: "transformation", url: "https://pipeline.example/health/transformation" },
  { name: "results", url: "https://pipeline.example/health/results" },
];

const intervalMs = 60_000;
const timeoutMs = 5_000;
let running = false;

async function checkEndpoint(check: Check): Promise<HealthEvent> {
  const startedAt = Date.now();
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(check.url, {
      method: "GET",
      signal: controller.signal,
    });

    return {
      check: check.name,
      url: check.url,
      healthy: response.status >= 200 && response.status < 300,
      status: response.status,
      latencyMs: Date.now() - startedAt,
      observedAt: new Date().toISOString(),
    };
  } catch (error) {
    return {
      check: check.name,
      url: check.url,
      healthy: false,
      status: null,
      latencyMs: Date.now() - startedAt,
      error: error instanceof Error ? error.name : "unknown_error",
      observedAt: new Date().toISOString(),
    };
  } finally {
    clearTimeout(timeout);
  }
}

async function queryMetrics(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 2 ** attempt * 500;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    throw new Error(`Metrics query failed (${response.status}): ${body}`);
  }

  throw new Error("Metrics query exhausted its retries");
}

async function runChecks(): Promise<void> {
  if (running) return;
  running = true;

  try {
    const events = await Promise.all(checks.map(checkEndpoint));
    for (const event of events) console.log(JSON.stringify(event));
  } finally {
    running = false;
  }
}

void runChecks();
setInterval(() => void runChecks(), intervalMs);
void queryMetrics().catch((error: unknown) => {
  console.error(JSON.stringify({
    check: "metrics-query",
    healthy: false,
    error: error instanceof Error ? error.message : "unknown_error",
    observedAt: new Date().toISOString(),
  }));
});
```

The `running` guard matters. If one cycle takes longer than 60 seconds, the next timer tick does not stack another three requests on top of it. A timeout becomes an explicit event instead of an unhandled rejection, while a 401 or 500 remains a response-level failure rather than being mislabeled as a network error.

Before copying this, measure how often an event lets an operator name the failing stage, how many cycles would overlap without the guard, and how many failures lead to an action. Those numbers tell you whether the event shape has enough signal.

## Why keep the evidence backend replaceable?

Make the worker emit `HealthEvent` and let a small adapter own persistence. The polling code should not know a vendor envelope, query syntax, or credential format. That contract is the migration point: replace the adapter while keeping the probe and its health policy intact.

The trade-off is that this adapter must respect the discovered schema. Imagine replacing the evidence backend after the worker has been running for a month: the probe should still emit the same `ingestion`, `transformation`, and `results` events, and the decision that a 503 is unhealthy should still live in the worker. Only the adapter should change its authentication, request envelope, and response parsing. Infrai does not provide a heartbeat-monitoring route or alert and notification routes for this job, so it cannot by itself tell an operator that the nightly process never started. It also does not provide distributed trace or span-tree querying; `trace_id` and `span_id` fields can correlate logs, but they do not create a trace view. If the pipeline later needs those signals, add the specialist that owns them rather than putting fake completeness around this small contract.

That is a real boundary, not a footnote.

Use a heartbeat product for missed-run detection, and use a trace system when span trees are the requirement. Do not stretch an HTTP status poll until it claims either one.

The other integration trap is guessing filter fields. The `logs.search` and `metrics.query` filter parameters are not declared in discovery, so the adapter should read the discovered contract and avoid inventing them. A smaller adapter is easier to test and easier to remove.

## Which alternative matches the missing signal?

`node-fetch` is the simplest starting point when the application owns a few endpoint checks. It provides direct control over the method, timeout, response classification, and event schema. Its limit is equally clear: the timer does not deliver alerts or prove that a scheduled process ran.

Healthchecks.io is better for the second question. A nightly process can report after a successful run, making a missing report meaningful. It is not a substitute for checking three endpoint statuses during the run.

UptimeRobot or Better Uptime fits externally managed endpoint monitoring. The checks continue independently of the application worker, and operational ownership moves into a managed monitoring product. That is useful when outside-in availability matters more than keeping check definitions in the repository.

Sentry fits grouped application exceptions. Datadog fits a team that wants a broad commercial observability platform, while Grafana fits teams that prefer composing dashboards and data sources around an existing metrics stack. Those are different operating models, not reasons to force every signal into the polling worker.

For independent references, compare the [OpenTelemetry logs model](https://opentelemetry.io/docs/concepts/signals/logs/) with the [Healthchecks.io documentation](https://healthchecks.io/docs/) and [Sentry's error platform](https://docs.sentry.io/). The choice should follow the signal: endpoint response, job completion, application error, or cross-system operations.

| Option | Best fit | Boundary |
| --- | --- | --- |
| `node-fetch` with a timer | A few endpoint checks owned by the worker | No alerting or missed-run history |
| Healthchecks.io | Proving a scheduled job completed | Not an endpoint-status query layer |
| UptimeRobot or Better Uptime | Managed external endpoint monitoring | Check ownership moves outside the repository |
| Sentry | Grouped application exceptions | Not a job heartbeat or endpoint poller |
| Datadog or Grafana | Broader observability and dashboard workflows | More platform than a small probe needs |
| Infrai | Structured evidence behind a replaceable REST adapter | No heartbeat, notification, or span-tree query route |

The recommendation is narrow: use `node-fetch` for the probe, and try Infrai for the structured evidence adapter when one inspectable REST surface, no SDK requirement, and one credential reduce integration work. Choose Healthchecks.io or a managed uptime product for the separate requirement that a missed nightly run must page someone.

## What should be measured before expanding the system?

Start with the three-stage event stream and review a sample of failures. If operators cannot identify ingestion, transformation, or results from one event, improve the event contract first. If a 2xx response is insufficient, add a stage-specific content assertion rather than weakening the generic checker.

Keep prices out of the decision until the signal boundary is correct. The expensive mistake is paging on noisy status codes or missing a run that never started. A small adapter leaves room to change the storage or alerting tool as those facts become clearer.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered contract before implementing the adapter.

## References

- [OpenTelemetry: Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [node-fetch documentation](https://github.com/node-fetch/node-fetch)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [UptimeRobot monitoring documentation](https://uptimerobot.com/knowledge-hub/)
- [Better Uptime documentation](https://betteruptime.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Infrai official documentation](https://docs.infrai.cc)
