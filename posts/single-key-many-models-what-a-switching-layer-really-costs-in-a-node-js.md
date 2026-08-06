# Single key, many models: what a switching layer really costs in a Node.js/Express API

**Short answer:** keep one HTTP request shape in front of every model your Node.js/Express backend calls, hold the model id in config behind a single key, and budget more time for token accounting than for the switch itself. Swapping the model is an afternoon. The bookkeeping around it took me three weeks.

I ship features alone, so every abstraction I add has to pay rent. When the third customer asked for a different model on their workspace, my first instinct was two `if` branches and a deploy. That instinct was wrong, though not for the reason I expected — the model quality argument turned out to be the easy half.

The hard half was a missing field.

## What breaks first when one Node.js backend has to switch models behind a single key?

Not the request. The request shape converges.

A chat body — a messages array, `temperature`, `max_tokens`, a model string — is close enough across providers that one `POST /v1/chat/completions` covers the ordinary chat, summarize and extract workloads. OpenAI, Claude and Gemini class models all answer to that shape through one gateway or another, so in your Express handler the vendor collapses into a string you read from an env var or a database column. That part is genuinely a day of work, and anyone telling you the switching layer is hard is selling something.

What diverges is everything wrapped around the answer.

Usage accounting is the worst of it. Token counts arrive in different places depending on whether you stream, some providers count reasoning tokens separately, and a streamed response may carry no usage payload at all unless you ask for one explicitly. Tool calling diverges next: the arguments come back as a JSON string that you have to parse yourself, the naming of the container object differs across families, and a model that's excellent at prose can be sloppy about emitting a schema your validator accepts. Error envelopes are the third: a 429 with a `Retry-After` header from one endpoint, a 429 with nothing useful from another, a context-length overflow that shows up as a 400 here and as a quietly truncated completion there. None of that is a defect in anyone's API — they're independent products that converged on a request format and never had a reason to converge on the rest. Your backend is where the difference has to be absorbed, and if you don't absorb it deliberately it gets absorbed by your metrics instead.

So the design question isn't which model to pick. It's which layer eats the inconsistency, and how loudly it complains when it can't.

## From one request to a cost ledger: the Express handler I actually run

The flow is deliberately boring. A client posts to my own route with a tier name rather than a model id, my handler maps that tier to a model string and a token cap, it calls one upstream base URL with one key, it pipes the stream straight through to the browser as server-sent events, and on the way out it writes a usage row keyed by tenant. The upstream never sees a session cookie. The browser never sees a model id it could tamper with.

```ts
import express from "express";

const app = express();
app.use(express.json());

const BASE_URL = process.env.LLM_BASE_URL!;   // anything that speaks the chat-completions shape
const API_KEY = process.env.LLM_API_KEY!;

// Tiers, not model ids: the client asks for "cheap", ops decides what that means today.
const TIERS: Record<string, { model: string; maxOut: number }> = {
  cheap: { model: process.env.MODEL_CHEAP!, maxOut: 512 },
  smart: { model: process.env.MODEL_SMART!, maxOut: 2048 },
};

app.post("/api/answer", async (req, res) => {
  const tier = TIERS[req.body.tier] ?? TIERS.cheap;

  const upstream = await fetch(`${BASE_URL}/chat/completions`, {
    method: "POST",
    headers: { authorization: `Bearer ${API_KEY}`, "content-type": "application/json" },
    body: JSON.stringify({
      model: tier.model,
      messages: req.body.messages,
      max_tokens: tier.maxOut,
      stream: true,
      // Ask for usage on the final chunk. Omit this and the ledger below records nothing.
      stream_options: { include_usage: true },
    }),
    signal: AbortSignal.timeout(30_000),
  });

  if (upstream.status === 429) {
    const wait = Number(upstream.headers.get("retry-after") ?? 1);
    res.setHeader("retry-after", String(wait));
    return res.status(429).json({ error: "rate_limited", retry_after: wait });
  }

  res.setHeader("content-type", "text/event-stream");
  let usage: { total_tokens: number } | undefined;

  for await (const line of lines(upstream.body!)) {
    if (!line.startsWith("data: ")) continue;
    const payload = line.slice(6);
    if (payload === "[DONE]") break;
    const chunk = JSON.parse(payload);
    if (chunk.usage) usage = chunk.usage;                 // may arrive only on the last chunk
    res.write(`data: ${payload}\n\n`);
  }

  // Fail loud: a request with no token count is an accounting hole, not a rounding error.
  if (!usage) throw new Error(`no usage for tier=${req.body.tier} model=${tier.model}`);
  await recordUsage(req.body.tenantId, tier.model, usage.total_tokens);
  res.end();
});
```

Two lines in there are the whole article: `stream_options`, and the throw. I'll explain why the second one exists.

## The field I assumed was there, and the three weeks it cost

My first version wrote `usage?.total_tokens ?? 0` into the ledger, because optional chaining is what you type when you're moving fast and the shape looks obvious. Every request returned 200. Latency held around 800 ms. Nothing paged, nothing retried, no error appeared in any log, and the per-tenant cost page rendered a clean, confident, entirely fictional zero for six days. I only caught it because a tenant I knew was hammering the smart tier showed up with the same spend as a tenant who'd churned. When I finally deleted the `?? 0` to see what was underneath, the runtime told me `TypeError: Cannot read properties of undefined (reading 'total_tokens')` — which is technically accurate and operationally worthless, since it names neither the chunk, nor the tier, nor the fact that the field is opt-in when streaming. I'd assumed a field was there because a non-streaming response had it. Two hours of staring at raw SSE frames later I found the opt-in flag, and then spent the rest of that sprint rebuilding six days of attribution from request logs I was lucky to still have. The switching layer had worked perfectly the entire time. That's what made it dangerous.

Since then I treat any per-request number I can't see in a test as unmonitored, and I write a failing assertion for it before I write the feature.

Here's how the options actually compare once you price in that kind of work rather than just the integration:

| Approach | Cost to swap a model | Auth surface | What you give up |
| --- | --- | --- | --- |
| One chat-completions contract, model in config | Config change, no deploy | One key | Vendor-specific extras get filed off |
| One SDK per vendor family | New adapter, new tests | One scheme per vendor | Weeks, mostly in the adapter |
| Proxy you host yourself | Config change | One key, plus an ops burden | A service you now page for |
| Direct to a vendor for one capability | Not swappable by design | Its own scheme | Portability, on purpose |

The catch is that the first row only pays off while your workload stays inside the common subset. If you need a vendor's prompt-caching semantics, a batch lane, or a long-context video mode, a shared contract doesn't support any of them, and the honest trade-off is to stick with a direct integration for that one path while the generic route carries the other 90%. Same answer when a compliance document names the processor by hand. A lowest-common-denominator surface earns its keep on volume, not on your most exotic call. And if you're self-hosting a local runtime such as Ollama for development, the same request shape usually covers it, which is a nice property to have for free.

## Before you ship the switch: the checks that earn their keep

Put the model id in exactly one place and read it back off the response instead of trusting what you sent, because a gateway that silently falls back to another model is indistinguishable from success at the HTTP layer. Assert on token counts in a test that runs against the real endpoint, not a mock — mocks return the field you assumed exists, which is precisely the failure this catches. Keep a small replay harness of 30 to 50 prompts with graded expectations so that changing a tier is a measurement rather than a vibe; mine is a script and a CSV, and that's enough. Cap `max_tokens` per tier rather than globally, since the cheap tier's whole point is a bounded worst case. Handle 429 by surfacing it, never by swallowing it into a retry loop that turns a visible limit into a mysterious latency spike. And if you're storing embeddings alongside all this, a Postgres extension like pgvector keeps the retrieval side on infrastructure you already back up, which is one fewer vendor in the blast radius.

As far as I can tell there's no configuration that makes the metadata problem disappear — you either budget for it in your own code or discover it in your billing. I'd rather budget.

## References

- [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [Chat completions API reference (streaming options and usage)](https://platform.openai.com/docs/api-reference/chat/create)
- [MDN: using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static)
- [pgvector, the Postgres vector similarity extension](https://github.com/pgvector/pgvector)
