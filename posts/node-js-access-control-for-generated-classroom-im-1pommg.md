# Node.js Access Control for Generated Classroom Images in Object Storage

Short answer: store each AI-generated classroom image under an opaque object key in a private bucket, keep that key in the application database, authorize the student or instructor on every download request, and only then create a short-lived presigned GET URL. That is the least complex design that keeps storage credentials off the client without turning model output into a public asset.

The flow has two separate trust decisions. A generation worker writes image bytes to object storage and commits an ownership record. Later, the API authenticates the requester, checks course and submission access in the database, and asks the storage adapter to sign a download for the recorded key. The browser receives a temporary capability URL; it never receives the bucket credential.

Keep the key. Discard the URL.

## Implementation walkthrough for a private image gate

This TypeScript example keeps provider details behind a deliberately narrow interface. The storage adapter is responsible for the actual object PUT and presigned GET operations; the application owns identity, authorization, metadata, and expiry policy. That boundary matters because replacing an object-storage service shouldn't require rewriting course-access rules.

```ts
import { randomUUID } from "node:crypto";

type ImageRecord = {
  id: string;
  objectKey: string;
  ownerId: string;
  courseId: string;
  contentType: "image/png" | "image/jpeg" | "image/webp";
  byteLength: number;
};

interface PrivateObjectStorage {
  put(input: {
    key: string;
    body: Uint8Array;
    contentType: ImageRecord["contentType"];
  }): Promise<void>;
  presignGet(input: {
    key: string;
    expiresInSeconds: number;
    responseContentType: ImageRecord["contentType"];
  }): Promise<string>;
}

interface ImageRepository {
  insert(record: ImageRecord): Promise<void>;
  findById(id: string): Promise<ImageRecord | null>;
}

type Viewer = {
  userId: string;
  courseIds: ReadonlySet<string>;
  isInstructor: boolean;
};

function mayDownload(viewer: Viewer, image: ImageRecord): boolean {
  return (
    viewer.userId === image.ownerId ||
    (viewer.isInstructor && viewer.courseIds.has(image.courseId))
  );
}

export async function saveGeneratedImage(
  storage: PrivateObjectStorage,
  images: ImageRepository,
  input: {
    bytes: Uint8Array;
    ownerId: string;
    courseId: string;
    contentType: ImageRecord["contentType"];
  },
): Promise<ImageRecord> {
  const id = randomUUID();
  const extension = input.contentType.split("/")[1];
  const objectKey = `courses/${input.courseId}/outputs/${id}.${extension}`;

  await storage.put({
    key: objectKey,
    body: input.bytes,
    contentType: input.contentType,
  });

  const record: ImageRecord = {
    id,
    objectKey,
    ownerId: input.ownerId,
    courseId: input.courseId,
    contentType: input.contentType,
    byteLength: input.bytes.byteLength,
  };
  await images.insert(record);
  return record;
}

export async function createImageDownload(
  storage: PrivateObjectStorage,
  images: ImageRepository,
  viewer: Viewer,
  imageId: string,
): Promise<{ status: 200; url: string; expiresInSeconds: number } | { status: 404 }> {
  const image = await images.findById(imageId);

  // Return the same response for missing and unauthorized records.
  if (!image || !mayDownload(viewer, image)) return { status: 404 };

  const expiresInSeconds = 120;
  const url = await storage.presignGet({
    key: image.objectKey,
    expiresInSeconds,
    responseContentType: image.contentType,
  });

  return { status: 200, url, expiresInSeconds };
}
```

The application route that calls `createImageDownload` should derive `viewer` from a verified session, never from a request-body user ID. Return the URL only over HTTPS and set the API response to `Cache-Control: private, no-store`. A URL issued for 120 seconds can still be copied and used until it expires, so the expiry is a damage limit, not a second authorization check.

There is one awkward failure window in `saveGeneratedImage`: the object PUT can succeed while the database insert does not. Don't hide it. Consider a worker that receives generation `gen_1842`, writes `courses/course_8/outputs/gen_1842.png`, and then loses its database connection before committing the image row. A blind retry with a new random key writes a second copy; the first has no owner record, no visible UI state, and no obvious deletion date. A safer sequence starts with an upload-intent row keyed by the stable generation ID, writes the object to the deterministic key, and changes the row to `ready` only after storage confirms the write. If the worker stops between those steps, a reconciliation job can inspect stale `uploading` intents, check for the expected key, and either finish the transition or remove the object after a grace period. The job must be idempotent because it may itself retry. This doesn't make the database and bucket one transaction. It makes every partial state named, observable, and recoverable without guessing which object belongs to which student submission.

Treat the database record as the authority and the bucket as a byte store. The row should hold the opaque object key, owner, course or tenant boundary, media type, byte length, creation time, retention state, and generation job ID. It should not hold the presigned download URL. Signed URLs expire, may embed security-sensitive query parameters, and make poor durable identifiers.

Key names deserve restraint. A prefix such as `courses/course_8/outputs/01J...png` is useful for lifecycle operations, but putting a student's email, assignment title, or prompt in the key leaks context into logs and request traces. An opaque ID provides enough uniqueness while the database carries the descriptive data. For the same reason, don't accept a client-provided object key in the download route and then sign it. Look up a server-stored key by image ID after authorization.

The private bucket policy is the default, not a cleanup step. Public access should not be required for either the generation worker or the browser: the worker uses a narrowly scoped write identity, and the browser uses the expiring GET URL created after the application check. Separate buckets or prefixes for raw student uploads, generated outputs, and publishable course assets can make retention and access reviews less ambiguous. A published illustration is a different data class from a private draft, even when both are PNG files.

Validate the bytes before storage. Check the decoder-recognized media type, enforce a byte limit, and generate any thumbnail in a controlled worker rather than trusting a filename extension. If the model returns base64, decode it once at the worker boundary and upload binary bytes; carrying base64 through the rest of the system increases memory and transfer overhead without improving access control.

## Reliability drill: stale access and partial writes

A presigned URL proves that whoever created it had signing authority. It does not know that a student dropped a course five minutes ago or that an instructor lost access to a class. Those facts live in the application, which is why signing must follow a fresh authorization decision.

This is easy to test and easy to get subtly wrong. Use fixtures for an owner, an instructor assigned to the course, an instructor assigned elsewhere, and an unrelated student. The first two should receive a URL. The others should receive the same `404` shape as a nonexistent image so the endpoint does not reveal which IDs exist. Also test that the storage signer is never called on a denied request; checking only the HTTP response can miss an expensive or security-relevant side effect.

A specific regression test should advance a fake clock by 121 seconds and assert that the issued capability is rejected by the storage layer. Another should revoke course membership before asking for a new link. Existing links generally cannot consult the changed membership state, which exposes the central trade-off: shorter expiry limits exposure, while longer expiry reduces refreshes and failed downloads on slow connections. I'm not sure one duration works for every classroom. Image size, mobile-network behavior, and the sensitivity of the assignment should decide it.

Revocation changes the architecture. If access must stop immediately, don't send the browser a direct presigned object URL. Stream the object through an authenticated application or edge endpoint that checks policy on every request, accepting the extra compute, bandwidth, and latency. For ordinary generated previews, a brief direct-download capability is often the simpler delivery path. For exam materials, safeguarding assessments, or content under a legal hold, the proxy can be the correct cost.

The catch is that direct signed delivery gives up some control after issuance. It is not suitable when every byte-range request needs current authorization, when downloads require one-time use, or when audit policy demands an application event for the completed transfer. Stick with an authenticated proxy in those cases. A direct URL is also a bearer capability: avoid placing it in analytics events, exception messages, chat transcripts, or durable logs.

Browser behavior adds another boundary. If the frontend fetches the image across origins rather than navigating directly to it, the object endpoint's cross-origin policy must allow the application origin and required method or headers. CORS controls whether browser JavaScript may read a cross-origin response; it is not user authorization. Test from the real production origin because a same-origin local setup will not exercise that path. The MDN CORS guide documents the browser's request and preflight model.

Cost is broader than stored gigabytes. Capacity planning should track original and derived image bytes, PUT and GET request counts, retention duration, abandoned generations, and data transfer. Published object-storage pricing separates storage, requests, retrieval or management features, and data-transfer considerations, so a cheap-looking storage rate alone is not a useful estimate. Measure the workload shape first. Then compare current provider terms against it.

Latency has a similar shape. Direct delivery removes the application from the data path, but the user may need one API round trip to obtain the URL before the image request begins. A client can request a fresh link when it needs the image, while avoiding speculative signing for every item in a long gallery. Don't mint hundreds of links that may never be opened.

## How can Node.js object storage control private image download cost?

Before deployment, exercise the upload, authorization, signing, download, expiry, and deletion path as one workflow. Confirm that logs contain the image ID and generation ID but neither credentials nor signed query strings. Emit counters for stored bytes, upload failures, denied download attempts, sign operations, orphan cleanup, and deletion lag; alert on trends rather than treating each denied request as an incident. The database and object inventory should be reconcilable in both directions, with explicit handling for an object without a row and a row without an object.

Retention should be a product rule expressed in data, not an undocumented bucket setting. Mark when a generated image becomes eligible for deletion, let a worker delete it, and retain enough tombstone state to make retries idempotent. A bucket lifecycle policy can serve as a final backstop, but the application still needs to explain to a learner why an image is available or gone. Test this with a fake clock and a tiny fixture image in continuous integration, then run a scheduled canary against the deployed storage adapter without exposing its URL.

Small system. Sharp edges.

The shipping decision is straightforward: use private object storage plus short-lived direct downloads for ordinary course images, and choose an authenticated proxy when immediate revocation or per-request audit is more important than delivery simplicity. In either design, keep identity and policy in the application, keep durable object keys in the database, and make cleanup part of the write path before usage grows.

## References

- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Object storage pricing dimensions](https://aws.amazon.com/s3/pricing/)

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
