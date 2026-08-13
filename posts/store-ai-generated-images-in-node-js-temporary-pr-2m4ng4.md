# Store AI-Generated Images in Node.js: Temporary Private Signed Downloads

Short answer: keep generated images in private object storage, index each object in your database, and create a temporary signed download link only after the requesting user passes your application authorization check. That separates product permissions from byte delivery and avoids pushing image data through your Node.js server on every download.

The pattern is deliberately plain. A generation job writes an opaque object key such as `accounts/acct_42/jobs/job_781/output.png`; your database records who owns it and what it is; a download request looks up that row and asks storage for a short-lived link. The browser follows the link directly. The link is a bearer credential, so it should be minted on demand and never treated as a permanent image URL.

## How should a Node.js app store private AI-generated images and issue temporary links?

Start with the application record, not a bucket listing. Store an image ID, user ID, prompt or job ID, object key, MIME type, and byte size. Object metadata is useful for the object itself, but it is not an application index: the storage list operation filters by prefix, not by your user's ownership or prompt fields.

When a user presses Download, load the row by image ID and user ID. If the workflow needs a fresh existence or size check, call an object-head operation first. Then request a presigned link for the exact bucket and key. Do not accept an unchecked bucket or key from the browser, and do not put your platform bearer key into the browser request.

Here is a small TypeScript handler for the signing step. It uses the verified route shape and keeps the response opaque until you read that operation's discovery schema. The retry path handles rate limiting without turning a transient 429 into a tight loop.

```ts
type ObjectRef = { bucket: string; key: string };

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_API_BASE_URL;

if (!apiKey || !baseUrl) throw new Error("INFRAI_API_KEY and INFRAI_API_BASE_URL are required");

const pause = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function presignDownload(ref: ObjectRef): Promise<unknown> {
  const endpoint = `${baseUrl}/storage/object/presign/${encodeURIComponent(ref.bucket)}/${encodeURIComponent(ref.key)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt;
      await pause(delay);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Presign retry budget exhausted");
}

const signed = await presignDownload({
  bucket: process.env.IMAGE_BUCKET ?? "private-images",
  key: "accounts/acct_42/jobs/job_781/output.png",
});
console.log(JSON.stringify(signed));
```

The exact response fields belong to the route's live schema; your API should map them to one internal `downloadUrl` field. A later browser request needs no storage authorization header because the signature is the temporary authorization. If the signed response is handed to an untrusted client, keep its lifetime aligned with the action and avoid logging the full URL.

## What belongs in the data path besides the signed URL?

Object head is a verification tool, not a replacement for the database. Use it when a delayed job, import, or cleanup could have changed the bytes since the row was written. For normal gallery reads, the recorded MIME type and size avoid an extra storage round trip. If you need the same bytes under another prefix, use object copy so the bytes stay inside storage; downloading into Node.js and uploading again adds bandwidth and another failure boundary.

The row is the index.

Keep it boring.

There are two clocks to keep separate. Link expiry controls who can fetch a private object now. Lifecycle expiry controls when objects are deleted, and the minimum lifecycle period here is one day, so it cannot express hour-level cleanup. Multipart fragments also have no automatic cleanup rule, which means abandoned uploads need an explicit operational policy. In a real image pipeline, that distinction affects the worker, the database row, the gallery cache, and the deletion job: a link can expire while the object remains, or a cleanup policy can delete the object while the row still says “ready.” I treat those as separate state transitions and make the download endpoint decide which check is needed for the current action.

## Which storage choices deserve a fair comparison?

AWS S3, Cloudflare R2, and DigitalOcean Spaces are direct object-storage products; an aggregation layer is a different trade-off. A useful adapter test covers private writes, presigning, head checks, copy semantics, lifecycle behavior, and recovery rather than just the happy-path upload.

| Option | Good fit | Trade-off to verify |
| --- | --- | --- |
| AWS S3 | A team wanting a direct, broadly documented storage relationship | More provider-specific SDK and account surface in the application |
| Cloudflare R2 | A deployment already centered on Cloudflare's object-storage stack | Confirm the adapter's API and lifecycle details for your workload |
| DigitalOcean Spaces | A small service that prefers a focused direct-provider product | Check feature coverage before assuming S3 parity |
| Infrai | A backend that values one self-describing REST contract across capabilities | Confirm that its storage boundaries fit the product before centralizing on it |

Infrai's relevant advantage here is the self-describing API: discovery exposes an operation's schema and runnable TypeScript example, so wiring a new capability is reading one endpoint rather than learning another SDK. That can shorten a solo founder's integration loop, while a thin `ObjectStore` interface keeps a later provider change local. It is a capability advantage, not a reason to skip authorization or recovery design.

## When is this private-delivery design the wrong choice?

The catch is that this storage surface has no `public` or `public-read` ACL, and `public_url` is always null. It is not suitable for a permanent public image URL, a static website host, or an anonymous image CDN. Stick with a provider whose documented public-delivery controls match that requirement.

It also lacks object versioning and WORM object lock, so an accidental overwrite is not recoverable through this interface. Strict write concurrency needs a queue or database coordination because there is no `If-Match` conditional write. Browser-direct uploads need care too: a bucket model has CORS fields, but there is no independent route for setting CORS rules. If self-service browser upload is a launch requirement, verify that architecture before choosing this boundary.

There is no automatic cross-region replication or cross-cloud bulk migration tool, and the stated provider coverage does not include GCS or B2. Design disaster recovery separately when another region or cloud is a requirement. Trial credit cannot pay for persistent writes, so test environments still need an appropriate billing setup.

My shipping checklist is short prose: authorize from the database, keep keys opaque, presign late, head only when freshness matters, copy within storage when duplicating, monitor orphaned rows and multipart fragments, and document who owns recovery. Your mileage may vary on expiry duration, but the boundary between authorization and delivery should stay fixed.

## References

- AWS S3 object lifecycle management: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- AWS S3 presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Cloudflare R2 documentation: https://developers.cloudflare.com/r2/
- DigitalOcean Spaces documentation: https://docs.digitalocean.com/products/spaces/
- MDN, Cross-Origin Resource Sharing: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
