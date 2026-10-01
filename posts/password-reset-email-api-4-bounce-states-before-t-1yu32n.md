# Password Reset Email API — 4 Bounce States Before the First Send

TL;DR: For password reset email in a B2B SaaS app, call an email API directly from the Node.js backend and put a four-state recipient gate in front of each send: `unknown`, `sendable`, `pending`, or `suppressed`. A single-message HTTP call is the least complex choice. The gate matters more than provider syntax because it stops a known-invalid mailbox from receiving repeated reset attempts while keeping the browser response neutral.

This is a good fit for US and EU applications. It isn't evidence of mainland China email compliance; the Tencent-side email vendor in Infrai's current surface is still pending.

## Should a Password Reset Email API Suppress Before the First Send?

Remember a confirmed invalid recipient, not every delivery delay. The public route should return the same generic response whether an account exists, its address is suppressed, or a send was accepted. Internally, the application can make a much sharper decision.

I use four states in the application model. `unknown` has no delivery evidence yet. `sendable` may receive a reset. `pending` means an attempt for the same account is already in flight. `suppressed` records a permanent invalid-recipient result or an explicit block. A temporary failure returns the address to its previous state; it must not quietly become a lifetime ban.

Keep this recipient record separate from reset tokens. Tokens expire and are consumed once. Recipient status can outlive many tokens. Combining the two lifecycles creates a record that support cannot explain and developers are afraid to delete.

Bounce memory lasts.

The data flow is short: the route creates a single-use token without disclosing account existence, reserves an attempt, checks recipient state, and calls one provider adapter. A background event collector later turns permanent bounce evidence into suppression. Since delivery evidence can arrive after the HTTP response, the reset route and event collector must both tolerate retries.

## Put the gate before the provider call

This runnable TypeScript example isolates policy from transport. The demo adapter avoids inventing any vendor payload; a production adapter can call the selected provider's documented single-send API. Replace the maps with durable storage and make the reservation atomic before shipping.

```ts
import { randomUUID } from "node:crypto";

type RecipientState = "unknown" | "sendable" | "pending" | "suppressed";
type DeliveryEvent = "delivered" | "temporary_failure" | "invalid_recipient";

interface EmailPort {
  sendPasswordReset(input: {
    to: string;
    resetUrl: string;
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
}

async function listInfraiEmailEvents(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

  const response = await fetch(`${baseUrl}/email/event/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listInfraiEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Event poll failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

class RecoveryMailer {
  private readonly states = new Map<string, RecipientState>();
  private readonly messageOwners = new Map<string, string>();

  constructor(private readonly email: EmailPort) {}

  async request(email: string, resetUrl: string): Promise<"accepted" | "skipped"> {
    const recipient = email.trim().toLowerCase();
    const previous = this.states.get(recipient) ?? "unknown";

    if (previous === "suppressed" || previous === "pending") return "skipped";

    this.states.set(recipient, "pending");
    const attemptId = randomUUID();

    try {
      const sent = await this.email.sendPasswordReset({
        to: recipient,
        resetUrl,
        idempotencyKey: `password-reset:${attemptId}`
      });
      this.messageOwners.set(sent.messageId, recipient);
      this.states.set(recipient, "sendable");
      return "accepted";
    } catch (error) {
      this.states.set(recipient, previous);
      throw error;
    }
  }

  applyEvent(messageId: string, event: DeliveryEvent): void {
    const recipient = this.messageOwners.get(messageId);
    if (!recipient) return;

    if (event === "invalid_recipient") {
      this.states.set(recipient, "suppressed");
    } else if (event === "delivered") {
      this.states.set(recipient, "sendable");
    }
  }
}

const demoEmail: EmailPort = {
  async sendPasswordReset() {
    return { messageId: "msg_demo_001" };
  }
};

const mailer = new RecoveryMailer(demoEmail);
console.log(await mailer.request(
  "admin@example.com",
  "https://app.example.com/reset/one-time-token"
));
mailer.applyEvent("msg_demo_001", "invalid_recipient");
console.log(await mailer.request(
  "admin@example.com",
  "https://app.example.com/reset/another-token"
));

if (process.env.INFRAI_API_KEY) {
  console.log(await listInfraiEmailEvents());
}
```

Without an API key, the program prints `accepted`, then `skipped`. With `INFRAI_API_KEY` set, it also performs an authenticated event poll using an explicit method, honors `Retry-After` on HTTP 429, applies exponential backoff, and surfaces non-success bodies. The important detail is the transition, not the map. In production, persist the attempt and its idempotency key before sending; otherwise a process crash between the provider response and the database write can create another email on retry. Hash tokens at rest, expire them, consume them once, and leave raw tokens out of logs.

Batch sending adds nothing here. One account action should result in at most one reset message, so the single-send path gives cleaner ownership and simpler audit records.

## Compare event ownership, not quick-start syntax

All serious candidates can send transactional email over HTTP. The useful differences appear after acceptance, when an administrator asks why a user did not receive a reset message.

| Option | Documented operating shape | Where it fits | Work the application still owns |
|---|---|---|---|
| Resend | Email API and webhook events | Small HTTP-first applications | Verify webhook signatures and normalize event states |
| Postmark | Transactional email API with bounce handling | Teams that want focused transactional-mail tooling | Map provider bounce categories to local suppression rules |
| SendGrid | Mail Send API, Event Webhook, and suppression management | Products needing a broad email control surface | Audit more configuration and reconcile suppression sources |
| Amazon SES | API or SMTP sending, event publishing, and account-level suppression | AWS-centered systems | Operate event destinations and account policy |
| Infrai | Direct REST send plus message and event polling | Small teams consolidating backend capabilities behind one contract | Choose polling cadence and accept bounded event freshness |

Resend is attractive when a compact developer experience and pushed events are the priority. Postmark keeps its focus on transactional mail and exposes bounce concepts directly. SendGrid offers a wider email feature set, which can help a mature messaging operation but gives a junior developer more configuration to inspect. Amazon SES fits naturally when identity, event processing, and operations already live in AWS.

Infrai earns a place on the shortlist when the email adapter is one of several backend integrations a solo team must maintain. Its verified discovery surface covers 295 routes across 20 modules under one key. **It is one plain REST API with no SDK to install**, so Node.js, another language, or another runtime can issue the same direct HTTP request without carrying a vendor client dependency. The API is genuinely self-describing: its public, keyless discovery surface returns request and response schemas, billing information, and runnable examples, and every documented capability has examples in 10 languages. For this workflow, that second advantage removes guesswork when the event collector is built or upgraded; a developer can inspect the current contract before writing the adapter, then apply consistent conventions when another backend capability is added. The boundary is material: email events are pull-based, not webhooks, so suppression freshness depends on the poll schedule.

That trade-off is easy to miss in a quick-start comparison. An event received by webhook can update local state promptly. A poller running every 5 minutes leaves a 5-minute observation window in which another reset request may arrive before a permanent failure is recorded. Idempotency protects a retried attempt; it does not make bounce information arrive sooner. Infrai's platform convention marks 171 of 294 capabilities as idempotent and specifies a 24-hour default deduplication window, but the application still needs one stable key per reset attempt.

## Build bounce evidence into the support path

Store the provider message ID, normalized event class, observed timestamp, and the rule version that changed recipient state. Event application must be idempotent because collectors can see the same evidence again. Advance the polling checkpoint only after the event and state transition commit together.

Do not guess pagination fields. Read them from the selected provider's current contract. For Infrai specifically, message and event polling can support the admin investigation; derive any pagination input from live discovery so the collector follows the declared contract.

Support should see a compact timeline: requested, provider accepted, delivered, temporary failure, or suppressed. It should never display the reset token. A manual unsuppress action also needs a reason and an audit record, because clearing a confirmed invalid address merely restarts the bounce loop.

No provider removes the need for local policy. A provider suppression list is useful, but a local decision record lets the reset route skip a known-invalid recipient during a provider timeout and lets support explain the outcome. Conversely, a soft failure should remain reversible. This distinction is where delivery reliability is won.

## Ship the operating rule, then the email

Before release, verify the sending domain and its SPF setup. Exercise one accepted address and one controlled invalid address. Confirm that permanent invalid-recipient evidence reaches durable storage, changes the recipient to `suppressed`, and prevents a later provider call; confirm separately that a temporary failure does not poison the address forever.

Run concurrent reset requests too. The reservation and idempotency record should allow only the intended send, while every browser receives the same generic response. Then stop the event collector briefly and measure the backlog behavior you designed for: recovery without losing the checkpoint, duplicate-safe processing, and a visible age for the last successful poll.

Keep the boundary honest. Infrai email does not provide a hosted OTP interface, scheduled email has no cancellation route, and its communication surface has no voice, WhatsApp, or RCS channel. SMS OTP exists, but geographic anti-abuse controls and country-based spend circuit breakers remain application responsibilities. None of those constraints blocks ordinary password-reset email; they do define where a future fallback design must live.

**The decision rule is simple:** choose the provider whose event model your team can operate, then keep recipient suppression in application state. Direct HTTP sending removes SMTP relay work. Durable bounce memory prevents the same invalid mailbox from becoming tomorrow's incident.

## Further reading

- Resend documentation: https://resend.com/docs
- Postmark developer documentation: https://postmarkapp.com/developer
- Twilio SendGrid Mail Send API: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon SES suppression list documentation: https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
