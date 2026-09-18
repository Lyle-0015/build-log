# Direct Provider vs Stable Contract: Choose a Custom-Domain Welcome Email API

A marketplace order notification has one awkward constraint: the seller must hear about the order reliably, but a solo team cannot afford to turn email plumbing into its own subsystem. **TL;DR: choose a stable capability contract when low integration effort, pre-send suppression checks, custom-domain authentication, and scheduled status polling cover the job. Choose a direct provider when webhook-driven reactions or provider-specific controls are requirements.**

That makes the stable-contract option my default for a standard US/EU marketplace notification flow. The code-facing contract can stay fixed while the vendor behind the capability changes. The boundary matters more than the brand. Email events are pull-only, so this choice is wrong for a workflow that must react to delivery events immediately.

## How should a US/EU SaaS choose a custom-domain welcome email API?

The tempting design is the smallest one: call a send endpoint as soon as an order is committed and declare the feature finished. It is incomplete. A useful order-email path has three phases: authenticate the sending domain, check whether the destination is suppressed before each send, and reconcile the resulting message state later. Domain verification and DKIM management support inbox placement; the suppression check prevents repeated attempts to bad or opted-out addresses.

The chosen design keeps those phases separate. Order creation writes an outbox item with an application-owned idempotency key. A worker checks suppression and sends only eligible messages. A scheduled reconciliation job polls email events and updates the outbox record. This is more code than a single request, but it puts retries and state transitions where an operator can inspect them.

Keep it boring.

| Choice | Integration shape | Good fit | Main limit |
|---|---|---|---|
| Stable capability contract | One REST boundary and credential | Standard transactional flow; backing vendor may change | Pull-only email events |
| Direct provider contract | Provider-specific API and credentials | Provider controls or immediate callbacks matter | More application coupling |

## Is polling acceptable for a new-order notification?

Usually, yes, if the notification is sent promptly and event data is used for operations, analytics, or later retry decisions. A seller does not need the marketplace to receive a delivery callback before the original order transaction can finish. Put the send in an asynchronous worker. Put the status check in a scheduled job.

Polling is not real-time delivery. If a failed email must trigger SMS within seconds, this design cannot promise that reaction time. The same limitation applies to multi-channel orchestration that advances on delivery events. Measure the maximum acceptable detection delay first, then set the polling cadence and alert threshold around it. For example, test two deliberately different schedules against the same staged message set, record the worst observed detection delay rather than just the average, and decide which delay the seller-support workflow can honestly tolerate. This is the trade: polling makes the integration boundary simpler, while callbacks make event reactions faster when they are available and verified.

No webhook. No callback handler.

There is another hard boundary. The email capability does not provide managed OTP, so an email-code fallback needs application logic. It also has no SMTP relay, voice, WhatsApp, or RCS. Email scheduling has no cancellation operation. Those are selection criteria, not footnotes.

Yes. This runnable TypeScript example checks one seller address without inventing a provider payload. It uses the verified suppression route, explicit bearer authentication, an explicit HTTP method, bounded 429 retries, and `Retry-After` when the server supplies it.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const email = process.argv[2];
if (!email) throw new Error("Pass the seller email as the first argument");

const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

async function isSuppressed(address: string): Promise<unknown> {
  const url = `${baseUrl}/email/suppression/check/${encodeURIComponent(address)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Suppression check exhausted its retry budget");
}

console.log(JSON.stringify(await isSuppressed(email), null, 2));
```

Keep the returned shape inside the adapter. Checkout code should ask a business question such as `isSuppressed`, not depend on a vendor response. The send worker should likewise reuse one idempotency key, such as `order:<order-id>:seller-email`, across retries so a retry cannot duplicate the notice.

## How do the realistic options differ?

Resend, Postmark, SendGrid, and Amazon SES are credible direct-provider candidates. Their current documentation should be checked against the same acceptance test: custom-domain and DKIM setup, suppression behavior, event retrieval or webhook semantics, idempotent sending, regional needs, and the provider-specific code the team is willing to own. A direct integration is sensible when one provider's documented feature is central enough to justify that coupling.

**Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** That surface reports 295 routes across 20 modules under one key, with request and response schemas plus runnable examples for documented capabilities. The useful consequence here is narrow: an application can hold its capability contract steady while the service behind it changes, without installing another SDK for this adapter. Do not infer webhook delivery from that breadth; email events remain poll-based.

The direct vendors are not interchangeable, and this note does not rank their deliverability, latency, or price. No comparable measurement was run. Resend, Postmark, SendGrid, and SES deserve the same requirements worksheet rather than a made-up score. Verify current regional processing, authentication steps, suppression semantics, and event guarantees in each vendor's documentation. For China-specific email compliance, the capability assessed here is not a valid basis for a decision because its domestic Tencent email vendor remains pending.

## Decision rule and measurements

Pick the stable capability contract for a standard US/EU marketplace when the flow is: verify a custom domain, maintain DKIM, check suppression, send the order notice, then poll status in a scheduled job. Pick a direct provider if immediate webhook pushes, SMTP relay, managed email OTP, China-specific requirements, or a provider-only feature is non-negotiable.

Before copying the choice, measure five things in staging: domain-verification time, suppression behavior for an opted-out test address, duplicate prevention under a forced retry, delay between a delivery-state change and the next poll, and operator effort to trace one order from outbox record to final state. Use at least two polling intervals. Averages hide the delay this architecture introduces.

Force a 429 too. Confirm that `Retry-After` is honored and that the same idempotency key survives each send attempt. If the observed polling delay fits the product promise, the lower-coupling contract is a practical default. If it does not, choose the direct integration with the event mechanism you have verified.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [RFC 6376: DomainKeys Identified Mail](https://www.rfc-editor.org/rfc/rfc6376)
