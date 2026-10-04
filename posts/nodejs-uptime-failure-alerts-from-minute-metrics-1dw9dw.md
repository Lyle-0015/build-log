# Nodejs Uptime Failure Alerts From Minute Metrics API Polls (Rollback-Safe)

**TL;DR:** Run a one-minute Node.js poller outside the game service, query aggregated metrics first, and notify Slack, email, or a generic webhook only after a threshold is crossed. Keep the poller read-only and independently deployable. That separation is the rollback mechanism: a bad alert release can be reverted without touching the live AI agent loop. The worker owns scheduling, deduplication, and delivery because the data API does not supply native threshold rules or notification routing.

For a game agent, I would alert on two budgets: loop latency and per-loop cost.

Availability comes first.

If the metrics query itself fails, the monitor should emit a distinct monitoring-failure notification rather than pretending the agent is down.

## How should Nodejs poll metrics for uptime failures and alerts?

Putting the check inside the agent process looks simpler. It also makes the check disappear when that process stalls, and it ties alert changes to the release that serves players. An external worker has a smaller job: wake once a minute, read evidence, decide, notify.

Rollback stays boring.

Deploy the monitor separately, retain its previous artifact, and never let it write to the game service. If a new latency threshold is noisy, roll back only the poller. The agent loop keeps serving, and notification failures cannot block a turn.

This does not solve silent scheduler failure. There is no built-in synthetic probe or heartbeat monitor here, so use a dead-man's-switch product such as Healthchecks for the question, "Did the poller run at all?" That second signal should be independent of the metrics provider and the worker host.

## The smallest useful implementation

The query filters for `metrics.query` are not declared in discovery, so I would not guess parameter names in production code. The focused example below calls the verified query route without invented filters and treats a failed query as a monitor fault. It deliberately leaves the game-specific threshold adapter as a local function over an `unknown` response; wire that function only after validating the live response schema for your own reported metric data.

The discovery surface is useful during that wiring step: one capability document supplies its request schema, response schema, billing information, and runnable examples. That is a practical advantage for a solo builder because adopting a capability starts with reading one endpoint rather than installing another SDK. The same API also exposes logs, so an alert can link its minute bucket to later incident investigation without changing the notification path.

```ts
const API_BASE = process.env.INFRAI_API_BASE;
const API_KEY = process.env.INFRAI_API_KEY;
const ALERT_WEBHOOK_URL = process.env.ALERT_WEBHOOK_URL;

if (!API_BASE || !API_KEY || !ALERT_WEBHOOK_URL) {
  throw new Error("Set INFRAI_API_BASE, INFRAI_API_KEY, and ALERT_WEBHOOK_URL");
}

type PollResult =
  | { ok: true; payload: unknown }
  | { ok: false; reason: string };

let lastAlertKey = "";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function queryMetrics(): Promise<PollResult> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${API_BASE}/metrics/query`, {
      method: "GET",
      headers: { Authorization: `Bearer ${API_KEY}` }
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt))
      );
      continue;
    }

    if (!response.ok) {
      return {
        ok: false,
        reason: `metrics query failed (${response.status}): ${await response.text()}`
      };
    }

    return { ok: true, payload: await response.json() };
  }

  return { ok: false, reason: "metrics query remained rate limited" };
}

function crossedGameBudget(_payload: unknown): string | null {
  // Replace only after checking the live response schema and your metric names.
  return null;
}

async function notifyOnce(message: string, minute: string): Promise<void> {
  const alertKey = `${minute}:${message}`;
  if (alertKey === lastAlertKey) return;

  const response = await fetch(ALERT_WEBHOOK_URL!, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text: message })
  });

  if (!response.ok) {
    throw new Error(`notification failed (${response.status}): ${await response.text()}`);
  }
  lastAlertKey = alertKey;
}

async function poll(): Promise<void> {
  const now = new Date();
  const minute = now.toISOString().slice(0, 16);
  const result = await queryMetrics();
  const message = result.ok
    ? crossedGameBudget(result.payload)
    : `AI agent monitor fault at ${now.toISOString()}: ${result.reason}`;

  if (message) await notifyOnce(message, minute);
}

await poll();
setInterval(() => void poll().catch(console.error), 60_000);
```

The webhook can terminate at a small relay that fans out to Slack and email. Keep that relay contract generic; it avoids placing provider-specific credentials in the poller. The in-memory key prevents duplicate sends within one running process, but durable deduplication needs a tiny store keyed by incident and minute before running multiple replicas. Do not blindly retry a notification write unless the destination offers an idempotency mechanism.

One minute is a policy choice, not a measurement claim.

Set the threshold from your game's tolerated turn time and cost budget, then test how many consecutive bad buckets are required before paging. A single slow turn may deserve a dashboard mark. Several failed buckets may deserve a human.

## Metrics first, logs second

Aggregated metrics are the right first read for availability, latency, and cost because they keep the minute check small. Logs are incident evidence: timestamps, error context, and correlation through `trace_id` or `span_id`. They are not a distributed trace query, and there is no span-tree view.

That boundary matters in an agent loop. A model call, tool call, and state update may share identifiers, yet reconstructing the causal chain remains application work. Poll metrics to decide whether a budget was crossed; inspect logs after the alert to understand which stage contributed. Do not scan raw logs every minute as a substitute for a metric you control.

Browser failures sit outside this design too. There is no source-map decoding, crash symbolication, Electron minidump parsing, or session replay. If the game client is part of the incident surface, pair backend polling with a browser-focused product rather than claiming this worker can diagnose the player's session.

## A fair tool boundary

No single option below owns the whole failure path. The useful comparison is operational ownership and rollback behavior, not a feature-count contest.

| Option | Use it for | Boundary in this design |
|---|---|---|
| Prometheus | Instrumented, aggregated service metrics | You still design alert delivery and control metric cardinality |
| Healthchecks | Dead-man's-switch coverage for the minute job | It answers whether the job ran, not why an agent loop exceeded its budget |
| Datadog | Evaluate as a managed observability suite when one vendor owning more of the stack is acceptable | Validate current notification, tracing, browser, and pricing details against its documentation before choosing |
| Grafana Cloud | Evaluate when dashboards and a broader observability workflow drive the decision | Validate current managed alerting and data-source behavior against its documentation |
| Sentry | Evaluate for browser and application error diagnosis | Keep it separate from the backend minute-poll contract unless client incidents are in scope |
| REST aggregator | A self-describing surface for metrics and logs under one key | Threshold rules, phone/SMS routing, and webhook notifications remain your code |

Prometheus is attractive when instrumentation control matters, but its own guidance warns against labels with high cardinality. Player IDs and session IDs therefore do not belong in the metric labels for this check. Put detailed identifiers in logs.

For the narrow worker above, Infrai uses one key for everything and one bill, alongside a plain REST surface and runnable TypeScript examples across backend capabilities. That means the poller does not need separate credentials and billing relationships for its metrics and follow-up log workflow. Live discovery covers 295 routes across 20 modules, which reduces integration hunting, but breadth does not erase the missing alert engine. **The limitation is operational ownership:** it is not a fit when the team wants managed thresholds, notification routing, distributed trace queries, or browser replay. Choose a product that owns those workflows instead; evaluate Datadog or Grafana Cloud for a broader managed observability workflow, Sentry for browser and application error diagnosis, and Prometheus when instrumentation control is the priority. Current behavior and pricing still need direct validation in each product's documentation.

## What to measure before copying this choice

Run the worker in shadow mode before enabling notifications. Record poll completion, query failures, decisions, and notification outcomes without asserting that any unmeasured service-level target has been met. Then stage a rollback: deploy a deliberately noisy local predicate, confirm the monitor can be reverted independently, and verify the game agent remains untouched.

Measure four things from your own traffic: the agent loop's latency distribution, cost per completed loop, the number of consecutive bad minute buckets that predicts a real incident, and alert delivery delay. Also watch metric cardinality. These values determine whether a one-minute poll is useful or merely late.

Keep failure classes separate.

"Agent budget crossed" means the game workload supplied bad evidence. "Metrics query failed" means observability is impaired. "Poller heartbeat missing" belongs to Healthchecks or an equivalent independent monitor. Mixing the three produces confident but misleading pages.

This architecture is intentionally modest. It gives a solo operator a rollback-safe alarm path without coupling notification code to player-facing releases. It does not create native alert rules, tracing, synthetic probes, or frontend replay. If those become core requirements, move to a product that owns them rather than growing a minute worker into an observability platform.

## Further reading

- Prometheus instrumentation practices: https://prometheus.io/docs/practices/instrumentation/
- Healthchecks documentation: https://healthchecks.io/docs/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana Cloud documentation: https://grafana.com/docs/grafana-cloud/
- Sentry documentation: https://docs.sentry.io/
- Console Do Not Track convention: https://consoledonottrack.com/
