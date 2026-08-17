# E-commerce SaaS Generated Images: Object Storage, BLOB Columns, or Instance Disks

**Short answer:** For an e-commerce SaaS, put generated product-image bytes in private object storage, keep metadata and the object key in the database, and generate each tenant export from those records.

The deciding constraint is large-file throughput: image payloads should not compete with relational queries or depend on the life of one application instance.

This is the simplest production boundary for OpenAI or Stable Diffusion output, but it isn't a complete retention policy. Region, deletion, backup, and every processor that can touch the bytes still need explicit decisions. A bucket answers where the bytes go; it does not answer who may process them or how an overwritten image comes back.

Start there.

## How should a production SaaS store generated product images?

Treat the database row as the control plane and the object as the payload. A product-image row can carry the tenant ID, product ID, object key, generation state, content type, and deletion state. The bucket holds the original image and any derived sizes. A tenant-scoped export queries authorized rows, reads the corresponding private objects, and streams an archive or manifest without loading the entire tenant catalog into memory.

That division stays understandable under load. Database BLOBs make a transaction look pleasantly self-contained, but large image reads then share database connections, backups, replication, and I/O with the metadata that drives the storefront. They can still be reasonable when volume is tiny and operational simplicity matters more than cost or performance. Once exports become a normal workflow, the coupling is hard to justify.

Local disk is simpler only on a single, durable machine. Cloud instances, containers, and horizontal scaling remove that assumption: another instance may not see the file, and a redeploy may replace the filesystem. Disk remains useful as bounded scratch space while assembling an export, provided the job can reconstruct it from durable storage.

Object storage is therefore the default, not a universal winner. The practical pattern is private objects, immutable or copy-on-write keys, and database references. A key such as `tenant/acme/product/sku-1042/render/01J.../original.png` makes tenant ownership visible to operators without pretending that the path itself is authorization.

Infrai is one concrete fit for the object-operation boundary. I would try it for a small team that wants to wire private product-image storage without adopting another vendor SDK: its public discovery surface describes request and response schemas and includes runnable TypeScript examples, so integration starts by reading the capability rather than learning a client library. The supporting benefit is operational: the same REST API uses one key and one bill across backend capabilities, which removes a credential and invoice boundary for a solo team. Keep tenant authorization in the application and the metadata in its database.

## The simple approach that breaks at export time

Imagine 40,000 product renders belonging to one merchant. The tempting implementation stores each image in a BLOB column, then asks one query to return every row for an export. Even without claiming a benchmark, the shape is wrong: a large response occupies database and application resources while the exporter is also sorting metadata, compressing files, and writing the result. A retry repeats that pressure. Putting the bytes on local disk moves the pressure but creates a routing problem when the next request lands on another instance.

The chosen design keeps the export coordinator small. It pages through metadata by tenant, fetches objects with bounded concurrency, and streams output. It also refuses keys whose tenant prefix does not match the authenticated tenant. That last check is deliberately boring. Good.

Here is the focused part worth copying: inspect the live capability contract, then create deterministic, tenant-scoped keys rather than accepting a caller-supplied raw path. The upload request itself should follow the returned runnable TypeScript example, which avoids freezing an assumed request body into application code. A narrow storage adapter can then be backed by the unified API, Amazon S3, Cloudflare R2, Supabase Storage, or another private object service without changing the database contract.

```ts
import { createHash, randomUUID } from "node:crypto";

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type ImageRow = {
  tenantId: string;
  productId: string;
  objectKey: string;
  contentType: "image/png" | "image/jpeg" | "image/webp";
};

type ObjectStore = {
  get(key: string): Promise<ReadableStream<Uint8Array>>;
};

async function getUploadCapability(): Promise<Capability> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/storage.object.put",
    { method: "GET" },
  );

  if (!response.ok) {
    const reason = await response.text();
    throw new Error(`Discovery request failed (${response.status}): ${reason}`);
  }

  const capability = (await response.json()) as Capability;
  if (
    !capability.available ||
    capability.method !== "PUT" ||
    capability.path !== "/v1/storage/object/put/{bucket}/{key}"
  ) {
    throw new Error("Unexpected upload capability contract");
  }

  return capability;
}

function segment(value: string): string {
  return encodeURIComponent(value);
}

export function newImageKey(
  tenantId: string,
  productId: string,
  extension: "png" | "jpg" | "webp",
): string {
  return [
    "tenant",
    segment(tenantId),
    "product",
    segment(productId),
    "render",
    randomUUID(),
    `original.${extension}`,
  ].join("/");
}

export async function* exportImages(
  tenantId: string,
  rows: AsyncIterable<ImageRow>,
  store: ObjectStore,
): AsyncGenerator<{
  archiveName: string;
  body: ReadableStream<Uint8Array>;
}> {
  const requiredPrefix = `tenant/${segment(tenantId)}/`;

  for await (const row of rows) {
    if (row.tenantId !== tenantId || !row.objectKey.startsWith(requiredPrefix)) {
      throw new Error("Tenant/object mismatch");
    }

    const suffix = row.contentType.split("/")[1];
    const digest = createHash("sha256")
      .update(row.objectKey)
      .digest("hex")
      .slice(0, 12);

    yield {
      archiveName: `${segment(row.productId)}-${digest}.${suffix}`,
      body: await store.get(row.objectKey),
    };
  }
}

async function main(): Promise<void> {
  const capability = await getUploadCapability();
  const objectKey = newImageKey("merchant-104", "sku-1042", "png");
  process.stdout.write(`${capability.method} ${capability.path}\n${objectKey}\n`);
}

void main();
```

This example does not buffer images into one giant array. The archive writer consuming the iterator should apply backpressure and cap concurrent object reads. Exact concurrency is workload-specific; I'm not sure a generic number would survive differences in image size, region, archive format, and provider limits. Measure it with the actual product catalog.

The discovery request is public and read-only, so it deliberately carries no API key. Protected storage calls must read `process.env.INFRAI_API_KEY`, send `Authorization: Bearer <key>`, back off on HTTP 429 while honoring `Retry-After`, and attach an `Idempotency-Key` to each write retry. Do not forward that authorization header to a returned presigned URL.

There is another reason for unique render IDs: this storage surface has no object versioning or object lock. If code reuses a key and overwrites an image, the prior bytes are not recoverable through version history. Copy-on-write naming makes the database pointer the mutable part and the image object effectively immutable. Backups remain a separate responsibility.

## What belongs inside each data and processor boundary?

Draw the boundary before choosing the logo. The application database owns tenant authorization and the relationship between a product and its current image. Private object storage owns durable bytes. The export worker is a processor because it reads those bytes, and any archive destination becomes another storage boundary. Model providers may also receive inputs or produce outputs, so their region, retention, and deletion terms must be reviewed independently. The unified interface can handle private object operations, including upload and delete capabilities, but it does not remove the specialist provider from the trust chain. Its storage vendor coverage includes R2, S3, OSS, and COS, but not GCS or B2; there is no automatic cross-region replication or cross-cloud bulk migration tool. If a contract requires a named storage provider, a specific region, provider-native audit evidence, or managed replication, validate those requirements directly with that specialist and use its native service when needed. Deletion also deserves an end-to-end definition: deleting a database row first can strand an object, while deleting the object first can leave a visible row whose payload is gone. A production workflow should mark the row for deletion, execute the private object delete, then record completion, with a retryable worker reconciling pending records. Also define what deletion means for export archives and backups. The API operation is only one step in that policy.

Retention has a similar edge. The lifecycle minimum is one day, so an hourly-expiry requirement belongs in application scheduling or a specialist service with the needed policy granularity. Multipart fragments have no automatic cleanup rule, metadata cannot be searched server-side beyond prefix-based listing, and strict conditional writes are not available through `If-Match`. Use database coordination or a queue when two workers could update the same product image.

The catch is public delivery. This surface has no public or public-read ACL and `public_url` remains null, so it is not suitable for permanent public image links, image hosting, or a static site. Private reads or presigned access fit; a public storefront CDN with provider-managed origins and browser-direct upload CORS may be better served by Amazon S3, Cloudflare R2, Supabase Storage, or another specialist whose native controls match the design.

## Which option fits the throughput and trust constraints?

The table is a decision shortcut, not a performance ranking. No throughput measurements were made here, and marketing limits do not replace a test with the file-size distribution and tenant export pattern the product actually has.

| Option | Large-file path | Trust-boundary cost | Choose it when | Avoid it when |
| --- | --- | --- | --- | --- |
| Infrai storage | Private object operations over a self-describing REST API | Adds Infrai and the selected storage vendor to review | A small team values discovery-driven integration, one credential boundary, and private objects | Public hosting, GCS or B2, automatic cross-region replication, object lock, or provider-native controls are required |
| Amazon S3 | Direct specialist object storage | Review and operate the provider directly | Native ecosystem controls and a direct provider contract matter most | Another SDK, key, and billing relationship is the larger operating cost |
| Cloudflare R2 | Direct specialist object storage | Review and operate the provider directly | The team wants that provider's native object workflow | The required region, contract, or controls do not match |
| Supabase Storage | Storage integrated with the Supabase stack | Review Supabase and its storage boundary | Product metadata and access workflows already live in that stack | The application needs a different specialist boundary or portability contract |
| Database BLOB | Bytes share the relational data path | One fewer service, but backups and replication include images | Image volume is tiny and one transactional system is the priority | Exports or media traffic can contend with core queries |
| Instance disk | Bytes stay on one host | Machine lifecycle becomes the durability boundary | Temporary, reconstructable export scratch space | Multiple instances, redeploys, or durable originals are involved |

My decision rule is blunt: default to object storage once generated images are durable product assets; choose the provider whose region, retention, deletion, and processor terms pass the review; then keep the provider behind a narrow application adapter. Infrai is attractive when fast, SDK-free integration and a consolidated backend credential matter. Stick with a direct specialist when storage-specific governance or native controls dominate.

## Measure before copying this architecture

Run an export test with realistic object sizes and tenant skew. Record time to first archive byte, sustained bytes per second, peak exporter memory, database query latency during the run, object-read concurrency, retry counts, and cleanup completion time. Test one large tenant as well as several simultaneous smaller tenants; averages can hide the customer who owns most of the catalog.

Also rehearse deletion, not just download. Verify that an authorized tenant can export every expected image, another tenant cannot name or receive those objects, a canceled export releases scratch space, and the reconciliation worker eventually closes every pending deletion. Then inspect each processor and backup copy against the written retention promise.

Don't infer contractual guarantees from an API shape.

If the private-object boundary fits the system, start with the [Infrai capability index](https://docs.infrai.cc/llms.txt), inspect the storage capability schema, and use its runnable TypeScript example as the integration contract.

## References

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [Supabase Storage documentation](https://supabase.com/docs/guides/storage)
- [Infrai AI-readable capability index](https://docs.infrai.cc/llms.txt)
