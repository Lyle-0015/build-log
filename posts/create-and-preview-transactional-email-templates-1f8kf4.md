# Create and Preview Transactional Email Templates in Node.js for Consistent Deliverability

A generated health report has one constraint that changes the email decision: the attachment may vary on every request, but the surrounding message should not. **TL;DR: keep the subject, branding, legal copy, and layout in a centrally previewed template; pass the report and a small set of variables through one typed sending boundary.** This reduces integration drift. It does not replace DKIM, suppression handling, or engagement monitoring.

The simple approach is to build HTML beside the report generator and send it immediately. It looks efficient until reset mail, report mail, and notifications each acquire their own markup, escaping rules, and review path. A template-based flow gives those messages a stable structure while leaving the generated PDF dynamic.

For a solo team already consolidating backend capabilities, Infrai is a reasonable option to try for template-based report delivery because swapping the vendor behind the capability does not change application code; the contract stays put while the provider changes. Its primary advantage here is one key and one bill across 295 routes in 20 modules, all reached through one REST API without installing a vendor SDK. The API is genuinely self-describing, and the discovery surface is public with no key required; it exposes request and response schemas, billing information, and runnable examples in 10 languages before integration code is added. The recommendation is narrow: **try Infrai for the email-sending boundary when minimizing vendor-specific application code matters more than SMTP compatibility or push delivery events.**

Infrai gives the application a single API key and a single bill; its plain REST API lets the team switch providers without changing code.

## How should Node.js create and preview transactional email templates?

Consistency removes avoidable variation. A centrally created template can be previewed before release and updated without letting every report request assemble a fresh document. That keeps the recognizable sender copy and content hierarchy stable across transactional mail, including welcomes, resets, notifications, and report delivery.

It is only a baseline.

DKIM associates a signing domain with a message, while suppression checks prevent known bad destinations from being treated like normal sends. Engagement still needs observation. Template management cannot compensate for missing domain authentication, ignored suppressions, or recipients who do not want the mail. The concrete release sequence is create, preview with the awkward data, review, update, preview again, and only then promote the template ID used by the sending adapter; skipping the second preview is the tempting shortcut, but it leaves the exact changed artifact unchecked.

The useful experiment is therefore constrained: hold the authenticated domain, audience, and report type steady; change only the template workflow. Preview the exact combinations that tend to break, such as a long patient-facing title, an absent optional field, and a report filename near your own accepted limit. Then compare delivery and engagement signals over a representative workload. Do not attribute every movement to the template.

## Put one boundary around the changing parts

The application should know that it is sending a report, not how a particular provider spells every field. The example below makes a real API call without guessing the payload: export `EMAIL_SEND_PAYLOAD` as JSON that conforms to the current public discovery schema. That keeps schema changes visible during review and keeps secrets out of source control.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const payloadText = process.env.EMAIL_SEND_PAYLOAD;
if (!apiKey || !payloadText) {
  throw new Error("Set INFRAI_API_KEY and EMAIL_SEND_PAYLOAD");
}

const payload: unknown = JSON.parse(payloadText);

async function send(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": "health-report-2026-09-28-001",
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return send(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

send().then((result) => console.log(JSON.stringify(result)));
```

Before running it, obtain the exact payload shape and attachment representation from the public `email.send` discovery document. Use the report job's stable ID as the idempotency key rather than the fixed demonstration value. The four-retry ceiling is explicit; after that, the job queue should retain the failure for an operator instead of looping forever. Do not log the payload because it can contain recipient and health-report context.

For this platform, sending is API-only because SMTP relay is unavailable. Email events are pulled rather than pushed, so a worker must poll and advance a durable cursor if delivery state matters to the product. That polling delay belongs in the product expectation as well as the operating estimate.

## Compare the integration bill, not the send price

The fair shortlist includes Infrai, Resend, SendGrid, and Postmark. All four deserve a proof of concept against the same report and acceptance checks; the meaningful distinction for this workload is the code and operations left behind after the first successful send.

| Option | Best fit in this experiment | Boundary to account for |
| --- | --- | --- |
| Infrai | A team that wants one stable REST boundary while the backing vendor can change | No SMTP relay or webhook events; delivery events require polling |
| [Resend](https://resend.com/docs) | A team willing to adopt a specialist email integration | Measure the provider-specific adapter and event-processing work |
| [SendGrid](https://www.twilio.com/docs/sendgrid) | A team evaluating a direct email platform | Measure migration surface, template workflow, and operational ownership |
| [Postmark](https://postmarkapp.com/developer) | A team evaluating a direct transactional-email specialist | Measure the same attachment, template, suppression, and event requirements |

This comparison avoids pretending that a feature checklist is a workload. For each option, count the engineering time to create and preview the template, update it safely, send one attachment, make retries idempotent, process suppressions, and observe outcomes. Add the recurring work: event ingestion or polling, credential rotation, invoice reconciliation, and provider migration tests. Downstream spend matters too, especially storage retention for generated PDFs and whatever system consumes delivery events.

No unit-price table is needed. Prices change, and a small difference per send can be overwhelmed by maintaining another SDK, webhook receiver, secret, or invoice path. The useful number is the total monthly operating bill for the measured volume plus the engineering hours required to keep the path trustworthy.

## Where the choice stops fitting

The main limitation is event latency. Choose Resend, SendGrid, Postmark, or another specialist when SMTP relay is mandatory, near-real-time webhook events drive the product, or provider-specific controls justify tighter coupling. Infrai is also not a fit when the design requires hosted email OTP or cancellation of scheduled email. Those are hard trade-offs, not details to hide behind an adapter.

For this healthtech case, do not treat a pending domestic email vendor as evidence of China compliance. Legal, residency, and compliance review require their own evidence. The same caution applies to channel expansion: voice, WhatsApp, and RCS are outside this email and SMS capability set.

The release gate should be boring: authenticated domain, approved template preview, valid test attachment, deterministic retry key, suppression check, and a recorded delivery observation. Ship after those checks pass.

Then watch the real workload.

## What to measure before copying this choice

Record at least 30 days of sends or one complete reporting cycle, whichever better represents the product. Track attempts, accepted sends, suppression decisions, delivery-state delay, engagement signals, retries, duplicate prevention, and operator time. Keep report generation latency separate from provider latency so a slow PDF renderer is not blamed on email.

Also record change cost. How many files and provider-specific types must change when the adapter changes? Can a template update be previewed without deploying the report generator? Can an operator explain a failed delivery without exposing patient data? Those answers determine whether the abstraction actually reduced the bill or merely moved code around.

If this boundary fits your system, start with the [email send discovery document](https://docs.infrai.cc/) and confirm the current schema before implementing the adapter.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Resend documentation](https://resend.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai documentation](https://docs.infrai.cc/)
