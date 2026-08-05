# What I check before trusting async transcription webhooks for long support-call audio

If you just want the recommendation: treat the webhook as an optimization and the job list as the source of truth. For long recordings — support calls, podcasts, anything past a few minutes — submit async jobs, take the callback when it arrives, and run a reconciliation sweep that re-reads job state from the API on a timer. The sweep is the part that saves you, because callbacks do go missing, and you won't be watching when they do.

That's the whole design. The rest of this is why it's shaped that way, and what it costs to run.

## What actually breaks in an async transcription pipeline for long audio recordings?

Not the model. Every incident I've had in this pipeline was plumbing, not accuracy.

A three-hour episode isn't coming back inside one HTTP request, so any batch-oriented API hands you a job id and promises to tell you later. There are two ways to learn about "later": you ask, or they tell you. The second one is cheaper and it's the one that quietly loses data.

Here's the one that cost me a weekend. My handler verified the signature, pushed the payload onto an in-process queue, and returned 200 right away — fast ack, low retry pressure, exactly what the tutorials say to do. Then a deploy rolled the container mid-batch. Those callbacks had already been acknowledged, so nothing was ever retried, and the queue died with the process. 1,204 support calls ran through that night; 37 came out with no transcript row and no error anywhere. I found out about 9 hours later, when the nightly rollup counted fewer transcripts than recordings and the gap was too big to hand-wave. The fix is boring and I'd still paint it on the wall: a 200 means you have committed, not that you have received.

The opposite failure is duplicates. Retry policies are at-least-once, so the same completion can land twice, or land while your sweep is already writing the same result. Idempotent handling isn't optional here — RFC 9110 is explicit that retry-safety is a property you design for, not something the transport gives you. Key the write on the provider's job id and make a second apply a no-op.

Then the ones specific to long audio: an upload that times out at the 40-minute mark on a bad office link, a result URL that expires before your consumer gets to it, and chunk boundaries that slice a sentence in half so a diarized speaker turn lands in the wrong bucket.

## The job ledger, in about fifty lines

The data flow is deliberately dull. One row per recording, written before the API call. A webhook that updates the row. A sweep that re-reads anything the webhook didn't finish.

```ts
type JobStatus = "pending" | "done" | "error";

interface JobRow {
  recordingId: string;
  providerJobId?: string;
  status: JobStatus;
  submittedAt: number;
}

const API = "https://transcribe.example.com/api";
const SELF = "https://hooks.example.com";

// The row is written BEFORE the POST. If the process dies right after the
// request goes out, the sweep still knows a job may exist for this recording.
export async function submitRecording(recordingId: string, audioUrl: string): Promise<void> {
  await db.jobs.upsert({ recordingId, status: "pending", submittedAt: Date.now() });
  const res = await fetch(`${API}/jobs`, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": recordingId,
      authorization: `Bearer ${process.env.TRANSCRIBE_KEY}`,
    },
    body: JSON.stringify({ audio_url: audioUrl, callback_url: `${SELF}/hooks/transcripts` }),
  });
  const job = await res.json();
  await db.jobs.update(recordingId, { providerJobId: job.id });
}

// Commit first, acknowledge second.
export async function onCallback(req: Request): Promise<Response> {
  const raw = await req.text();
  if (!validSignature(raw, req.headers.get("x-signature"))) {
    return new Response("bad signature", { status: 401 });
  }
  const event = JSON.parse(raw);
  await applyResult(event.job_id, event);   // idempotent, keyed on the job id
  return new Response("ok", { status: 200 });
}

// Every 10 minutes. Anything still pending after 15 minutes gets re-read from
// the API, so a dropped callback costs latency instead of data.
export async function reconcile(): Promise<void> {
  const stale = await db.jobs.pending({ olderThanMs: 15 * 60_000 });
  for (const row of stale) {
    if (!row.providerJobId) {
      await submitRecording(row.recordingId, await sourceUrl(row.recordingId));
      continue;
    }
    const job = await fetch(`${API}/jobs/${row.providerJobId}`, {
      headers: { authorization: `Bearer ${process.env.TRANSCRIBE_KEY}` },
    }).then((r) => r.json());
    if (job.status !== "pending") await applyResult(row.providerJobId, job);
  }
}
```

Signature checks deserve their own three lines, because a webhook endpoint is a public POST target that writes to your database, and string equality leaks timing.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

function validSignature(raw: string, header: string | null): boolean {
  const mac = createHmac("sha256", process.env.WEBHOOK_SECRET!).update(raw).digest();
  const sent = Buffer.from(header ?? "", "hex");
  return sent.length === mac.length && timingSafeEqual(sent, mac);
}
```

Two details carry most of the weight. `applyResult` runs from both the callback and the sweep, so it has to converge to the same row either way. And the sweep's threshold is a product decision disguised as a constant: 15 minutes means a lost callback shows up as a quarter-hour delay on a support-call summary, which nobody notices, versus a missing transcript, which somebody eventually does.

## Webhook, polling, or both

| Approach | Time to result | Fails when | What it costs you |
| --- | --- | --- | --- |
| Callback only | Seconds after the job finishes | Endpoint is redeploying, DNS blips, retries exhaust | Silent data loss, no floor under it |
| Polling only | One poll interval, plus queue lag | Volume grows and you're burning requests on nothing | Request overhead, slower tail |
| Callback plus sweep | Seconds normally, one interval worst case | Both the callback and the sweep are broken at once | An extra timer and one more code path |
| Long-lived stream | Near-immediate | Connection churn, no replay on reconnect | Connection management you'll own forever |

Polling alone is genuinely fine for a while, and I'd stick with it if you're pushing a few dozen recordings a day — a cron tick every couple of minutes over a short pending list is less machinery than an authenticated public endpoint. The catch is that it stops scaling in an annoying place: interval short enough to feel live, list long enough that most polls return nothing.

Callbacks flip that. You get near-immediate results and near-zero idle traffic, and you inherit an endpoint that has to be reachable, verified, idempotent, and fast, forever. The hybrid is the only shape I've run that survives a bad deploy, and it's maybe thirty extra lines.

Streaming is a different product. If a supervisor needs live captions on an active call, this whole batch-job architecture isn't the right tool and you should be looking at a socket API instead.

## Where the money goes on long audio

Transcription is billed by audio duration almost everywhere, not by request count, which changes what you optimize. Retrying a failed HTTP call is free-ish. Re-running a 3-hour podcast because you lost the result is a full re-bill, and that's the real argument for the ledger: it isn't just reliability, it's not paying twice. Dropping and re-submitting 37 support calls cost me more than the sweep costs to run in a year.

Trimming silence before submission is the other lever. Support calls carry hold music, dead air, and IVR preamble; on a sample of about 400 of ours, cutting leading and trailing silence took roughly 12% off billable minutes. I'm not sure that generalizes — your call center's queue behavior is doing most of the work there, and podcast audio has almost no slack to cut.

Chunking is where I've seen people lose quality trying to save money. If your provider caps duration, split with overlap and stitch; if it doesn't, send the whole file and let the service handle context.

```bash
ffmpeg -i call-8842.wav -f segment -segment_time 900 -c copy chunk-%03d.wav
```

Add the diarization and word-timestamp options only where a human or a downstream model actually consumes them. On support calls, speaker labels earn their keep. On a solo podcast, I've paid for turn detection I never read once.

## What I check before turning it on

The list is short and I run it every time. Signature verified with a constant-time compare, secret rotated out of the repo. Write committed before the 200 goes back, in one transaction with the status flip. Apply path shared by the callback and the sweep, keyed on the provider job id so a duplicate delivery is a no-op. Sweep on a 10-minute timer, plus a daily alert when any job sits pending past 24 hours, because a stuck queue is invisible until you count.

Then the one that would have caught my 37 calls: a daily reconciliation metric comparing recordings created against transcripts stored, alerting on any nonzero gap. Not an average, not a rate — a raw count, in a channel someone reads. As far as I can tell there's no clever substitute for counting both sides.

And test the failure directly. Point the callback URL at a black hole for an hour, submit real jobs, and confirm the sweep catches every one of them. If it doesn't, you don't have a reconciliation sweep — you have a comment that says you do.

## References

- RFC 9110: HTTP Semantics — https://www.rfc-editor.org/rfc/rfc9110
- RFC 2104: HMAC — Keyed-Hashing for Message Authentication — https://www.rfc-editor.org/rfc/rfc2104
- RFC 9457: Problem Details for HTTP APIs — https://www.rfc-editor.org/rfc/rfc9457
- The Idempotency-Key HTTP Header Field (IETF HTTPAPI draft) — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- MDN: 202 Accepted — https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/202
- Node.js crypto: timingSafeEqual — https://nodejs.org/api/crypto.html#cryptotimingsafeequala-b
- FFmpeg segment muxer documentation — https://ffmpeg.org/ffmpeg-formats.html#segment_002c-stream_005fsegment_002c-ssegment
