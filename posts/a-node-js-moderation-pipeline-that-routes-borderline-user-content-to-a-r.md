# A Node.js moderation pipeline that routes borderline user content to a review queue

## TL;DR

Send each item to a cheap chat model with a strict JSON schema, then branch on the verdict: publish, block, or park it in a review queue for a human. The queue is the part that actually matters — the rest of the design exists so every piece of user content lands in exactly one bucket and never falls out of the pipeline unnoticed. One table, one worker, roughly 80 lines of TypeScript.

I ship a small product on my own: profiles, comments, a public feed. Moderation moved to the top of my roadmap the week a spam ring found the signup form.

A feed of user generated content becomes a liability the moment strangers can post to it. Mine did.

The first version took an afternoon. The second took three weeks, because the first one lied to me about what it had processed.

Most write-ups on this topic stop at the model call, which is the easy 10%. The prompt is nearly a solved problem — any mini-tier model will label obvious spam and obvious harassment correctly on the first try. What decides whether your setup survives contact with real users is the backend plumbing around it: what happens on a timeout, what happens when the same job gets delivered twice, and who looks at the pile of stuff the model wasn't sure about.

## How does a review queue fit into a Node.js moderation pipeline for user content?

It sits between the model and your database, and it exists because a classifier that only says yes or no forces you to be wrong in one of two expensive directions. Block too eagerly and you burn real users. Publish too eagerly and you're the spam host. A third bucket lets the machine punt, which is the only honest answer for the 5–8% of items that sit near the line.

So the model returns three actions, not two. Here's the classifier I run in production, trimmed of logging:

```ts
// classify.ts — one verdict per item, Node 20+, zero deps
const KEY = process.env.OPENAI_API_KEY;
if (!KEY) throw new Error("OPENAI_API_KEY is empty");

export type Verdict = {
  action: "publish" | "review" | "block";
  category: "none" | "spam" | "harassment" | "hate" | "sexual" | "violence";
  confidence: number;
};

export class Retryable extends Error {
  constructor(message: string, readonly retryAfterSec: number) {
    super(message);
  }
}

const verdictSchema = {
  name: "verdict",
  strict: true,
  schema: {
    type: "object",
    properties: {
      action: { type: "string", enum: ["publish", "review", "block"] },
      category: {
        type: "string",
        enum: ["none", "spam", "harassment", "hate", "sexual", "violence"],
      },
      confidence: { type: "number" },
    },
    required: ["action", "category", "confidence"],
    additionalProperties: false,
  },
};

const POLICY = [
  "You moderate comments on a developer community.",
  "block = unambiguous spam, harassment, hate, sexual or violent content.",
  "review = you are unsure, or the text is borderline under the policy.",
  "publish = clearly fine.",
  "When unsure, choose review. Do not guess.",
].join(" ");

export async function classify(text: string): Promise<Verdict> {
  const res = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "gpt-5-mini",
      messages: [
        { role: "system", content: POLICY },
        { role: "user", content: text.slice(0, 4000) },
      ],
      response_format: { type: "json_schema", json_schema: verdictSchema },
    }),
  });

  // 429 and 5xx are transient. They are NOT a verdict, and they are not a publish.
  if (res.status === 429 || res.status >= 500) {
    const after = Number(res.headers.get("retry-after")) || 0;
    throw new Retryable(`upstream ${res.status}`, after);
  }
  if (!res.ok) throw new Error(`classify ${res.status}: ${await res.text()}`);

  const body = await res.json();
  return JSON.parse(body.choices[0].message.content) as Verdict;
}
```

Two things in there are deliberate. The policy tells the model to pick `review` when unsure, because a model that's been told "decide" will always decide, and a forced coin flip on borderline text is worse than an honest shrug. And the strict JSON schema means I parse a known shape instead of writing defensive code around prose — with `strict: true` the enum is enforced by the provider, so an unexpected action string can't reach my switch statement.

The confidence number is the one field I'd call optional. Self-reported confidence from an LLM is soft, and I use it for sorting the human queue by nastiness rather than for routing. Routing runs off `action` alone.

## The three-status contract that keeps the queue honest

Every item in my `comments` table carries a `mod_status` of `pending`, `published`, `blocked`, or `review`, plus the model's category and its reason string. Three rules govern transitions, and I'd write all three again on any pipeline I build.

Nothing is visible while `pending`. New content is invisible until a verdict exists, which turns the whole moderation question into "how fast is the queue" instead of "how bad is the leak." My p95 sits around 900ms, so nobody notices.

Every write is keyed on the item id, never appended. Queues redeliver — SQS, BullMQ, Cloud Tasks, all of them are at-least-once, and your consumer has to assume a job can arrive twice. An `UPDATE ... WHERE id = $1 AND mod_status = 'pending'` makes the second delivery a no-op instead of a second verdict row and a second webhook to your moderator Slack channel.

Exhausted retries route to `review`, never to `publish`. Fail closed. **A failed classification is not a verdict, and it must never resolve into published content.**

```ts
// worker.ts — the only place mod_status changes
import { classify, Retryable, type Verdict } from "./classify.ts";

const MAX_ATTEMPTS = 5;

export async function handle(job: { id: number; text: string }) {
  let verdict: Verdict | null = null;

  for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
    try {
      verdict = await classify(job.text);
      break;
    } catch (err) {
      const wait =
        err instanceof Retryable && err.retryAfterSec > 0
          ? err.retryAfterSec * 1000
          : Math.min(30_000, 2 ** attempt * 500) + Math.random() * 250;
      if (attempt === MAX_ATTEMPTS) break;
      await new Promise((r) => setTimeout(r, wait));
    }
  }

  // no verdict after N attempts => a human decides, and the counter records it
  const final = verdict ?? { action: "review", category: "none", confidence: 0 };
  metrics.increment("moderation.verdict", { action: final.action, degraded: String(!verdict) });

  await db.query(
    `UPDATE comments
        SET mod_status = $1, mod_category = $2, mod_confidence = $3, moderated_at = now()
      WHERE id = $4 AND mod_status = 'pending'`,
    [final.action === "publish" ? "published" : final.action, final.category, final.confidence, job.id],
  );
}
```

Observability is two counters and one alert, and it costs about twenty minutes to wire up. Count verdicts by action, count degraded verdicts (the ones that hit the fallback), and alert when the `review` rate doubles against its weekly baseline. A `review` spike is your early warning for both a coordinated spam wave and a broken prompt, which is a lot of signal for one threshold. Testing is similarly cheap: I keep 40 fixture comments — 20 obviously fine, 20 obviously not, hand-labelled — and CI asserts the classifier still routes them correctly before a prompt edit or a model swap ships. That suite has caught exactly one regression in a year, and it was the one that mattered: a prompt tweak that made the model far more eager to block, which I'd have discovered from angry users instead of from a red build. Deployment-wise, the worker holds no state, so I run two of them, and anything that fails past the retry ceiling gets a row in the queue rather than a page at 3am. The moderator UI is a table with three buttons. It doesn't need to be more than that until you have more than one moderator.

## The 429 that my retry loop ate

Here's the failure that produced the design above, and I still find it a little embarrassing.

A newsletter linked to our feed, and we took about 6,000 comments in twenty minutes — maybe 30x normal. My worker pool ran 12 concurrent jobs, which was fine on a normal Tuesday and immediately blew through the account's per-minute request limit. The API started returning 429 with `retry-after: 22`. My retry helper at the time did three attempts at 200ms, 400ms and 800ms, so all of my retries — 1.4 seconds of them, total — landed inside the same rate-limit window that had just rejected me. Every one failed. And the `catch` at the bottom returned `{ action: "publish", confidence: 0 }` as a "safe default," because when I wrote that line I was thinking about a stuck queue, not about a leak. The processing dashboard read 100% the entire time, since I was counting fulfilled promises rather than actual verdicts. So there was no alert, no error spike, nothing to look at. Roughly 2,900 comments went straight to the feed unmoderated over about 40 minutes, and I found out from a user's email, not from my own instrumentation. The fix was three lines: honour `retry-after` when the header is present, raise the ceiling to five attempts with jitter, and send the exhausted case to `review` instead of `publish`. I assumed a moderation delay was worse than a moderation miss. It's the other way around, and it isn't close.

I'm not sure I'd have caught it even with better logging, honestly — the swallowed error never became a log line at all. What actually saved me later was the degraded-verdict counter, which makes "the model didn't answer" a visible, alertable event rather than a silent default.

## Which model or API should do the classifying?

There are four realistic options, and the honest answer is that **the plumbing around the call matters more than which model makes it**.

| Approach | What it costs | Policy fit | Where it breaks |
|---|---|---|---|
| Dedicated moderation endpoint (OpenAI) | free | fixed category list | your own rules — off-topic, scams, ban-evasion |
| Chat model + JSON schema (gpt-5-mini, Claude, Gemini) | per-token, cents per thousand short items | anything you can write down | latency and per-call cost at high volume |
| Self-hosted small model (Ollama, behind LiteLLM) | GPU or CPU time | anything, plus data stays in your VPC | you now operate a model server |
| Specialised scoring API (Perspective) | free with quota | toxicity only, as a 0–1 score | not a general policy engine |

If your policy is close to the standard safety categories, start with OpenAI's moderation endpoint — it's free, it's one call, and paying a chat model to re-derive "is this hate speech" is money set on fire. The catch is the fixed taxonomy. Mine includes "posting affiliate links in a help thread," which no stock category covers, so I run the chat-model path and pay for it.

Among chat models the differences are mostly ergonomic rather than qualitative. OpenAI gives you `response_format` with a JSON schema and `strict: true`. Gemini takes a `responseSchema` alongside `responseMimeType: "application/json"`. With Claude the path I've used is a tool definition — you declare the verdict as a tool's input schema and read the tool input instead of parsing a message, which works fine and is one more concept to hold. All three are one HTTP call behind an adapter, so I keep the provider in an env var and route through LiteLLM in front of them, and switching costs an afternoon instead of a sprint. Vendor lock-in on a classification workload is a self-inflicted wound. The prompt is fifty words and the output is an enum — that's the most portable thing in your codebase.

Two places where I'd tell you not to do any of this. If you're on a regulated dataset that can't leave your infrastructure, stick with a local model through Ollama and accept the ops cost; a hosted API is not a good fit no matter how the latency compares. And if you're moderating tens of millions of items a day, per-item LLM calls stop making sense economically — cascade instead, with a cheap embedding or hash lookup catching known-bad content first and the model only seeing what survives. The embeddings API is genuinely good for the near-duplicate case, since spam rings post the same text 400 times with different usernames, and one nearest-neighbour lookup kills the batch for a fraction of a classification call.

One more thing I'd skip on day one: an appeals flow. Log the reason string alongside every block so you can reconstruct a decision months later, and handle appeals by hand until the volume actually hurts. As far as I can tell in 2026, nobody has been sunk by a slow appeals process — plenty have been sunk by an unmoderated feed.

## References

- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- OpenAI structured outputs (JSON schema in chat completions) — https://platform.openai.com/docs/guides/structured-outputs
- OpenAI rate limits and the `retry-after` header — https://platform.openai.com/docs/guides/rate-limits
- OpenAI embeddings guide (near-duplicate spam clustering) — https://platform.openai.com/docs/guides/embeddings
- Gemini structured output — https://ai.google.dev/gemini-api/docs/structured-output
- Anthropic tool use (pinning Claude's output shape) — https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- LiteLLM, open-source gateway across providers — https://github.com/BerriAI/litellm
- Perspective API — https://developers.perspectiveapi.com
