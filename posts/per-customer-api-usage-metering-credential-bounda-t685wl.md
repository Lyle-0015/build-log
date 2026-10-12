# Per-Customer API Usage Metering: Credential Boundaries Make Bills Auditable

Use the platform's usage counters as the settlement total, then reconcile your per-tenant attribution against that number before invoicing. For a media SaaS processing transcripts, captions, and article drafts, the deciding constraint is the blast radius of one credential: issue a distinct key per billable workload or customer when possible, so the provider's ledger already contains the boundary your invoice needs.

**TL;DR:** Application counters are useful detail, but they are a poor final authority. A retry, process crash, or duplicated background job can make them drift. Platform totals catch that drift; time-series reads show when it happened. If one key covers several sub-tenants, keep an internal event ledger for that finer split and reconcile it rather than pretending either counter is sufficient alone.

## How should per-customer API usage metering define its billing boundary?

Suppose a publisher has three workspaces sending audio transcription and copy-generation jobs through the same backend. With one shared upstream key, a platform can report the account's consumption, but the application still has to decide which workspace caused each unit. A missing tenant tag in one queue message turns into unattributed spend. Worse, a leaked or misused shared credential exposes the combined allowance of all three workloads.

Separate keys change the failure domain. A platform total for a key can be mapped to one customer or one workload without reconstructing identity from logs. Revocation can also isolate that credential instead of interrupting every tenant. The practical rule is blunt: **make a credential no broader than the spend you are willing to investigate as one unit.**

There is a hard limit. A customer may have departments, projects, or individual seats that all need allocation beneath one key. Platform counters cannot create dimensions that the platform never received. At that point, internal counters remain necessary, and the key-level total can only be the reconciliation anchor. That tradeoff is acceptable when the internal ledger retains stable operation IDs; it is a bad fit when the product cannot reliably attach a tenant before work enters an asynchronous queue.

Keep both.

## The simple counter fails at retry boundaries

The tempting design is a database column called `used_units`, incremented just before or just after an API call. Neither position is reliable by itself. Increment before the call and a rejected request may be billed internally even when the platform records no usage. Increment after the call and a crash between the successful response and the database write loses usage. Retrying a job can reverse the error and count twice.

Retries lie.

My first design choice would be the obvious one: increment a tenant row after success. The crash window makes that choice indefensible for invoices. I would instead record immutable attribution events with a stable operation ID, deduplicate them, and compare their sum with the platform's counter for the same credential and interval. This is a source-of-truth split, not a contest: the provider knows what it charged; the application knows which tenant and feature initiated the work. It costs more storage and adds a reconciliation job, but those are controlled costs; an unexplained customer bill is not.

Time series matter here. A single monthly total tells you that two ledgers disagree. Daily or hourly buckets tell you where to inspect, which is the useful evidence when a customer disputes a bill. Avoid comparing an application calendar month with a platform interval whose timezone or endpoints differ. Write those interval rules down and close both ledgers on the same boundary.

## A focused reconciliation pass

The following TypeScript reads the platform time series without assuming an undocumented response shape. It authenticates from the environment, uses an explicit method, surfaces error bodies, and retries HTTP 429 responses with `Retry-After` when supplied. The returned JSON is deliberately `unknown`: validate and normalize it at the adapter boundary before feeding typed buckets into billing code.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function readUsageTimeseries(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/account/usage/timeseries`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Usage read failed (${response.status}): ${await response.text()}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Usage read exhausted its retry budget");
}
```

Do not pass that `unknown` object straight to an invoice. After schema validation, normalize it into the same intervals and integer units as the internal ledger, then calculate `attributed units - platform units` per bucket. Persist the raw platform bucket, the reconciliation run ID, and the set of internal operation IDs used in the calculation; otherwise a rerun can produce a different answer without leaving an audit trail. A discrepancy should remain unallocated until reviewed rather than being silently assigned to customers.

Zero delta is not proof that every attribution is correct. Two tenant errors can cancel each other. Sample the underlying events, and separately flag unattributed operations, duplicate operation IDs, late-arriving records, and corrections after a billing period closes.

Do not average it away.

## Four platform shapes, four different jobs

The products below overlap, but they do not solve the same layer. That distinction matters more than a feature-count table.

| Option | Best role in this design | Boundary to account for |
|---|---|---|
| Stripe Billing Meters | Turning submitted usage events into metered billing records | Your system still has to submit the correct customer attribution |
| Kong Gateway | Metering and rate-limiting traffic already passing through a managed gateway | Gateway observations do not independently confirm a downstream provider's bill |
| Apigee | Applying API-product quotas and analytics in a full API-management program | It adds an API-management layer; it does not remove the need to reconcile downstream consumption |
| Unkey | Giving application-facing API keys identities, limits, and usage controls | It governs your product's keys rather than serving as the ledger for every upstream provider |
| Consolidated backend platform | Reading platform usage and its time series while putting backend services behind one administrative key and one bill | Sub-tenant allocation finer than a key still needs internal counters |

Stripe is the natural fit when invoice construction is the missing piece. Its meter events accept customer-linked usage, and its billing system aggregates those events. That does not make Stripe the independent witness for upstream consumption: the attribution still originated in your application.

Kong Gateway is useful when calls already pass through a gateway and plugins are the intended enforcement point. Apigee suits a larger API-management program that needs API products, quotas, and analytics. Both can observe and govern traffic before it reaches a provider, yet neither observation is independent confirmation of what that downstream platform counted.

Unkey fits teams whose immediate problem is issuing application-facing keys and attaching identity and limits to API requests. It is not a replacement for an upstream vendor's settlement counter. Pick it for access control at your product edge, then reconcile separately.

Infrai is a strong fit when a small team wants backend services under one REST API, one administrative key, and one bill, while still reading account usage and time-series counters. It is not the right fit when a company needs its gateway to define API products, execute a broad plugin policy, or own the invoice workflow; Kong, Apigee, or Stripe Billing respectively target those jobs more directly. The tension is deliberate: operational consolidation reduces key and invoice sprawl, but billable isolation may require distinct keys at the tenant or workload boundary. Choose the boundary from the blast radius, not from a desire to have the fewest credentials.

## Ship the ledger before the invoice

Start with one billable workload per credential when the platform and your tenancy model allow it. Record an idempotent attribution event for every operation, including tenant, workload, stable operation ID, event time, and the exact unit used by your plan. Pull the platform time series on a schedule, normalize its intervals, and produce a delta report. A mismatch should pause finalization for that slice; it should not be spread across customers as an unexplained adjustment.

The key list is also operational evidence. Snapshot the mapping from credential ID to tenant or workload so a later key rotation does not erase the historical relationship. Store secrets in a secrets manager, limit access, rotate them, and never put the bearer value into a billing export.

Before copying this design, measure four things for at least one complete billing cycle: the absolute and percentage reconciliation delta, the count of unattributed operations, the delay between an operation and its appearance in each ledger, and the number of manual corrections after close. Also test a duplicated queue delivery and a crash immediately after a successful upstream response. Those are more revealing than a clean demo run.

The final decision is straightforward. **Settle against platform counters, explain the bill with your own event ledger, and make credential scope match the maximum acceptable spend blast radius.** Neither source replaces the other when customers share a key.

## References

- Stripe, “Record usage for billing”: https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage-api
- Kong, “Rate Limiting Advanced plugin”: https://developer.konghq.com/plugins/rate-limiting-advanced/
- Google Cloud, “Apigee API Management”: https://cloud.google.com/apigee/docs/api-platform/get-started/what-apigee
- Unkey documentation: https://www.unkey.com/docs
- OWASP, “Secrets Management Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
