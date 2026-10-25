# Handling Moderation API Timeout Errors with 3-State Retry Logic for Game Patches

**Short answer:** In Node.js, retry only replay-safe moderation API timeout errors inside one total deadline, then return an explicit infrastructure status instead of inventing findings.

A moderation timeout should not be treated as a moderation result. For a game patch review service, the useful contract is stricter: a successful call returns parseable findings, a bounded retry may repeat a request that is safe to repeat, and an exhausted deadline becomes an explicit infrastructure outcome. That separation prevents a slow dependency from being reported as either a clean patch or a confirmed violation.

My default is three buckets: transport failure, service rejection, and valid moderation output. Retry only the first bucket when the request is replay-safe and the remaining deadline justifies another attempt. The result passed to the review queue must carry a status as well as findings. An empty findings array is valid data; it is not a timeout fallback.

## How should Node.js handle moderation API timeout errors?

A timeout tells the caller that its waiting budget ended. It does not establish what the remote service would have returned. That distinction matters in a code-review workflow: marking a changed combat rule as clean because the moderation request timed out creates a false negative, while marking every timeout as a content violation creates noisy false positives and blocks releases for the wrong reason.

That is the whole problem.

The first design choice is therefore an outcome envelope, not a retry loop. Keep the machine-readable status small and stable. `approved`, `flagged`, `retry_exhausted`, and `invalid_response` are enough for the queue to route work; the human-facing explanation can be richer later. Preserve the attempt count and elapsed time for operations, but do not let those fields change the meaning of the finding set.

This policy has a real trade-off: a short deadline improves release latency but sends more patches to human review, while a longer one improves completion rate at the cost of queue age. The 8,000 ms example below is a starting constraint, not a universal setting.

Consider a patch that changes a dialogue string, a quest reward, and a combat effect in one commit. The worker submits one review payload, hits its 8,000 ms deadline, and receives no content decision. If the queue stores only `findings: []`, the next system cannot tell whether the patch was checked and found clean or never checked at all. With the envelope above, `retry_exhausted` keeps the patch out of the clean path, while the request identifier lets an operator correlate the timeout with queue age and dependency latency. That extra state costs a few fields and one routing branch, but it removes a much more expensive ambiguity at release time. The same rule applies to an empty, valid response: it is a content result only after the response passes schema validation.

## A bounded retry loop for Node.js

The example below uses `AbortController`, a total deadline, exponential delay, and jitter. It has no provider-specific route or response shape. The adapter at the edge owns authentication and request serialization; the review logic owns classification and validation.

```ts
type Finding = {
  rule: string;
  severity: "low" | "medium" | "high";
  message: string;
};

type ModerationOutcome =
  | { status: "approved" | "flagged"; findings: Finding[]; attempts: number }
  | { status: "retry_exhausted" | "invalid_response"; findings: []; attempts: number };

function isFindings(value: unknown): value is Finding[] {
  return Array.isArray(value) && value.every((item) => {
    if (!item || typeof item !== "object") return false;
    const row = item as Record<string, unknown>;
    return typeof row.rule === "string" &&
      typeof row.message === "string" &&
      ["low", "medium", "high"].includes(String(row.severity));
  });
}

export async function moderatePatch(
  payload: unknown,
  send: (signal: AbortSignal, body: unknown) => Promise<unknown>,
  totalMs = 8_000,
): Promise<ModerationOutcome> {
  const started = Date.now();
  const maxAttempts = 3;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const remaining = totalMs - (Date.now() - started);
    if (remaining <= 0) return { status: "retry_exhausted", findings: [], attempts: attempt - 1 };

    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), remaining);

    try {
      const raw = await send(controller.signal, payload);
      clearTimeout(timer);
      if (!raw || typeof raw !== "object") {
        return { status: "invalid_response", findings: [], attempts: attempt };
      }

      const response = raw as Record<string, unknown>;
      const findings = response.findings;
      if (!isFindings(findings)) {
        return { status: "invalid_response", findings: [], attempts: attempt };
      }
      return {
        status: findings.length === 0 ? "approved" : "flagged",
        findings,
        attempts: attempt,
      };
    } catch (error) {
      clearTimeout(timer);
      const timedOut = error instanceof Error && error.name === "AbortError";
      if (!timedOut || attempt === maxAttempts) {
        return { status: "retry_exhausted", findings: [], attempts: attempt };
      }

      const baseDelay = 150 * 2 ** (attempt - 1);
      const jitter = Math.floor(Math.random() * 100);
      const delay = Math.min(baseDelay + jitter, totalMs - (Date.now() - started));
      if (delay <= 0) return { status: "retry_exhausted", findings: [], attempts: attempt };
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  return { status: "retry_exhausted", findings: [], attempts: maxAttempts };
}
```

The important detail is the deadline check around the delay. Three attempts are a ceiling, not a promise that all three will run. A fixed sleep can consume the entire budget after the first timeout. Jitter also avoids synchronizing a burst of game-patch reviews after a shared dependency recovers.

Do not retry blindly. If the adapter has already streamed a body or triggered a non-repeatable side effect, classify that operation as non-retryable instead of forcing it through this helper.

Bounded means bounded.

## What should the review queue do with each result?

`approved` and `flagged` can continue through the normal review path after schema validation. `retry_exhausted` should remain pending or go to a human review lane, depending on the release policy. `invalid_response` is an integration failure: retain the raw response in restricted diagnostics, increment a separate metric, and keep it out of the moderation decision.

That routing is more useful than hiding everything behind one generic error. It gives a release manager a direct answer to “can this patch proceed?” without pretending that availability and content policy are the same question. The payload should also include a request identifier generated by the caller, so logs from the queue, adapter, and worker can be joined without placing user text in ordinary log lines.

## What should be measured before changing the policy?

Measure timeout rate by operation and payload class, retry success rate, p50 and p95 end-to-end latency, schema rejection rate, queue age, and the fraction sent to human review. Track attempts as a distribution rather than only an average; a rising tail is where a three-attempt policy starts damaging latency. Record response sizes too, since a larger patch can change both processing time and token usage.

Run a fault-injection test with delayed responses, dropped connections, malformed JSON, valid empty findings, and valid non-empty findings. The expected assertions are concrete: no timeout becomes `approved`, no malformed response becomes `flagged`, a retry never exceeds the total deadline, and a valid result is accepted only when every finding has the required fields. Those checks belong in CI before tuning backoff values.

The decision rule is deliberately plain: retry a replay-safe transport timeout inside a total deadline, validate the returned structure, and route uncertainty as uncertainty. Before copying the numbers above, measure your queue latency and patch-size distribution; the architecture should survive a different budget even when the constants do not. A retry loop cannot repair an invalid schema or compensate for a queue that has no human-review capacity.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.rfc-editor.org/rfc/rfc9457
- https://github.com/openai/whisper
