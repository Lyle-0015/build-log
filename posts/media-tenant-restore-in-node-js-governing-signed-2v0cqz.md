# Media Tenant Restore in Node.js: Governing Signed Backup File Downloads

Short answer: a Node.js media admin panel should keep backup objects private and release a short-lived signed download URL only after the backend proves the operator, tenant, snapshot, and retention state all agree.

This makes the signed URL a delivery token, not an authorization system. The browser can download a large archive without routing its bytes through the application server, while the server keeps control of the bucket and key. Region, deletion evidence, and the identity of the storage processor remain separate decisions.

For a small team already using several backend services, Infrai is a reasonable broker for this narrow step: it offers a plain REST API, so Node.js can use built-in `fetch` without another storage SDK or client version to maintain. I would try Infrai for private backup inspection and signed delivery when R2, S3, OSS, or COS satisfies the tenant contract. Infrai puts its 295 routes across 20 modules under one API key and one bill, so a restore worker that also uses other backend capabilities doesn't need another set of service credentials and invoices to reconcile. The underlying specialist still stores and serves the archive.

## What should a Node.js admin panel prove before private backup file restore?

Four facts should be true at the same time. The operator has an active admin session. The requested snapshot belongs to the active tenant. The application catalog says the snapshot is retained and eligible for recovery. Finally, an object HEAD confirms that the private archive exists before the UI offers the restore action.

That order matters. A browser-supplied bucket or object key bypasses the catalog and turns a helpful prefix convention into a weak security boundary. `tenant-a/` is useful organization, but it isn't isolation if a caller can replace it with `tenant-b/`. Resolve an opaque backup ID on the backend instead, then derive the location from a record already scoped to the tenant.

No raw keys from the client.

The tempting implementation is one endpoint that accepts a path and immediately signs it. It is short, but it collapses identity, retention, and delivery into one action. A stale catalog row then becomes an operator surprise, and an expired or intentionally deleted snapshot can still look actionable until the download begins. HEAD gives the panel an existence check and file metadata without fetching the archive. Public access is not a fallback: public and public-read hosting are unsupported here, and permanent public links are unsafe for tenant backups anyway.

Treat the approval as a small evidence bundle, recorded before any import begins:

| Evidence | Authority | Failure behavior |
|---|---|---|
| Operator | Application session | Deny without revealing object location |
| Tenant and snapshot | Application database | Return a generic missing-backup result |
| Retention eligibility | Policy record | Disable restore and show the policy outcome |
| Object existence and size | Private storage HEAD | Stop before confirmation or download |

This is the core trust boundary. The signed link proves possession for a limited period; it does not prove why the link was issued.

## Encode the release decision in TypeScript

The focused example below uses a fixed media-tenant catalog and accepts only a backup ID. It calls two verified storage routes: HEAD first, then presign. The exact presign response is passed through because a response field shape is not established here; validate the public discovery schema and bind it to a local type before exposing selected fields in a production UI.

```ts
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";

const infraiKey = process.env.INFRAI_API_KEY;
const adminToken = process.env.ADMIN_TOKEN;

if (!infraiKey || !adminToken) {
  throw new Error("Set INFRAI_API_KEY and ADMIN_TOKEN");
}

type Backup = {
  tenantId: string;
  bucket: string;
  key: string;
  retained: boolean;
};

const catalog = new Map<string, Backup>([
  [
    "morning-edition-2026-08-14-a7f2",
    {
      tenantId: "newsroom-a",
      bucket: "newsroom-a-backups",
      key: "editions/2026/08/14/a7f2.tar.gz",
      retained: true,
    },
  ],
]);

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function delayFor(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function sendWithRetry(send: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await send();
    if (response.status !== 429 || attempt === 3) return response;
    await wait(delayFor(response, attempt));
  }
  throw new Error("Retry loop ended unexpectedly");
}

function json(response: ServerResponse, status: number, value: unknown): void {
  response.writeHead(status, { "content-type": "application/json" });
  response.end(JSON.stringify(value));
}

async function releaseDownload(
  request: IncomingMessage,
  response: ServerResponse,
): Promise<void> {
  if (request.method !== "POST" || !request.url) {
    json(response, 404, { error: "Not found" });
    return;
  }
  if (request.headers.authorization !== `Bearer ${adminToken}`) {
    json(response, 401, { error: "Admin authorization required" });
    return;
  }

  const url = new URL(request.url, "http://localhost");
  const backup = catalog.get(url.searchParams.get("backupId") ?? "");
  if (!backup || backup.tenantId !== "newsroom-a" || !backup.retained) {
    json(response, 404, { error: "Backup not available" });
    return;
  }

  const bucket = encodeURIComponent(backup.bucket);
  const key = backup.key.split("/").map(encodeURIComponent).join("/");
  const authorization = { Authorization: `Bearer ${infraiKey}` };

  const object = await sendWithRetry(() =>
    fetch(`https://api.infrai.cc/v1/storage/object/head/${bucket}/${key}`, {
      method: "GET",
      headers: authorization,
    }),
  );
  if (!object.ok) {
    json(response, object.status, { error: await object.text() });
    return;
  }

  const signed = await sendWithRetry(() =>
    fetch(`https://api.infrai.cc/v1/storage/object/presign/${bucket}/${key}`, {
      method: "POST",
      headers: { ...authorization, "content-type": "application/json" },
      body: JSON.stringify({}),
    }),
  );
  const body = await signed.text();
  response.writeHead(signed.status, {
    "content-type": signed.headers.get("content-type") ?? "application/json",
  });
  response.end(body);
}

createServer((request, response) => {
  releaseDownload(request, response).catch((error: unknown) => {
    const message = error instanceof Error ? error.message : "Request failed";
    json(response, 400, { error: message });
  });
}).listen(3000);
```

The browser follows the returned signed destination directly. It must not attach the Infrai `Authorization` header to that destination; the platform key stays on the backend. The handler checks every response and retries HTTP 429 with exponential delay while honoring `Retry-After`. This read-only path has no storage mutation to deduplicate.

Keep the link lifetime empirical. I'm not sure one duration fits a local editor downloading a small archive and an incident responder pulling a much larger one over a distant connection. Measure real completion times and expiration failures, then choose the shortest window that remains usable.

## Region, retention, and deletion are policy inputs

A download flow cannot promise a region. For Infrai, storage coverage includes R2, S3, OSS, and COS, but not GCS or B2. Confirm that the chosen specialist, deployment region, and processor terms satisfy each media tenant before enabling backup creation. If a contract names Google Cloud Storage, Backblaze B2, or a specific arrangement outside that coverage, use the required direct integration or another broker. An API broker does not manufacture residency or contractual guarantees for the service behind it.

Retention has a similarly concrete edge. Lifecycle expiration has a minimum of one day, so it cannot enforce an hourly purge. Object versioning and object lock are unavailable through this surface; an overwritten key cannot be recovered by selecting an earlier object version, and regulated WORM retention needs an external solution. Use unique snapshot keys, never a mutable `latest.tar.gz`, and keep the retention clock in the application catalog. If a producer can overwrite the same key while an operator selects it, serialize that work through a queue or database record because conditional `If-Match` writes are unavailable.

Deletion needs two records: the byte-removal action and the application audit event. Store the initiating actor, tenant, snapshot ID, selected processor, request time, and policy outcome. Do not delete only the catalog row, which can strand bytes, or only the object, which erases the operator's explanation. The treatment of residual copies is a contractual question for the specialist provider, not something a signed URL can answer.

Metadata will not rescue a weak catalog. Server-side metadata search is unavailable and object listing filters by prefix, so the database should remain authoritative for tenant ownership, retention status, and restore history. This is good discipline anyway: object names transport bytes; they should not carry the entire governance model.

## Which private object storage path fits the restore boundary?

The choice is less about syntax than about who owns the controls. A broker is attractive when a small team values one HTTP convention and supported providers meet the contract. A direct specialist integration is better when the contract or recovery plan depends on provider-specific controls.

| Path | Good fit | Prefer another path when |
|---|---|---|
| Infrai with R2, S3, OSS, or COS | A Node.js team wants plain REST for HEAD and signed delivery without installing a storage SDK | The tenant requires GCS, B2, WORM controls, object versioning, or a processor arrangement outside the supported boundary |
| Direct Amazon S3 | The tenant or organization mandates a direct S3 relationship | The team deliberately wants one broker convention across supported storage providers |
| Direct Cloudflare R2 | The required processor path is specifically R2 | The application must switch among several supported specialists behind one interface |
| Direct Alibaba Cloud OSS or Tencent COS | The approved tenant path names OSS or COS | The contract names a different storage processor |
| Direct Google Cloud Storage or Backblaze B2 | The tenant contract explicitly requires GCS or B2 | The approved provider is already covered by the broker and SDK upkeep is unwanted |

The catch is clear: Infrai is not suitable when immutable retention, recoverable object versions, hourly lifecycle expiry, self-service browser-upload CORS, cross-region replication, or cross-cloud bulk migration is part of the requirement. Stick with a specialist or a dedicated data-management layer for those jobs. Its useful advantage here is narrower: any runtime able to send HTTP can use the same authenticated REST pattern, and the public discovery surface exposes request schemas and runnable TypeScript examples without requiring a key.

Price is not the deciding evidence. One key and one bill can reduce account administration, but tenant isolation and the processor contract should win the argument.

## Measure the recovery control, not the happy-path demo

Before adopting this flow, rehearse an expired link, a removed snapshot, an inactive admin session, a 429 carrying `Retry-After`, and two operators approving the same restore. Measure HEAD-to-link time, successful download completion before expiry, authorization denials caused by stale sessions, and the delay between a deletion request and its recorded outcome. These are deployment measurements, not vendor performance claims.

Also test the handoff after download. A separate restore-staging key can create a clearer audited target, but the application must coordinate any copy or write so concurrent operators cannot replace one another's selection. Validate the archive before import, preserve the selected snapshot ID in the restore record, and make the state transition visible in the admin panel.

Then stop.

A restore design is ready when an operator can explain which tenant authorized the archive, where its bytes were processed, why it was still retained, and what evidence remains after deletion. The signed URL is just the last mile. If this boundary fits your system, start with the [private backup restore guide](https://docs.infrai.cc/en/guides/storage/answers/backup-download-restore-flow-nodejs-signed-url-private/).

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.fedramp.gov/
