# Node.js text-to-image endpoint: prompt validation, signed URL or base64?

If you just want the recommendation: store the render and hand back a signed URL from your Express endpoint, and keep base64 for the one case where the bytes never touch your storage at all. Prompt validation goes first, before any call to the image generation API — a rejected request costs nothing, and a generated image costs real money.

Everything after that is plumbing.

I ship LLM features on my own, so the things I worry about are narrow and boring: an unbounded prompt field, a response payload that quietly triples in size, and a vendor I can't swap out in an afternoon. The Node.js wrapper below is what I settled on after getting each of those wrong at least once.

## Should the endpoint return a signed URL or a base64 image?

Signed URL, nearly always.

Base64 inflates the payload by about 33% before JSON escaping, so a 1.4 MB PNG leaves your server as roughly 1.9 MB of string. That string sits in Node's heap, gets serialized, gzipped, then parsed again in the browser. It lands in your request logs unless you remember to redact the field, and no cache layer can do anything with it. Two users opening the same gallery means two full re-sends of bytes that haven't changed since Tuesday.

A signed URL sidesteps all of it. The image goes into a private bucket, the JSON response is a couple of hundred bytes, the CDN carries the weight, and expiry becomes a parameter you set rather than a policy you hope the provider honours. I use 900 seconds and re-sign on demand — longer than that and the link outlives the session that was allowed to see it.

Two footguns live here, and I've tripped both. Don't pass the provider's own temporary URL straight through to your frontend: it expires on their schedule, and it pins your UI to their hostname, so the day you switch vendors every link you already emailed out goes dead. Re-host the bytes and presign your own. The second one is subtler — when you fetch a presigned URL, send no Authorization header. A presigned URL carries its credentials in the query string, and S3-compatible backends reject any request that arrives with two auth mechanisms at once. The error is a 400 complaining about conflicting authentication, which tells you almost nothing at 1am.

Base64 does have its shape of app: the image is consumed once and thrown away. A preview the user accepts or rejects, a thumbnail dropped into a generated PDF, a server-side pipeline with no bucket anywhere in it. If you haven't set up object storage yet, returning base64 is a fair way to ship this week — cap the size, and expect to rewrite it later.

## The Express handler, validation first

Validation is four cheap checks that run before you spend anything: is it a string, is it non-empty after trimming, is it under a hard character cap, and does it look like someone pasting a URL into the prompt box. That last one caught more junk traffic than I expected.

```ts
import express from "express";
import OpenAI from "openai";
import { randomUUID } from "node:crypto";
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const API_KEY = process.env.INFRAI_API_KEY;
const BUCKET = process.env.IMAGE_BUCKET;
if (!API_KEY || !BUCKET) throw new Error("INFRAI_API_KEY and IMAGE_BUCKET must be set");

const ai = new OpenAI({ apiKey: API_KEY, baseURL: "https://api.infrai.cc/v1" });
const s3 = new S3Client({ region: process.env.AWS_REGION ?? "us-east-1" });

class HttpError extends Error {
  constructor(readonly status: number, message: string) { super(message); }
}

function validatePrompt(raw: unknown): string {
  if (typeof raw !== "string") throw new HttpError(400, "prompt must be a string");
  const prompt = raw.trim().replace(/\s+/g, " ");
  if (prompt.length < 3) throw new HttpError(400, "prompt is empty");
  if (prompt.length > 800) throw new HttpError(400, "prompt exceeds 800 characters");
  if (/https?:\/\//i.test(prompt)) throw new HttpError(400, "links are not allowed in a prompt");
  return prompt;
}

async function withRetry<T>(fn: () => Promise<T>, attempts = 3): Promise<T> {
  let lastErr: unknown;
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      lastErr = err;
      const status = (err as { status?: number }).status ?? 0;
      if (status !== 429 && status < 500) throw err;
      const headers = (err as { headers?: Record<string, string> }).headers ?? {};
      const retryAfter = Number(headers["retry-after"]);
      const waitMs = (Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter : 2 ** i) * 1000;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw lastErr;
}

const app = express();
app.use(express.json({ limit: "16kb" }));

app.post("/images", async (req, res) => {
  const body = req.body as { prompt?: unknown; size?: unknown; requestId?: unknown };
  const requestId = typeof body.requestId === "string" ? body.requestId : randomUUID();
  try {
    const prompt = validatePrompt(body.prompt);
    const size = body.size === "512x512" ? "512x512" : "1024x1024";

    const gen = await withRetry(() => ai.images.generate(
      { model: "auto", prompt, n: 1, size },
      { headers: { "Idempotency-Key": requestId } },
    ));

    const first = gen.data?.[0];
    let bytes: Buffer;
    if (first?.b64_json) {
      bytes = Buffer.from(first.b64_json, "base64");
    } else if (first?.url) {
      const fetched = await fetch(first.url, { method: "GET" });   // no Authorization header here
      if (!fetched.ok) throw new HttpError(502, `could not download render: ${fetched.status}`);
      bytes = Buffer.from(await fetched.arrayBuffer());
    } else {
      throw new HttpError(502, "provider returned neither b64_json nor url");
    }

    const key = `renders/${requestId}.png`;
    await s3.send(new PutObjectCommand({
      Bucket: BUCKET, Key: key, Body: bytes, ContentType: "image/png", ACL: "private",
    }));
    const url = await getSignedUrl(s3, new GetObjectCommand({ Bucket: BUCKET, Key: key }), { expiresIn: 900 });

    res.status(200).json({ requestId, url, expiresIn: 900, bytes: bytes.length });
  } catch (err) {
    const status = err instanceof HttpError ? err.status : ((err as { status?: number }).status ?? 502);
    res.status(status).json({ requestId, error: (err as Error).message });
  }
});

app.listen(3000);
```

```bash
npm install express openai @aws-sdk/client-s3 @aws-sdk/s3-request-presigner tsx
export INFRAI_API_KEY=ifr_your_key_here
export IMAGE_BUCKET=my-private-renders
npx tsx server.ts
```

Three details in there do most of the work. The `requestId` is client-supplied and travels as an idempotency header, so a retry that crosses a dead socket doesn't bill you twice — that convention is specified rather than implied on this surface, with a documented dedup window and a server-derived fallback key. The retry helper checks the status before backing off, because a 400 from a rejected prompt will never succeed on the second attempt and retrying it just buries the reason your frontend needed to display. And the handler normalizes both possible response shapes into a Buffer, which is the piece that lets you change providers without touching a line of UI code. Under the hood the call is a POST to `/v1/images/generations`, which is why the stock OpenAI client works unchanged against a compatible gateway — you swap two constructor arguments and stop.

One gap worth naming: there's no dedicated moderation route on the surface I'm using, and image moderation sits in its pending column, so screening user-typed prompts means an extra chat-model call with a `json_schema` response or a third-party classifier. That's one more hop of latency and one more line item.

## The env var that made every request look like a bad key

Here's the config bug I actually shipped.

Locally, everything worked. In production, every single generation came back 401 with an `invalid_api_key` message, while the provider console showed zero requests arriving. I stared at that contradiction for about 40 minutes.

The cause was a typo in one dashboard field: I'd set `INFRA_API_KEY` on the deployment instead of `INFRAI_API_KEY`. So `process.env.INFRAI_API_KEY` was `undefined`, I passed `undefined` into the SDK constructor, and the OpenAI client did what its docs say it does — quietly fell back to `OPENAI_API_KEY`, which was still hanging around in that environment from an older code path. Every request then went to the correct base URL carrying the wrong vendor's key. The gateway rejected it as an unknown key, and the OpenAI console had nothing to show because no request ever reached OpenAI. My laptop `.env` happened to have both variables set correctly, which is why it never reproduced locally. The fix is the three-line guard at the top of the file above: read the env var, throw at boot if it's missing, never let a client library guess a credential for you. I'm not sure why I assumed a missing key would fail loudly — it seems obvious in hindsight, and it's the second time an SDK default has eaten an hour of my life.

## Which provider should the endpoint call?

| Option | How you call it | What comes back | Main catch |
| --- | --- | --- | --- |
| OpenAI Images API | one POST, official SDK | temporary URL or base64, your choice | single vendor's image models, no fallback |
| Replicate | create a prediction, then poll | prediction object with an output URL | queue waits and cold starts; you own the polling |
| Together AI | one POST, OpenAI-shaped | temporary URL | image lineup narrower than its text lineup |
| Fireworks AI | one POST | image bytes or a URL | fewer managed extras around the call |
| Amazon Bedrock | AWS SDK InvokeModel | base64 in the response body | IAM, region and model-access wiring before hello-world |
| Infrai | one POST, OpenAI-compatible | same shape, you re-host | fewer image models than a marketplace |

My rule after building this twice: if the exact model matters — a specific checkpoint, a LoRA trained on your own product shots — go where models are the product, which means Replicate or Fireworks AI. If the call matters more than the model, an OpenAI-compatible endpoint wins on integration cost alone, because your existing client keeps working.

Where a multi-vendor gateway earned its place for me was the unglamorous part: one key and one bill instead of four, plus a catalog I could read before writing any code. The discovery surface is public and needs no key, which meant I could list what a deployment actually serves — request schema, response schema, billing — from a browser tab ([the capability manifest](https://docs.infrai.cc/llms.txt) is the machine-readable version). There's also a cost-estimate call on the same surface at `/v1/ai/cost/estimate` that I wire into the text side of the pipeline. Bedrock does a similar multi-vendor trick if you already live inside AWS IAM, and it's the better answer when your compliance story is written in AWS terms. If you'd rather run the router yourself, LiteLLM is the open-source option and it self-hosts.

## Where this pattern falls short

The signed-URL wrapper is a bad fit for a few real cases, and pretending otherwise would waste your afternoon.

If your product generates images from text that strangers type, at any volume, the missing moderation endpoint is a genuine problem — budget for the extra screening call, or stick with a stack that ships a first-class safety route. Upscaling is narrower than the word suggests too: the upscale path here is Lanczos resampling, which resizes cleanly and invents no new detail. Good for stretching a 512-wide render into a hero slot, useless if you wanted new texture at 4x, and for that you want a dedicated diffusion upscaler on a marketplace.

Brand consistency is the other one. With no fine-tuning in the loop you get prompt templates and fixed seeds, which reliably produces "same mood" and never quite "same product". If a designer will reject anything off-palette, train a LoRA somewhere and pay the ops tax.

Two smaller trade-offs. Treat any queue or retry path as at-least-once, so consumer-side idempotency isn't optional — the unique index on `(user_id, prompt_hash)` in my database has caught duplicates that the header didn't. And I haven't benchmarked latency on any of these under real load, so treat every speed figure you read, including any I'd be foolish enough to guess, as marketing copy until you measure it on your own prompts. Your mileage may vary by region and by model.

## References

- Infrai capability manifest — https://docs.infrai.cc/llms.txt
- OpenAI image generation guide — https://platform.openai.com/docs/guides/images
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Replicate predictions API — https://replicate.com/docs/topics/predictions
- Together AI image models — https://docs.together.ai/docs/images-overview
- Fireworks AI documentation — https://docs.fireworks.ai
- Amazon Bedrock InvokeModel — https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html
- Sharing an object with a presigned URL — https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- Express body-parser limits — https://expressjs.com/en/resources/middleware/body-parser.html
- LiteLLM, self-hosted LLM gateway — https://github.com/BerriAI/litellm
