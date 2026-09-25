# Choose an Email API by Receipt Evidence (Custom Domain, No Webhooks)

A property-management receipt flow should choose its email API by the evidence it can retain after payment settles: authenticated domain control, suppression enforcement, stable message identifiers, and queryable delivery events. When webhooks are off-limits, polling can work, but only if the application owns a durable state machine rather than treating one successful API response as proof of delivery.

Short answer: require DKIM on a domain you control, test suppression behavior before launch, store the provider message ID beside the payment and receipt IDs, and poll until each message reaches a documented terminal or review state. Region labels and a pleasant SDK matter less than proving who requested the send, what was accepted, and what happened next.

## How should you choose an email API for a custom receipt flow?

Suppose a tenant pays invoice `inv_8421` for unit `4B`. The payment system settles it, the ledger posts it, and the receipt worker submits an email. Those are three different facts. An HTTP success from the email API proves only that the request was accepted under that API's contract; it does not, by itself, prove inbox delivery.

Acceptance comes first.

That distinction changes the selection process. I would start with a small evidence matrix and reject any candidate whose documentation cannot answer it unambiguously.

No evidence, no claim.

| Evidence question | Record owned by the application | Capability to verify in an API trial |
|---|---|---|
| Which business event caused the message? | Payment ID, receipt ID, tenant ID, template revision | Metadata or a correlation field survives retrieval |
| Which identity signed the mail? | Sending domain and selector configuration revision | Custom-domain DKIM can be verified in received headers |
| Was the recipient already ineligible? | Suppression decision and policy reason | Suppression status is queryable before or after submission |
| What happened after acceptance? | Provider message ID and normalized event history | Events can be fetched without a webhook |
| Where is evidence retained? | Internal audit record with retention class | Documented regional processing and retention terms |

The US/EU requirement needs the same precision. A vendor's company address, API hostname, or marketing region is not evidence of data location. Record the contracted processing region, subprocessors, transfer terms, deletion behavior, and which message fields appear in logs. Legal review determines whether those controls meet the application's obligations; an engineering checkbox cannot.

## The simple approach fails at the word "sent"

The tempting implementation stores `sent_at` immediately after a successful POST. It is compact, and it erases the distinction the audit needs. Acceptance, delivery, bounce, complaint, and suppression are different states. Conflating them creates a receipt record that sounds stronger than its evidence.

That is the trap.

Use an outbox row as the handoff between payment settlement and email submission. A unique constraint on the business event prevents two workers from creating two logical receipts. Keep retries behind an idempotency key when the API supports one; otherwise, reconcile an ambiguous timeout before resubmitting. A client timeout does not establish that the remote service rejected the first attempt.

Here is the narrow contract I want the application to own:

```ts
type DeliveryState =
  | "queued"
  | "accepted"
  | "delivered"
  | "suppressed"
  | "bounced"
  | "complained"
  | "review";

type ReceiptEvidence = {
  paymentId: string;
  receiptId: string;
  recipientHash: string;
  templateRevision: string;
  providerMessageId?: string;
  state: DeliveryState;
  observedAt: string;
};

interface MailEvidencePort {
  submit(input: {
    idempotencyKey: string;
    from: string;
    to: string;
    subject: string;
    html: string;
  }): Promise<{ messageId: string }>;

  inspect(messageId: string): Promise<{
    state: Exclude<DeliveryState, "queued">;
    occurredAt: string;
  }>;
}
```

The adapter can translate a provider's vocabulary into this deliberately small model. Preserve the raw response separately if policy permits, because normalization can discard detail needed during a dispute. Do not put full message bodies or plain recipient addresses into general-purpose logs merely because they are convenient.

## Polling is a control loop, not a timer

A fixed loop that asks about every message every few seconds will waste requests, amplify outages, and still leave unclear stopping conditions. The worker needs a due time, an attempt count, a lease so two workers do not poll the same row, and a review deadline. Add jitter to exponential backoff, honor documented rate-limit signals, and cap concurrency.

Stop automatically when the provider reports a terminal state defined in its documentation. Move unknown states, exhausted attempts, and records older than the evidence window to `review`; do not quietly relabel them `delivered`. This is the uncomfortable state, but it is honest.

Polling also creates a product-selection test that demos often skip: can one message be retrieved directly by its stable identifier, or does the integration have to scan a broad event feed? Direct lookup makes isolation and reconciliation easier. If only a feed exists, verify pagination order, cursor durability, event deduplication, and how long events remain available. The application should checkpoint a cursor only after committing the corresponding evidence records.

The trade-off is real. Polling is a poor fit when the application needs near-instant reactions at very high message volume, because repeated reads add load and terminal evidence arrives only after the next scheduled check. A documented event push model may fit that case better. Polling earns its place when inbound callbacks are prohibited or costly to secure, and when a bounded observation delay is acceptable; even then, the event-retention window must be longer than the worst credible outage of the polling worker.

Suppression needs its own preflight. Test an address already on the suppression list, observe whether submission is rejected or accepted into a suppressed state, and confirm that the result remains queryable. Never bypass a complaint or unsubscribe suppression just to force a receipt through; route the case to an approved alternate process instead. The exact policy depends on the message category and applicable law.

## DKIM proves domain responsibility, not message success

DKIM attaches a cryptographic signature to selected message headers and the body. The receiving system retrieves the public key from DNS and verifies the signature as specified by RFC 6376. That establishes responsibility by the signing domain for the signed content. It does not promise delivery, inbox placement, or that the recipient read the receipt.

For evaluation, use a subdomain dedicated to transactional mail and retain the DNS change record, selector, activation time, and verification result. Send test messages to accounts you control, then inspect the raw headers for a passing DKIM result and the expected signing domain. Also verify SPF and DMARC alignment as part of the domain's authentication posture; DMARC's identifier alignment and policy model are defined by RFC 7489.

Key rotation is where a paper checklist becomes operational work. The candidate must allow overlapping selectors long enough to publish and validate a new key before retiring the old one. Monitor DNS resolution from more than one network. A dashboard's green badge is useful, but received headers and DNS records are the evidence.

## A focused acceptance test beats a feature grid

Run the same controlled receipt through every candidate under consideration. Use synthetic tenant data and addresses your team owns. The trial should include one normal delivery, one pre-suppressed recipient, one hard bounce, a repeated submission using the same business key, an API timeout with reconciliation, and polling after the original access token is rotated.

Pass or fail the trial on artifacts, not impressions. Can an operator move from `paymentId` to the immutable receipt payload, submission attempt, remote message ID, and chronological observations without searching ad hoc logs? Can access to message content be separated from access to delivery metadata? Can records be deleted or retained according to the declared schedule?

Keep the scoring blunt. I would weight evidence completeness and suppression correctness above SDK ergonomics, then evaluate regional terms, latency under the expected workload, rate limits, exportability, and total operating cost. Price belongs in the model, but it is not the decision. A cheap request becomes expensive when a solo operator must reconstruct missing history during a payment dispute.

Before copying this design, measure the daily settled-payment volume, the delay from settlement to API acceptance, time to terminal observation, percentage entering `review`, duplicate logical receipts, suppression outcomes, polling requests per completed receipt, and evidence age at deletion. Those numbers reveal whether the control loop is affordable and whether its gaps are operationally manageable.

The final choice should be the service whose documented behavior survives that test and whose contract matches the required jurisdictions. Keep the provider behind the evidence port, keep business identifiers in your database, and make uncertainty visible. That is enough architecture for a receipt flow without turning email into the center of the product.

## Further reading

- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc5321
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
