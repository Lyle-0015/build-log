# AI Image Generation for a Node.js SaaS: Uploads, Presets, Ratios, and Pricing

Short answer: put the image generator behind an asynchronous, server-owned job policy. Let Next.js collect an upload, prompt subject, preset, and aspect ratio; let Node.js validate them, reserve a tenant budget, and record every provider operation before it runs. The useful unit is not “one prompt.” It is one auditable image job with a known owner, policy version, and cost allowance.

That design fits an e-commerce SaaS better than a prompt box wired straight to an image API. A sales call may end with a CRM action such as “create a product campaign,” but the resulting image work still has to answer boring questions: which tenant paid for it, which preset was used, whether the upload was allowed, and what happens when a user opens two browser tabs. Those questions are the feature.

The narrow demo is easy. The accounting is where it survives contact with customers.

## Why does an AI image generator need a job ledger?

Start with a request record, not a provider response. A job should have an application-generated ID, tenant ID, authenticated actor, preset ID, requested ratio, upload reference, policy version, status, and an allowance reservation. Store the prompt subject separately from the rendered prompt so a later review can tell what the customer entered and what the server added.

The state machine can stay small: `queued`, `running`, `succeeded`, `failed`, and `cancelled`. A reservation belongs to the job, not the browser tab. On success, settle the reservation with the measured operation cost; on an accepted failure, release it according to the accounting policy; on a retry, create a new attempt under the same job rather than silently charging a second operation to the first attempt.

| Evidence | Recorded on | Why it matters |
| --- | --- | --- |
| Tenant and actor IDs | Every state transition | Support can identify ownership without reading prompt text |
| Preset, ratio, and policy version | Job creation | A later policy edit does not rewrite history |
| Reservation and settlement | Each attempt | Usage reports can reconcile estimates with actual charges |
| Provider status and attempt number | Worker events | Retry decisions remain explainable |

This is especially important for per-tenant cost visibility. A monthly total is too late to help a support engineer explain a bill. The ledger should make it possible to group usage by tenant, preset, aspect ratio, action type, and request ID. It should also retain the policy decision that rejected a request. Rejections are product data: a tenant repeatedly hitting an image-count limit may need a different plan, while repeated prompt-policy rejections may mean the preset wording is too broad.

Consider two browser tabs opened from the same CRM action. Both send the same tenant ID and the same preset, and both see one remaining allowance if they read the balance before either writes. A plain read-then-write check therefore accepts both. The server should instead create a reservation with a conditional balance update, attach that reservation to a job ID, and return the job only after the update succeeds. The worker claims the queued job once, records its attempt, and settles the reservation after the provider response has been mapped to a terminal result. If the worker loses its connection after the remote call, the job must remain in an unknown or retry-review state until the adapter's documented idempotency behavior makes another attempt safe; treating every network timeout as a free retry is how a small SaaS loses both money and trust. The UI can poll a job status or receive an event, but it should never infer completion from a button click. This is a longer path than sending the prompt directly, yet every step answers a question a tenant will eventually ask.

Do not let the client calculate the charge. The client can display an estimate, but the server owns the reservation and the final settlement. A transaction or equivalent compare-and-set operation prevents two simultaneous requests from spending the same remaining allowance. A queue then gives the worker one place to enforce concurrency and retry rules.

## How should a Next.js and Node.js app handle uploads, prompts, presets, and aspect ratios?

Treat the browser as an input surface, not a policy boundary. The server should resolve a preset from an allowlist and discard client attempts to replace its template, output count, or credit weight. Keep the first preset set deliberately small: a square product tile, a portrait catalog card, and a wide campaign image are enough to expose most of the workflow without making the UI a matrix of exceptions.

An upload needs its own gate before it reaches a worker. Check the authenticated tenant, declared media type, byte limit, and storage ownership. Generate a private storage key rather than accepting a path supplied by the browser. If the chosen image operation does not accept reference material, reject that combination as a documented capability boundary. Do not pass an unverified field name into an adapter and hope the remote service interprets it correctly.

The prompt is another boundary. Store a user subject such as “blue insulated bottle on a kitchen counter” as data, then compose it with the selected server-side preset. Keep moderation, authentication, upload validation, and budget checks as separate decisions. OWASP's LLM guidance is a useful reminder that user-controlled text remains untrusted after it has been placed inside a friendly template.

Here is the policy layer I would test independently of any image provider. It returns a job input and an explicit local endpoint; the worker's provider adapter is a separate module whose schema must come from its current, verified documentation.

```ts
type PresetId = "catalog-square" | "catalog-portrait" | "campaign-wide";
type Ratio = "1:1" | "4:5" | "16:9";

type ImageInput = {
  tenantId: string;
  actorId: string;
  presetId: PresetId;
  subject: string;
  ratio: Ratio;
  uploadKey?: string;
};

const PRESETS: Record<PresetId, { ratios: readonly Ratio[]; maxSubject: number }> = {
  "catalog-square": { ratios: ["1:1"], maxSubject: 240 },
  "catalog-portrait": { ratios: ["4:5"], maxSubject: 240 },
  "campaign-wide": { ratios: ["16:9"], maxSubject: 320 },
};

export function validateImageInput(input: ImageInput) {
  const preset = PRESETS[input.presetId];
  const subject = input.subject.trim();

  if (!input.tenantId || !input.actorId) {
    throw new Error("Missing authenticated context");
  }
  if (!preset || !preset.ratios.includes(input.ratio)) {
    throw new Error("Preset and aspect ratio do not match");
  }
  if (subject.length < 3 || subject.length > preset.maxSubject) {
    throw new Error("Subject length is outside the preset policy");
  }
  if (input.uploadKey && !input.uploadKey.startsWith(`${input.tenantId}/`)) {
    throw new Error("Upload does not belong to the tenant");
  }

  return {
    tenantId: input.tenantId,
    actorId: input.actorId,
    presetId: input.presetId,
    ratio: input.ratio,
    subject,
    uploadKey: input.uploadKey,
  } as const;
}
```

The policy is intentionally unexciting. A request from a CRM action can call it with a known tenant and preset; a human using the Next.js form can call the same function through the server route. The two entry points then share validation, reservation, and audit fields instead of developing slightly different meanings for “one image.”

## What should the worker measure before it ships an image job?

Measure the full path, not just model latency. Useful fields include queue wait, provider time, upload bytes, ratio, preset, attempts, reservation amount, settled amount, and final status. Tag each event with the tenant and job ID, but keep prompt text out of general logs unless a documented retention policy permits it.

The most revealing test cases are not happy-path prompts. Send an oversized upload and expect a local rejection. Change the tenant prefix on an upload key and expect a 403-style application decision before an external call. Submit the same idempotency key twice and verify that the ledger has one charge. Open two concurrent requests with one remaining allowance and verify that only one reservation wins. Force a provider timeout and verify that the worker can distinguish an unknown outcome from a confirmed rejection before retrying.

I treat a `413`-class upload rejection and a `429`-class rate response as different product events. The first is an input or plan problem; the second is an operational scheduling problem. Retrying both with the same policy creates noise and can create duplicate work. A retry should use bounded exponential backoff, honor a supplied retry delay, and stop after a small configured attempt count. The job should remain inspectable after the worker gives up.

Use contract tests for the provider adapter and policy tests for the app. The adapter test should assert the exact method, request shape, authentication behavior, response mapping, and status handling documented for the selected image service. The policy test should assert that disallowed ratios and uploads fail without invoking the adapter. No endpoint guessed from REST naming belongs in production code.

I'm not sure one preset count will fit every catalog. Your mileage may vary with image reuse, seasonal campaigns, and how often a sales call becomes a real CRM action. That is why the first rollout should compare completed jobs, repeat generations, rejection reasons, queue delay, and settled cost per tenant by preset. Set thresholds from that baseline rather than borrowing a universal “acceptable” number.

## When should a team change the architecture or the backend?

Change the architecture when the ledger is no longer answering the operational questions. If jobs pile up, add worker concurrency controls and a visible queue state before increasing provider parallelism. If tenants cannot predict spend, improve estimates and reservation messaging before adding more models. If users reject the output, inspect preset fixtures and crop policy before exposing a larger prompt surface.

The trade-off is real: a narrow preset system limits creative freedom, a queue adds latency, and a ledger adds storage and reconciliation work. That is acceptable for a SaaS product whose promise includes predictable tenant usage. It is not suitable when the product is an open-ended creative playground where customers require arbitrary dimensions and many independent attempts; in that case, a different quota model and a more expressive workflow may be the honest choice.

Pricing belongs in the decision record, but it should not be the only reason to select a backend. Compare the documented input contract, upload support, ratio controls, latency, safety controls, response durability, retry semantics, data handling, and adapter maintenance. Keep the provider-specific code behind one interface so a change in that decision does not rewrite the Next.js form or the tenant ledger.

Ship one narrow workflow first: CRM action, server policy, queued job, private upload, one generated result, and a settled ledger entry. Then inspect real usage. The shortest path to a reliable image generator is not the shortest path to an image endpoint; it is the shortest path to knowing what happened for every tenant.

## References

- OWASP Top 10 for Large Language Model Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- openai/whisper — https://github.com/openai/whisper
