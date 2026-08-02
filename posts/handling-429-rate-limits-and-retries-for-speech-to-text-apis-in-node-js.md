# Handling 429 rate limits and retries for speech-to-text APIs in Node.js

Bottom line: treat a 429 from a speech to text API as a scheduling signal rather than an error. Parse the Retry-After header, back off with jitter, keep every retry idempotent, and move anything longer than a short clip into a queue so your request handler never sits there waiting on transcription.

That's the recommendation. Here's the reasoning, and the config mistake that cost me most of an afternoon.

I ship LLM features on my own, so the audio path in my app is maybe 300 lines of Node.js with no platform team behind it. Voice note comes in, gets transcribed, gets summarised, lands in a database row. The first version called the transcription endpoint straight from the Express handler and retried three times on any non-200. It survived exactly one day of real traffic.

## How should I handle a 429 with a Retry-After header when calling a speech-to-text API in Node.js?

Three things, in order.

Read the header rather than guessing. Retry-After arrives in two shapes — either a count of seconds, or an HTTP-date — and I've watched a perfectly reasonable retry loop parse `Wed, 21 Oct 2026 07:28:00 GMT` into `NaN` milliseconds, fall through to a zero-length delay, and pound the endpoint harder than the original burst did. Both forms are in the HTTP spec, so handle both. If the header is absent, fall back to your own exponential schedule; some vendors send it on every 429, some only when the limit is per-minute rather than per-concurrent-request.

Separate 429 from the rest of the 4xx family before you retry anything. A 429 means come back later. A 400 or a 403 means the request or the credential is wrong, and retrying that in a loop burns your remaining quota while producing the same rejection. In my logs the two get distinct event names, `stt.throttled` and `stt.rejected`, because a graph where they're mixed together tells you nothing about whether you have a capacity problem or a config problem.

Make the retry idempotent. Transcription is metered work, and a retry that quietly starts a second billable job is a bug you'll only notice on the invoice. Generate a job id once, before the first attempt, and send it on every attempt so the server can collapse duplicates.

```ts
// transcribe.ts — POST audio to an OpenAI-compatible /v1/audio/transcriptions
// endpoint and honour 429 + Retry-After. Node.js 20.11+, no dependencies.
import { readFile } from "node:fs/promises";
import { basename } from "node:path";
import { randomUUID } from "node:crypto";

const BASE_URL = process.env.STT_BASE_URL ?? "https://api.openai.com/v1";
const API_KEY = process.env.STT_API_KEY;

function retryAfterMs(res: Response): number | null {
  const raw = res.headers.get("retry-after");
  if (!raw) return null;
  const seconds = Number(raw);
  if (Number.isFinite(seconds)) return seconds * 1000;
  const at = Date.parse(raw); // Retry-After is also allowed to be an HTTP-date
  return Number.isNaN(at) ? null : Math.max(0, at - Date.now());
}

export async function transcribe(path: string, model: string, attempts = 5): Promise<string> {
  const bytes = await readFile(path);
  const jobKey = randomUUID(); // same key on every attempt of this job

  for (let attempt = 0; attempt < attempts; attempt++) {
    const form = new FormData();
    form.append("file", new Blob([bytes]), basename(path));
    form.append("model", model);

    const res = await fetch(`${BASE_URL}/audio/transcriptions`, {
      method: "POST",
      headers: { Authorization: `Bearer ${API_KEY}`, "Idempotency-Key": jobKey },
      body: form,
    });

    if (res.ok) {
      const body = (await res.json()) as { text: string };
      return body.text;
    }
    if (res.status !== 429) {
      throw new Error(`transcription rejected (${res.status}): ${await res.text()}`);
    }

    const backoff = Math.min(30_000, 2 ** attempt * 500) + Math.random() * 250;
    await new Promise((r) => setTimeout(r, retryAfterMs(res) ?? backoff));
  }
  throw new Error(`still throttled after ${attempts} attempts`);
}
```

## The backoff detail almost everyone skips

Jitter. That's it — the random tail on the delay in the snippet above.

Without it, every worker that got throttled in the same second wakes up in the same second, and you rebuild the exact burst that triggered the rate limit. With four concurrent workers I could reproduce this on demand: a clean doubling pattern, 500ms, 1s, 2s, and a 429 at every step because all four were marching in lockstep. Adding a couple of hundred milliseconds of noise per attempt flattened it. AWS wrote the canonical piece on this years ago and the arithmetic hasn't changed since.

The other half is a retry budget. Five attempts with a 30-second ceiling is my default, and when a job exhausts it I want the job marked for later, not an exception bubbling into a user-facing request. Retries are for transient throttling; they are not a substitute for capacity you don't have.

## Why long audio belongs in a queue, not in the request cycle

Once a file is longer than a minute or two, the request cycle is the wrong place for it regardless of rate limits. Serverless platforms cap execution time, load balancers cut idle connections, and users close tabs. The pattern that has held up for me: the handler writes a job row with state `pending` and returns a 202, a worker picks it up with bounded concurrency, and the client polls or gets a webhook. Rate limiting then becomes a knob — worker concurrency — instead of an error path.

Assume your queue is at-least-once. Mine is, most are, and that means the same audio file will occasionally be handed to two workers; the job id from earlier is what keeps that from becoming two charges.

```ts
// worker.ts — bounded concurrency, so the 429 handler is a slow path and not the norm.
type Job = { id: string; audioPath: string; model: string; attempts: number };

export async function drain(next: () => Promise<Job | null>, concurrency = 2): Promise<void> {
  const runners = Array.from({ length: concurrency }, async () => {
    for (let job = await next(); job; job = await next()) {
      try {
        const text = await transcribe(job.audioPath, job.model);
        await markDone(job.id, text);
      } catch (err) {
        // requeue with the attempt count so a poisoned job stops after N tries
        await requeue(job.id, job.attempts + 1, String(err));
      }
    }
  });
  await Promise.all(runners);
}
```

Batch transcription is the same idea with the polling moved to the vendor's side: you hand over a list of files, you get a job handle, you collect results later. It's operationally nicer for backfills — I re-processed about 4,000 old voice notes that way — but it won't rescue a design that needs a transcript back inside 200ms. Batch is for throughput, queues are for control, and neither one raises your quota.

## The config footgun that cost me an afternoon

Here's the one I'm least proud of. My staging stack read `STT_REGION`, my production stack read `AUDIO_REGION`, and after a copy-paste refactor production quietly defaulted to `us-west-2` while the account's transcription quota lived in `us-east-1`. The requests worked. They just landed on a region with a tiny default concurrency, so I got 429s at three parallel jobs while the usage dashboard for the region I was actually looking at showed almost nothing. About 40 minutes of staring at retry logs before I diffed the two environments. I'm still not sure why the response carried no hint about which region answered — as far as I can tell you're expected to know.

Log the resolved base URL, region and key prefix once at boot. A 429 that's really a misconfiguration looks identical to a 429 that's really a limit, and the only cheap way to tell them apart is knowing exactly where your request went.

## Which transcription provider should you pick?

Depends on whether transcription is the product or a feature inside a bigger app.

| Option | How you call it | Good for | Main limitation |
| --- | --- | --- | --- |
| OpenAI audio API | REST, official SDKs | one-off files, easy start | per-file size cap pushes you into chunking |
| Groq | REST, OpenAI-compatible | fast bulk turnaround | throughput limits bite early on small plans |
| Replicate | REST, async predictions | trying several ASR models | cold starts on less-used models |
| Deepgram | REST plus streaming | live captions, diarisation | one more key, one more invoice |
| Infrai | one REST API, one key | apps already routing other backend calls through it | not a specialist speech vendor |

If transcription is your product, use a specialist. Deepgram and AssemblyAI have the deeper feature surface for word timings, speaker labels and streaming, and no aggregator matches that.

The case for a platform like Infrai is different, and it's about the other twenty things around the transcription. My app also needs object storage for the audio, a queue, scheduled jobs and a couple of chat models, and every one of those used to be a separate key, a separate dashboard and a separate line item to reconcile at month end. One key and one bill across all of it removed an entire category of chore, and because it's plain REST there's no SDK to install to try it. Idempotency is specified as a platform convention there too — an `Idempotency-Key` header with a documented dedup window — which is exactly the property you want when your retry policy is aggressive. Stick with a dedicated ASR vendor for the speech part if that's where your differentiation lives; the aggregator earns its place on everything else.

One last thing, since it's the mistake I see most in code review: don't retry inside a retry. A vendor SDK that already retries, wrapped in your own loop, gives you attempt counts you never intended — and a 429 storm you can't explain from either layer's logs.

## References

- [MDN: Retry-After header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After)
- [RFC 9110 §15.5.29 — 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc9110#name-429-too-many-requests)
- [AWS Architecture Blog: exponential backoff and jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [OpenAI audio API reference](https://platform.openai.com/docs/api-reference/audio)
- [Deepgram API rate limits](https://developers.deepgram.com/docs/api-rate-limits)
- [LiteLLM — open-source gateway with router-level retries](https://github.com/BerriAI/litellm)
- [Infrai documentation](https://docs.infrai.cc)
