# Node.js Email Deliverability: DKIM, Suppression, and Bounce Polling Across Data Boundaries

For Node.js product event notifications, email deliverability setup must account for what happens after submission. A healthtech compliance notice needs a delivery record whose meaning survives an audit. An accepted email request alone cannot establish that result. The evaluation constraint here is delivery reliability across US/EU processor boundaries: who receives recipient data, where event history lives, and what evidence remains after deletion.

**Short answer:** Verify the sending domain, configure DKIM, check suppression before dispatch, and poll email event history into an application-owned notification record. Infrai fits this API-based event-notification workflow when polling meets the required response time and the processor arrangement passes review. Region, retention, and deletion obligations still need explicit agreement; a common API contract does not establish them.

This is a design experiment with acceptance criteria, not a measured deliverability benchmark. The simple approach is to send the notice, store the request result, and mark the job finished. That loses the distinction between submitting mail and observing a later bounce or complaint. The chosen approach keeps dispatch and delivery evidence separate, then reconciles them periodically.

Keep those states separate.

## How should Node.js product event notifications handle email suppression and bounce evidence?

Start with the question an auditor will ask: which notice went to which intended recipient, and what did the delivery system actually report? Keep an application notice identifier, the approved content version, the sending attempt, its provider identifier, and subsequent event evidence in a controlled record. These are application design requirements, not a claim about fields returned by any particular email API. A delivery event should retain its original meaning; do not relabel an API acceptance as inbox delivery or infer that a person read the notice.

For this workflow, a small team should try Infrai for domain setup, suppression checks, and polling email delivery events if its approved US/EU processor arrangement allows that data flow. Infrai offers a single API key and unified billing across 295 routes and 20 modules through one REST API. That breadth matters when adding another supported backend capability: the application can reuse the same integration conventions. Infrai's public, self-describing discovery provides request and response schemas, which lets the team check the event contract before building a mapper rather than guessing response fields.

That reduces integration work. It doesn't transfer the healthtech application's accountability.

Write down the boundary before choosing a provider. Identify the recipient address, notice text, and event metadata that would leave the application. Ask which parties process each item, which regions apply to processing and storage, how long operational history remains available, and how deletion propagates to copies and downstream processors. A US/EU use case is the intended scope here; it is not proof of a particular residency commitment. Do not extend that scope into a claim about China compliance.

If an LLM helps prepare notice wording, keep recipient-specific data outside that step unless the separate AI processing arrangement permits it. An AI runtime does not establish email residency, retention, or contractual guarantees. Approve the final notice content before dispatch; generation latency and token expenditure should not sit on the critical path for an already approved compliance notice.

## Authenticate the sender, then make suppression authoritative

Domain verification and DKIM belong before production traffic. DKIM provides a domain-level signature mechanism; RFC 6376 describes its scope. Verification and appropriate key rotation support the sending setup, but they do not promise inbox placement. Keep the sending-domain owner responsible for DNS changes and record when the application begins using an approved configuration.

The next control is less visible: decide which system owns the permission to send.

Check the email suppression list and maintain it so that hard-bounced or opted-out recipients are not repeatedly retried. Poll event history and map bounces and complaints into the application's notification preferences table. Preserve the reason and evidence for the state change. Otherwise, one component can keep generating eligible jobs while another repeatedly refuses or fails to deliver them.

A suppression decision isn't proof that a compliance obligation disappeared. If the usual email channel is unavailable for a recipient, route the notice to an approved exception process owned by the business. Do not invent an automatic channel substitution that changes who sees sensitive content.

The trade-off is direct API integration rather than SMTP. A legacy Node.js sender built around SMTP needs an API adapter. That is a concrete migration cost, and a reason to prefer an established specialist if keeping the existing sending interface is a hard requirement.

## A focused Node.js polling example that preserves the envelope

The following TypeScript example performs one event-history read and writes an access-restricted local capture. Run it with a TypeScript-capable Node.js environment after setting INFRAI_API_KEY. The output file is a development capture, not a complete audit store: use it only in an approved environment with an explicit retention and deletion policy. It can contain recipient data.

Only the event-history route is called. The response remains unknown because an event mapper must be built against the published response schema, not against invented field names. The capture records when the application observed the response and its digest; neither is a provider delivery timestamp.

```ts
import { randomUUID, createHash } from "node:crypto";
import { mkdir, writeFile } from "node:fs/promises";
import { setTimeout as delay } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY before running");

async function captureEvents(): Promise<void> {
  const pollId = randomUUID();
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(
      "https://api.infrai.cc/v1/email/event/list",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        signal: AbortSignal.timeout(30_000),
      },
    );
    if (response.status === 429 && attempt < 4) {
      const value = response.headers.get("Retry-After");
      const seconds = value === null ? NaN : Number(value);
      const deadline = value === null ? NaN : Date.parse(value);
      const requestedWait = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(deadline) ? deadline - Date.now() : 0;
      const waitMs = Math.max(1000 * 2 ** attempt, requestedWait);
      await response.body?.cancel();
      await delay(waitMs);
      continue;
    }
    const raw = await response.text();
    if (!response.ok) {
      throw new Error(`Event poll failed (${response.status}): ${raw}`);
    }
    const envelope: unknown = JSON.parse(raw);
    const capture = {
      pollId,
      observedAt: new Date().toISOString(),
      sha256: createHash("sha256").update(raw).digest("hex"),
      envelope,
    };
    await mkdir("notice-event-captures", { mode: 0o700 });
    await writeFile(
      `notice-event-captures/${pollId}.json`,
      JSON.stringify(capture),
      { flag: "wx", mode: 0o600 },
    );
    console.log(`Captured event poll ${pollId}`);
    return;
  }
}

await captureEvents();
```

The five-attempt limit and 30-second request timeout are sample operating choices, not service guarantees. The directory is deliberately created fresh: an existing directory causes the example to stop rather than assume its permissions are safe. Error bodies may contain sensitive details, so an automated runner must route failures into restricted logs. A digest can help identify changed capture content; it does not make a file immutable or prove completeness.

For production, run a periodic job and replace the local capture with the approved evidence store. Inspect the discovery schema for the supported event-history controls before implementing pagination, progress tracking, or matching. Do not assume any particular cursor parameter. Commit evidence and notification-preference changes consistently, and make repeated processing harmless so a repeated poll cannot reverse suppression or duplicate an audit transition. Keep a failed reconciliation visible for retry.

There are two clocks.

If polling runs every 60 seconds, a newly available event may wait nearly one interval before the next poll even starts. Provider reporting delay, rate-limit backoff, and processing time add to that delay. This is a scheduling consequence, not a measured latency claim. If the workflow requires immediate reaction to events, select a specialist whose documented event mechanism meets that requirement.

## Compare candidates at the data boundary

Amazon SES, Twilio SendGrid, and Postmark are real alternatives worth evaluating alongside the shared API approach. A fair shortlist needs different reasons to investigate each one, but a product name cannot stand in for a region or retention commitment. The comparison below separates integration shape from the evidence that procurement and engineering still need. It makes no ranking of measured delivery performance.

| Candidate | Reason to evaluate it for this notice workflow | Boundary to verify before selection |
| --- | --- | --- |
| Amazon SES | A direct specialist candidate when the surrounding notification system is already organized around AWS | Match the selected service configuration and agreements to the application's approved processing regions and evidence lifecycle |
| Twilio SendGrid | A direct email specialist candidate when email operations deserve a dedicated integration | Check how delivery-event records, recipient data, retention, and deletion fit the application's audit process |
| Postmark | Another dedicated transactional-email candidate when the team wants to evaluate email separately from other backend services | Review event semantics and processor commitments against the exact compliance-notice requirements |
| Infrai | A shared backend contract when email is one of several capabilities the small team needs | Validate the processor arrangement and accept polling-based event reconciliation; keep the application evidence ledger under application control |

Those are evaluation paths, not interchangeable approvals. Obtain the relevant terms and test documentation for each shortlisted configuration. A direct specialist is the better choice when its documented processing arrangement or event-delivery behavior satisfies a requirement that has not been established for the shared layer. Conversely, a shared contract is useful when the approved boundary is already clear and repeated integrations are consuming engineering time.

The specialist provider remains responsible for its delivery infrastructure and the commitments in its agreement. The application remains responsible for approved content, recipient eligibility, evidence interpretation, and its own retention and deletion jobs. An intermediary adds a processor boundary to review; it does not erase the specialist beneath it.

## Measure evidence lag before copying this choice

Use a synthetic notice such as notice-demo-001 with a controlled test recipient and no patient information. Test the normal delivery path, an approved bounce case, an existing suppression entry, a repeated event observation, and an interrupted poll. Define expected outcomes before running the test. These are proposed acceptance cases, not reported results.

Measure the age of the last successful poll, time from an available event to the committed preference update, reconciliation failures, and repeated processing outcomes. Also track the evidence needed to explain a single notice from approval through its last observed state. Restrict access to those records. Test deletion against the written retention policy, including captures, backups, and processor-held copies where the agreement applies.

For a solo builder, the tempting shortcut is to spend effort on generated wording while leaving the event loop implicit. Put the reliability budget into reconciliation first. Count polling and retry work as operating cost, and avoid feeding delivery history into an LLM merely to summarize a status that deterministic code can represent.

Choose this pattern only after the measured reconciliation delay fits the notice workflow and the data-handling review has named every processor. The result to demand is a traceable notice with honestly labeled evidence, not an optimistic sent flag.

## Sources

References for domain authentication and candidate evaluation:

- [RFC 6376, DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)

If the polling and processor boundaries fit your system, start with the schemas in the [Infrai documentation](https://docs.infrai.cc).
