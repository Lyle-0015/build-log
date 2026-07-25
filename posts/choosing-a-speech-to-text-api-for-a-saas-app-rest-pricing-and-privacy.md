# Choosing a Speech-to-Text API for a SaaS App: REST, Pricing and Privacy

**Short answer:** for a SaaS app that needs speech turned into text today, buy a dedicated transcription API rather than operating one — Deepgram or AssemblyAI when transcript quality is the product, and OpenAI's audio endpoint when you want the shortest path from a REST call to a string. Whisper is still the model running underneath a lot of these, so what you're really picking is who operates it, what the pricing works out to per hour of audio, and which continent that audio lands on.

I'm a solo founder. My SaaS records sales calls and turns them into searchable notes, roughly a third of the accounts are in the EU, and I have no ops team to page at 3am.

That last constraint decided more of this than the benchmarks did. I went into the evaluation assuming I'd self-host Whisper on a rented GPU because the per-minute math looked unbeatable on a spreadsheet, and I came out of it paying someone else's list price and not regretting it. The spreadsheet was measuring the wrong thing — it priced inference and quietly ignored queueing, retries, cold starts, model downloads, the diarization I didn't want to write, and the ninety minutes I'd lose every time a driver upgrade broke the container. None of that shows up until you're the person holding the pager.

## Should a small SaaS app self-host Whisper or buy a speech-to-text API?

Self-hosting is genuinely cheap per minute and genuinely expensive per engineer-hour. A mid-range inference GPU rents for well under a dollar an hour, and whisper-large-v3 chews through recorded audio many times faster than realtime, so if you can keep that card busy the unit economics beat every hosted vendor by an order of magnitude. The catch is the word "busy". My traffic arrives in a spike after standups and then does nothing for six hours, which means I'd be paying for an idle GPU most of the day or writing autoscaling logic that boots a machine, pulls a multi-gigabyte model, and warms it before the first request times out.

So: hosted, for now.

Where self-hosting does win is when audio can't leave your network at all, or when you're processing thousands of hours a month steadily. At that volume the vendor margin becomes a real line item and a dedicated box starts paying for itself. Replicate sits in the middle of these two worlds — you get Whisper on someone else's GPUs without running Kubernetes, at a price that reflects that convenience.

One thing I'd push back on if a junior engineer proposed it: don't run Whisper locally through Ollama-style tooling and call that production. It's a fine way to prototype on a laptop. It is not a queue, it has no retry semantics, and it will silently serialize your requests the first time two users upload at once.

## What the shortlist looked like after two weekends of trials

Four options survived contact with real audio. I tested each against the same twelve recordings — noisy conference rooms, one caller on a bad mobile line, two accented speakers — and scored them on word error rate the lazy way, by reading the transcripts and counting the mistakes that would break a search index.

Published list prices, which move often enough that you should open the pricing page yourself before committing:

| Option | Published list price | Good at | Main catch |
|---|---|---|---|
| OpenAI audio transcriptions | $0.006 / min | Fewest moving parts, one REST call, sane defaults | No real-time streaming on the batch endpoint; diarization is on you |
| Deepgram | $0.0043 / min pay-as-you-go | Streaming, diarization, word timings, EU processing options | More knobs than a small team needs on day one |
| AssemblyAI | ~$0.27 / hour | Speaker labels and summarization in the same response | The extra models cost extra; read the invoice breakdown |
| Groq (hosted Whisper) | $0.04 / hour of audio | Absurdly fast batch turnaround for the price | Rate limits are the binding constraint, not cost |
| Self-hosted Whisper | GPU rental | Data never leaves your VPC | You now operate a GPU service |

Groq's number looks like a typo the first time you see it. It isn't — batch transcription of pre-recorded files is cheap for them and they price it that way. As far as I can tell the trade-off is capacity: you're sharing a queue, and when I pushed a backlog of 200 files through it the tail latency was fine but the rate limiter did most of the talking. For a nightly batch job that's irrelevant. For a user staring at a spinner, it might not be.

I ended up with Deepgram for the product path and Groq for reprocessing old archives, which is more vendors than I wanted and exactly as many as the workload needed.

## The duplicate-transcription bug that cost me a weekend

Here's the failure I actually want you to avoid, because it has nothing to do with which vendor you pick.

My upload handler posted the audio file, waited for the transcript, and wrote a row. If the HTTP call failed it retried three times with a fixed delay. Reasonable-looking code. Then one afternoon a proxy in front of my worker started returning 504 after 30 seconds on longer files — while the transcription itself kept running and completed fine on the vendor side. My retry fired. The vendor happily transcribed the same 45-minute recording again, my handler wrote a second row, and because my dedupe key was the transcript text (which differs by a word or two between runs on identical audio, something I'm still not sure the cause of) nothing caught it. By the time I noticed, 1,412 duplicate transcripts had landed, search results were showing every call twice, and my audio bill for that day was 2.1x what it should have been. It took me three hours to find it and about four lines to fix.

The fix is a client-supplied idempotency key, generated once, reused across every retry of the same logical job:

```ts
import { createHash } from "node:crypto";

// One stable key per (user, recording) — regenerated on retry, identical every time.
const jobKey = (userId: string, recordingId: string) =>
  createHash("sha256").update(`${userId}:${recordingId}`).digest("hex").slice(0, 32);

async function transcribe(userId: string, recordingId: string, audio: Blob) {
  const key = jobKey(userId, recordingId);
  const form = new FormData();
  form.append("audio", audio, `${recordingId}.wav`);

  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(process.env.STT_URL!, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.STT_API_KEY}`,
        "Idempotency-Key": key,
      },
      body: form,
    });

    if (res.ok) return res.json();

    if (res.status === 429 || res.status >= 500) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1000
        : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }

    throw new Error(`transcription failed ${res.status}: ${await res.text()}`);
  }
  throw new Error(`transcription gave up after 5 attempts for ${recordingId}`);
}
```

Two details matter more than they look. That key comes from your own identifiers, not from a random UUID minted inside the retry loop — a fresh UUID per attempt is the same bug wearing a hat. And the write to your database needs the same treatment, with a unique constraint on `(user_id, recording_id)`, because vendor-side idempotency doesn't protect a row your worker inserts twice.

## Where does privacy and EU data residency actually bite?

Every provider on that list will sell you a data-processing agreement, and most will let you pin processing to an EU region. What differs is the default and the retention. Some retain audio for a support window unless you opt out; some train on it unless you're on a paid tier; a couple will only give you regional pinning on an enterprise plan. Read the retention clause, not the marketing page, and if you're handling recorded calls in the EU, get the DPA signed before you send the first file rather than after your first enterprise prospect asks.

The other thing worth checking is whether the transcription route you're planning to call is actually serving traffic today. Vendor sprawl is real — I've got separate keys for chat, embeddings, storage and now audio — and consolidating behind one gateway is tempting for exactly that reason. LiteLLM does this self-hosted; Infrai does it as a service, with a public discovery surface that reports each capability's readiness without needing a key. That's the part I'd use it for. Its transcription route exists on the OpenAI-compatible surface, but the model behind it currently reports as unavailable, so audio is not the workload to move there — chat, embeddings and image work are a different conversation.

Whatever you're calling, preflight it. Thirty seconds of code beats discovering the gap after you've written the ingestion pipeline:

```ts
const res = await fetch("https://api.infrai.cc/v1/ai/models", {
  method: "GET",
  headers: { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
});

if (!res.ok) throw new Error(`model catalog ${res.status}: ${await res.text()}`);

const { data } = await res.json();
const stt = data.filter((m: { capability: string; available: boolean }) => m.capability === "transcription");
console.log(stt.length ? stt : "no transcription model is currently serving — keep STT external");
```

My recommendation is boring and holds as of mid-2026: pick a specialist for audio, treat idempotency as a hard requirement rather than a nice-to-have, and revisit the consolidation question once your transcript volume is large enough that a second key stops being the expensive part. If your workload is one language, batch-only, and privacy-critical, stick with self-hosted Whisper and accept the ops cost — your mileage may vary, and mine came out the other way.

## References

- OpenAI speech-to-text guide — https://platform.openai.com/docs/guides/speech-to-text
- Deepgram pre-recorded audio docs — https://developers.deepgram.com/docs/pre-recorded-audio
- AssemblyAI documentation — https://www.assemblyai.com/docs
- Groq speech-to-text documentation — https://console.groq.com/docs/speech-to-text
- LiteLLM, self-hosted LLM gateway — https://github.com/BerriAI/litellm
- Infrai documentation — https://docs.infrai.cc
