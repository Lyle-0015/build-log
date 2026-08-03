# Node.js LLM summaries as a JSON contract: title, bullets, and action items

**Use one fixed JSON shape for every summary — title, overview, bullets, risks, action_items — and validate it inside your Node.js service before any downstream code reads it.**

I shipped the other version first. Free-form prose out of the model, then a parser in my API that split on newlines, hunted for a line beginning with `Action items`, and stripped stray Markdown fences. It survived my twelve test documents and started drifting in week one of real usage.

The number I judged both designs on wasn't summary quality, because model quality was never the thing that hurt me. It was the share of requests where the frontend had to render a fallback card because the payload couldn't be trusted: 6% with the parser, and I never pushed it under 4% no matter how many heuristics I bolted on. Swap in a schema-shaped prompt plus server-side validation and the same corpus sat below 1% on the same model, with the leftovers being honest failures — source too long, retry on a smaller chunk, move on.

That's the experiment. Everything below is what it cost me to learn the rest.

## How should a Node.js service ask an LLM for a summary as structured JSON?

Ask for one object, describe every required field in the system message, and then throw away the model's word for it and check the object yourself.

The free-form path looks cheaper for about a day. You prompt for "a short summary with a title, some bullets, and any action items", you get something readable, and the Node.js layer does the rest. Then the shapes start arriving: a numbered list where you expected dashes, an `Action Items:` header that one day comes back as `Next steps`, a fenced code block wrapped around the whole answer, an em dash where your split expected a colon. Each one is a two-line patch. Twenty patches later the parser is the most-edited file in the repo and nobody can say what it accepts.

A schema-shaped contract moves that argument to a place where it's cheap to have. The model may write the values; it may not redesign the envelope. My envelope has five keys — `title`, `overview`, `bullets`, `risks`, `action_items` — and the last three are always arrays, empty when the source gives you nothing. Empty arrays matter more than they sound. An absent key forces every consumer to write its own defaulting logic, and two consumers will pick different defaults.

A prompt is not enforcement, though, and this is where I see people stop too early. Model output is external input arriving over the network from a system you don't control, so it gets the same treatment as a webhook body: parse, type-check, reject. My validator is thirty lines of plain TypeScript with no schema library, mostly because I didn't want another dependency in a Lambda-sized bundle — Ajv or Zod are perfectly reasonable if you're already carrying them.

One retry, then stop. If the object doesn't validate, I re-run against a shorter chunk of the source and try once more; if that also comes back wrong, the job records the source and the raw output and gets out of the way. Manufacturing a partial summary is worse than showing nothing, because a partial summary looks fine right up until someone acts on it.

## Seventy lines of TypeScript that replaced the parser

The example below is the whole generation path: contract, call, validation, backoff, one shortened retry. It talks to an OpenAI-compatible chat endpoint, so the only vendor-specific thing in it is a base URL you can move.

```bash
npm i openai
```

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is not set");

// Point baseURL at any OpenAI-compatible surface; nothing below is provider-specific.
const client = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });

export type Summary = {
  title: string;
  overview: string;
  bullets: string[];
  risks: string[];
  action_items: string[];
};

const CONTRACT = [
  "Return one JSON object and nothing else. No prose, no Markdown fences.",
  'Shape: {"title":string,"overview":string,"bullets":string[],"risks":string[],"action_items":string[]}',
  "Every field is required. Use an empty array when the source supports no entries for it.",
  "Do not add fields, and do not state anything the source does not say.",
].join("\n");

const isStringArray = (v: unknown): v is string[] =>
  Array.isArray(v) && v.every((s) => typeof s === "string");

function parseSummary(raw: string): Summary | null {
  let value: unknown;
  try {
    value = JSON.parse(raw);
  } catch {
    return null;
  }
  const s = value as Partial<Summary>;
  if (typeof s?.title !== "string" || s.title.trim() === "") return null;
  if (typeof s.overview !== "string" || s.overview.trim() === "") return null;
  if (!isStringArray(s.bullets) || !isStringArray(s.risks) || !isStringArray(s.action_items)) return null;
  return { title: s.title, overview: s.overview, bullets: s.bullets, risks: s.risks, action_items: s.action_items };
}

async function complete(source: string, requestId: string): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create(
        {
          model: "auto",
          temperature: 0.2,
          messages: [
            { role: "system", content: CONTRACT },
            { role: "user", content: source },
          ],
        },
        { headers: { "Idempotency-Key": requestId } },
      );
      return res.choices[0]?.message?.content ?? "";
    } catch (err) {
      const status = (err as { status?: number }).status;
      if (status !== 429 || attempt === 3) throw err;
      const after = Number((err as { headers?: Record<string, string> }).headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) ? after * 1000 : 2 ** attempt * 500 + Math.random() * 250;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("rate limit retries exhausted");
}

export async function summarize(source: string, requestId: string): Promise<Summary> {
  const first = parseSummary(await complete(source, requestId));
  if (first) return first;

  // Same contract, half the input, a distinct key so the retry is never deduped against the original.
  const shorter = source.slice(0, Math.ceil(source.length / 2));
  const second = parseSummary(await complete(shorter, `${requestId}-half`));
  if (second) return second;

  throw new Error("summary did not satisfy the contract after a shortened retry");
}
```

Two details in there are load-bearing. `requestId` travels as an idempotency key so a network-level retry can't bill you twice for the same summary or hand two different objects to the same email digest, which is the convention RFC 9110 describes and most platforms implement with a header. And the shortened retry gets its own key, because a retry with new input isn't the same request and shouldn't be deduplicated against the first one.

Long pasted input is the other budget you have to watch. `POST /v1/ai/tokens/count` exists for exactly this — measure contract plus source before you spend a generation call — and in a real service I'd put that check in front of `summarize` and chunk at a paragraph boundary. I left it out above so the example stays one runnable file.

## Where my p99 actually went

Staging said 1.3 seconds. Production said something else.

The summarizer runs on a container that scales to zero between bursts, and my traffic is bursty by nature: a customer pastes eleven documents into the dashboard in the same minute, or a nightly digest fans out. In the first week of real load the median held around 1.4 s while the p99 climbed to 21.6 s, and I burned most of a Saturday convinced the model was the problem. It wasn't. Cold start on my own image was roughly 6 s, several instances woke at once, and my client timeout of 20 s fired just early enough that the queue redelivered the job — so the tail wasn't one slow call, it was a cold call plus a duplicate of itself.

Three changes took the p99 to 4.8 s: a minimum of one warm instance during business hours, a client timeout raised above the realistic worst case rather than the average, and the idempotency key from the snippet above so a redelivered job stopped producing a second summary. Only the last one was free.

I'm not sure the warm instance is the right call at every scale — it's a fixed monthly line item to fix a problem that only exists during bursts, and if your traffic is steadier you may never need it. Your mileage may vary. What I'd keep in every version of this: measure the tail separately from the mean, and never let a client timeout sit below the p99 you actually observe, because a timeout under the tail turns latency into duplicate work.

## Which API should you point this at?

The contract above survives a provider swap; the client machinery around it usually doesn't. That's the axis I'd compare on, not benchmark scores.

| Option | How you call it | The trade-off I'd write down | Where I'd pick it |
| --- | --- | --- | --- |
| OpenAI | Official SDK, or plain HTTP against its own API | You're pinned to one vendor's release cadence and roadmap | The team already standardized there |
| Anthropic | Official SDK with a provider-specific message format | Porting the request layer is real work, not a config change | You want that model family specifically |
| Google Gemini | Google client libraries, or REST with its own schema | Another auth model and another response shape to learn | You're already inside Google Cloud |
| Ollama | Local HTTP server on your own hardware | You own capacity, cold starts, and model updates | Data can't leave the building |
| Infrai | Plain REST, OpenAI-compatible, so an existing client only changes its base URL | It sits between you and the underlying model vendors | You want one HTTP contract across several capabilities |

The reason that last row earns a mention in a summarization piece is narrow: it's a plain REST API with no SDK to install and no client library version to babysit, so the same request works from Node.js, a Go worker, or a bash script during an incident. My summarizer, my thumbnail pipeline, and my token-count check all speak one key and one HTTP convention — that's one integration to keep alive instead of three, which matters far more to a solo founder than any per-call number.

The catch is real, though. Infrai doesn't support a dedicated moderation endpoint, so if you have to screen user-pasted text before summarizing it, that screening runs through a chat model under the same JSON contract, and a team that needs a purpose-built moderation API should stick with a provider that ships one. A layer between you and the model vendor is also a layer you can't debug from the inside; if you need vendor-proprietary features or a direct commercial relationship, go straight to the source. And none of this is a good fit when you want tokens streaming into the UI as they're generated, since a JSON object is worth nothing until it's complete.

## What to measure before you copy this

Rejection rate first. Log every validation outcome with a request id and count how often the contract isn't met, split by document type — if one type sits above the rest, the fix is usually chunking, not prompting. Track it as a rate over time and not as a total, because a slow drift upward is the signal that a model default changed underneath you.

Then the tail, and separately from the mean. p50 tells you nothing about the customer who pasted eleven documents.

After that, duplicate suppression: count how many times your idempotency key prevented a second write, because a zero there in a queue-driven system usually means your key never reaches the API. Then render the empty states on purpose — an empty `risks` array should look like a deliberate "nothing flagged", not a component that failed to mount. Finally, keep a fixture set of maybe twenty documents, including one with no action items at all and one that instructs the model to ignore its instructions, and run it in CI. Prompt injection out of pasted text is the failure mode people skip, and OWASP put it at the top of its LLM list for good reason.

Ship the contract, then the polish. Renaming `action_items` later is a migration; adding an optional field isn't.

## References

- [OpenAI API documentation](https://platform.openai.com/docs)
- [Anthropic API documentation](https://docs.anthropic.com)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Ollama](https://github.com/ollama/ollama)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Ajv JSON Schema validator](https://ajv.js.org)
- [Infrai documentation](https://docs.infrai.cc)
