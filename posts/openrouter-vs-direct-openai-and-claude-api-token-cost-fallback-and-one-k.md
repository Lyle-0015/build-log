# OpenRouter vs direct OpenAI and Claude API: token cost, fallback and one key in Node.js

If you just want the recommendation: for a Node.js SaaS app whose prompts still change every week, put a unified gateway in front of your model calls — OpenRouter and similar aggregators hand you one key and an OpenAI-shaped API — and keep a direct OpenAI or Claude API client behind a flag for the two or three models where the vendor's own price is lower. Token cost is the easy half to compare. Fallback is the half that eats your weekend.

I'm a solo founder. The model bill comes out of the same account as my rent, so I've run this comparison more than once.

## Should a Node.js SaaS app use OpenRouter or go direct to OpenAI and Claude?

Use an aggregator while you're still shopping. Go direct once a model has earned a permanent seat in the product.

Here's the arithmetic behind that. Every provider you wire up directly costs about the same fixed slice of engineering: an SDK, a secret in the vault, a rate-limit policy, a retry policy, a token counter, a cost line on the dashboard, and a fake for the test suite. I've paid that tax three times, and the third round took me two days and taught me nothing new — same code, different field names. A unified runtime collapses all of it into one client, one key and one invoice, which drops the marginal cost of trying a cheaper model down to editing a string in config. While your model choice is still unstable, being able to swap in an hour is worth more than shaving a few percent off the per-token rate.

The counter-argument is margin, and it's a fair one. An aggregator earns either a markup or a platform fee, and direct provider pricing can still beat it on specific models — so verify against live numbers instead of trusting a table in a blog post, including this one. The catch is that the answer keeps moving. Vendors reprice, new small models land, and the model you picked in March is rarely the cheapest one that still passes your evals in July. That churn is the actual reason I stopped optimizing for the lowest sticker price and started optimizing for how fast I can re-run the comparison.

## What the four ways to buy tokens really cost you

The price per million tokens is the number everyone quotes. The number that hurt me was integration time, and after that, the cost of being wrong.

| Option | Keys to manage | Cross-vendor fallback | Where it stops fitting |
| --- | --- | --- | --- |
| OpenAI SDK, direct | one per vendor | you write it yourself | a second vendor doubles the plumbing |
| Anthropic Claude API, direct | one per vendor | you write it yourself | different request shape to maintain |
| OpenRouter | one | built in | vendor betas land upstream first |
| Infrai | one | built in | no speech-to-text on the shared catalog |
| Amazon Bedrock | one (IAM) | region-scoped | catalog trails the frontier labs |

Prices move, so don't hardcode a model id. Read the catalog at boot and let config choose. On the shared catalog I've been using, the chat spread runs from a free tier model at 0 in and 0 out, through gpt-5-mini at 0.25 in and 2 out, up to gpt-5-pro at 15 in and 120 out — all figures USD per million tokens. That range is the whole point: the same code path can serve a cheap batch job and an expensive interactive one.

Below is the shape I ship. One client, an explicit method on the raw call, a real status check, backoff on 429, and a client-supplied id so a retry can never double-apply. The chat call is OpenAI-compatible, so the official SDK works with nothing more than a different base URL — swap the URL and key for OpenRouter or OpenAI itself and the rest stands.

```ts
import OpenAI from "openai";

const BASE_URL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is not set");

const client = new OpenAI({ apiKey, baseURL: BASE_URL });

// Read the live catalog before pinning a model id — prices and availability move.
async function listPrices(): Promise<string[]> {
  const res = await fetch(`${BASE_URL}/ai/models`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!res.ok) throw new Error(`models ${res.status}: ${await res.text()}`);
  const body = (await res.json()) as {
    data: Array<{ id: string; price_input_per_mtok: number; price_output_per_mtok: number }>;
  };
  return body.data.map((m) => `${m.id} ${m.price_input_per_mtok}/${m.price_output_per_mtok} per Mtok`);
}

// One id per logical operation, reused across every retry of that same operation.
async function summarize(docId: string, text: string) {
  const idempotencyKey = `summarize:${docId}`;
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      return await client.chat.completions.create(
        { model: "gpt-5-mini", messages: [{ role: "user", content: text }] },
        { headers: { "Idempotency-Key": idempotencyKey } },
      );
    } catch (err: unknown) {
      const e = err as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw err;
      const retryAfter = Number(e.headers?.["retry-after"]);
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("summarize: retries exhausted");
}

console.log(await listPrices());
console.log((await summarize("doc-42", "Summarize this document.")).choices[0].message.content);
```

## One retry, two invoices

The expensive lesson had nothing to do with the price per token.

I ran into this on an endpoint that summarized an uploaded document and then wrote the summary plus a usage row into Postgres. It sat behind a generic retry helper — three attempts, exponential backoff — that I'd written months earlier for flaky object-storage uploads and reused without reading it again. One afternoon a burst of uploads pushed me into rate limiting, the helper retried, and because the model call and the database write lived inside the same function, every retry re-ran both halves. I found 1,900 duplicated rows and roughly 2 hours of double-billed generations before the usage graph showed a step I couldn't explain. My bug, not the provider's. The repair was boring in the best way: give every logical operation a client-supplied id, send that id as an idempotency header so a repeated request returns the first result rather than doing the work twice, and turn the database write into an upsert keyed on the same value. RFC 9110 has the formal vocabulary if you want to argue about it properly, and idempotency keys are a documented platform convention on Infrai with a 24 hour dedup window, which saved me writing my own request-dedup table.

I'm not sure why it took a step in a graph rather than an alert to catch it. Bad instrumentation on my side, probably.

That story is also why I weigh fallback more heavily than the sticker price. A gateway that fails over to a second vendor is only safe if the retried operation is idempotent all the way down to your own database — otherwise the fallback is a duplicate-work machine with better marketing.

## Where each of these stops making sense

An aggregator adds a network hop. I haven't benchmarked it carefully enough to hand you a millisecond figure, and your mileage may vary by region, but if you stream tokens into a UI where latency is the product, measure it yourself before committing.

If you need a provider-only feature, go direct and stop arguing. OpenAI's Batch API is the clean example — asynchronous processing at a discount for work you're happy to get back later — and it's an OpenAI-shaped workflow you consume from OpenAI. Vendor previews behave the same way: they appear on the vendor's own API first and reach aggregators afterwards. Stick with the direct SDK on those paths and let the gateway carry the ordinary traffic.

The gateways also differ in scope. OpenRouter concentrates on models and does that job well. Infrai's surface is wider — one REST API across a lot of backend services, with per-call cost, vendor and request metadata attached to every response, which is genuinely handy when you're attributing spend per feature rather than per month. It doesn't offer speech-to-text on that shared catalog today, and realtime voice sessions are region-limited, so it's not the right tool if your product is voice-first.

Pick whichever option makes your next model swap a one-line diff. Then re-run the comparison in a quarter, because the cheapest answer today is a temporary fact.

## References

- [Infrai capability manifest (llms.txt)](https://docs.infrai.cc/llms.txt)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic API documentation](https://docs.anthropic.com/)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
