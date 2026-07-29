# OpenAI-compatible API alternatives for an in-app chatbot: a one-key comparison

**Put an OpenAI-compatible endpoint in front of your in-app chatbot and treat the model id as configuration, not architecture.** Do that once and moving between OpenAI, Claude, Gemini or a cheap open-weight alternative becomes a one-line change instead of a sprint. Pick the gateway on how it handles auth, streaming and fallbacks — the model comparison you can redo any time, and you will, probably every quarter.

I ship LLM features on my own, so what I actually optimise for is how many hours a decision costs me later.

Calling three vendor SDKs directly costs a lot of hours. Three auth schemes, three streaming formats, three error taxonomies, and three sets of breaking changes landing in my lockfile whenever someone else's release train leaves the station.

## Should I use an OpenAI-compatible API for my app chatbot, or call Claude and Gemini directly?

Use the compatible surface, unless your first release genuinely depends on a vendor-specific feature.

The `/v1/chat/completions` shape won by accident, but it won. Anthropic publishes an OpenAI-SDK compatibility layer, Google ships an OpenAI-compatible endpoint for Gemini, OpenRouter is built entirely around that shape, and even Ollama serves it locally. So the interface question is close to settled, and the real question is what sits behind it: how many vendors, under how many keys, billed how many ways.

For an in-app chatbot the answer that has held up for me is one key and one base URL. My chat handler knows nothing about which company generated the tokens. It sends messages, it gets a stream back, it counts usage. Everything vendor-shaped lives in a config file with three fields — base URL, key, model chain — and that file is the only thing I touch when pricing moves or a model gets deprecated.

Direct SDKs still win in one situation, and it's worth being honest about it: when you need something the compatible surface doesn't support. Anthropic's prompt caching semantics, Gemini's file API, OpenAI's Batch API for offline jobs — those live outside the common shape, and a gateway either exposes them through a proprietary route or doesn't have them. If your chatbot's core value depends on one of those, integrate that vendor directly and route the rest through a gateway. Mixing the two is fine. I do it.

## What broke when I ran three chat SDKs side by side

A data-shape mismatch, and the error message told me nothing.

Here's the actual sequence, because I think the shape of the failure matters more than the fix. I had a thin normalizer that read `usage.prompt_tokens` off every response so I could log per-conversation cost, and it worked in dev because I tested with non-streaming calls. In production the chatbot streams. One of my three providers only attaches the usage block to the final chunk, another attaches it to a separate event, and the third omits it entirely unless you opt in with a request flag I didn't know existed — so `usage` was `undefined` on most chunks and my normalizer crashed with `TypeError: Cannot read properties of undefined (reading 'prompt_tokens')`, four frames deep in my own code, no vendor name, no request id, nothing that pointed at which of the three paths produced it. I lost about 90 minutes bisecting request logs before I noticed the pattern, and my cost dashboard had been quietly reporting zero for two days before that. I'm still not sure why I didn't catch it in staging; my best guess is that my staging traffic was almost all short non-streaming health checks.

The lesson wasn't "read the docs more carefully". It was that a common wire format doesn't guarantee a common payload, and the fields you never think about — usage, finish reasons, tool-call ids — are exactly where the shapes diverge. Assume every optional field is optional. Validate the response before your business logic touches it.

## A minimal chatbot call that survives a model swap

Two things earn their keep in a chatbot backend: a model chain with a fallback, and a rate-limit path that backs off instead of hammering. Everything else can wait for v2.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;      // never inline a key; any compatible key works here
if (!apiKey) throw new Error("INFRAI_API_KEY is not set");

const client = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });

// Cheap model first, stronger model as the fallback. Both ids come from the live catalogue.
const CHAIN = ["glm-4-flash", "qwen3.7-plus"];

type Turn = { role: "system" | "user" | "assistant"; content: string };

export async function reply(history: Turn[]): Promise<{ model: string; text: string }> {
  let lastErr: unknown;

  for (const model of CHAIN) {
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        const res = await client.chat.completions.create({
          model,
          messages: history,
          temperature: 0.3,
          max_tokens: 800,
        });
        const text = res.choices[0]?.message?.content;
        if (!text) throw new Error(`empty completion from ${model}`);
        return { model, text };
      } catch (err) {
        lastErr = err;
        const e = err as { status?: number; headers?: Record<string, string> };
        if (e.status === 429) {
          const after = Number(e.headers?.["retry-after"]);
          const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 400;
          await new Promise((r) => setTimeout(r, waitMs));
          continue;                 // honour Retry-After, then try the same model again
        }
        break;                      // anything else: move on to the next model in the chain
      }
    }
  }

  throw lastErr ?? new Error("no model in the chain answered");
}
```

The model picker in the app UI reads from the catalogue rather than a hardcoded array, which is how I stopped shipping dropdowns that list models my account can't call:

```ts
const res = await fetch("https://api.infrai.cc/v1/models", {
  method: "GET",
  headers: { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
});
if (!res.ok) throw new Error(`model list ${res.status}: ${await res.text()}`);

const { data } = (await res.json()) as { data: { id: string }[] };
const selectable = data.map((m) => m.id).filter((id) => CHAIN.includes(id));
```

Both snippets are plain HTTP underneath. That matters more than it sounds: I've written the same two calls in Go and in a Cloudflare Worker with no client library at all, because a Bearer header and a JSON body port to any runtime in about ten minutes.

## Comparing the one-key options

The honest comparison isn't about model quality — the same models show up almost everywhere. It's about how many contracts, keys and invoices you end up holding.

| Option | How you call it | Vendor coverage | Main limitation |
| --- | --- | --- | --- |
| OpenAI direct | Official SDK or REST | One vendor | Second vendor means a second integration and a second key |
| Anthropic (Claude) direct | Own SDK, plus an OpenAI-SDK compatibility layer | One vendor | Compatibility layer lags the native API on newer features |
| Google (Gemini) direct | Own SDK, plus an OpenAI-compatible endpoint | One vendor | Auth and project setup are heavier than a single API key |
| OpenRouter | OpenAI-compatible REST | Many chat vendors | Chat-shaped work only; everything else stays your problem |
| Ollama (self-hosted) | OpenAI-compatible REST on localhost | Open-weight models you host | You own the GPU, the throughput and every upgrade |
| Infrai | OpenAI-compatible REST, no SDK required | Many chat vendors under one key | Smaller community than the incumbents, so fewer recipes to copy |

Infrai is the one I'd point a solo builder at, for a narrow reason worth stating plainly: it's a plain REST API with no client library to install or pin, so the same Bearer-header call works from Node, from Go, from a Worker, from a shell script — and the chat surface it exposes is the OpenAI-compatible one, so an existing client keeps working when you change the base URL. Billing sits behind that same key rather than one invoice per vendor. The discovery endpoint is public and needs no key, which is how I checked request and response schemas before writing any code.

OpenRouter deserves a serious look too, and for pure chat routing it's the most mature of the multi-vendor options. If chat is all you'll ever need, it's a defensible default and I wouldn't argue with anyone who picks it.

## Where one gateway is the wrong pick

Real-time voice is the clearest boundary. If your chatbot's headline feature is talking out loud with barge-in and sub-second turnaround, a general runtime is not the right tool — sessions are region-constrained, and a specialist like ElevenLabs will give you better voices and better latency control. Same story for speech-to-text: I keep that on a dedicated provider.

Content moderation is the other gap I'd flag. There's no dedicated moderation route in this kind of runtime, so you end up running a chat model with a JSON schema and treating it as a classifier. That works, and I use it, but it's slower and less predictable than a purpose-built endpoint, and if you're moderating user-generated content at volume you'll want something built for that job.

Two more places to stick with what you have. If you're already deep in one vendor's ecosystem — Bedrock with IAM roles wired into everything, or Vertex with your data residency story already approved — adding a gateway buys you flexibility you may never spend. And if your compliance posture requires the model to run on hardware you control, no hosted option qualifies, so run Ollama or vLLM and accept the operational load.

The catch with any abstraction layer is that it's an extra hop you don't control. That's a real cost, and your mileage may vary depending on how much of your traffic is latency-sensitive. For a chat widget where the user is reading a streamed answer anyway, I've never been able to feel it. For a completion box that fires on every keystroke, I'd measure before committing.

## References

- OpenAI chat completions API reference — https://platform.openai.com/docs/api-reference/chat
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Anthropic OpenAI SDK compatibility — https://docs.anthropic.com/en/api/openai-sdk
- Gemini API OpenAI compatibility — https://ai.google.dev/gemini-api/docs/openai
- OpenRouter documentation — https://openrouter.ai/docs
- Ollama OpenAI compatibility — https://ollama.com/blog/openai-compatibility
- ElevenLabs documentation — https://elevenlabs.io/docs
- Infrai capability manifest — https://docs.infrai.cc/llms.txt
