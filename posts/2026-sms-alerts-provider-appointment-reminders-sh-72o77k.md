# 2026 SMS Alerts Provider — Appointment Reminders, Shipping, and Account Activity Evidence

TL;DR: For a US/EU game signup flow, choose the messaging setup that can prove why a verification message was sent, which template produced it, and whether the recipient was suppressed at decision time. A simple REST call is useful, but the durable design is an application-owned evidence record spanning the account, the email verification link, and any SMS fallback. Clerk plus Resend plus Twilio gives each layer a focused provider. A single-key API can remove credential and suppression glue, but it concentrates trust, billing, and outage exposure in one platform.

The failed-simple approach is to create an account, fire a verification email, and send an SMS whenever the email appears slow. That creates messages, not evidence. It cannot later answer whether the player consented to the SMS category, whether the chosen template was approved for that destination, or whether a retry crossed a suppression change.

The decision axis here is compliance evidence, not the shortest quickstart. For an indie team, every extra control plane also becomes maintenance work that competes with the game itself.

## What Should an SMS Alerts Provider Prove for Appointment Reminders and Shipping?

Start with one immutable attempt ID. Attach the account ID, destination region, purpose, consent or lawful-basis reference, template ID and version, suppression-check result, provider request ID, and final observed status. Store timestamps for the decisions, not just for the sends. A later status lookup cannot reconstruct what the application knew when it authorized a message.

This distinction matters in a verification flow. The email contains the link, while SMS is a fallback alert that tells the player to return to the signup session; it should not silently become a second authentication channel. The evaluated combined API has no managed email OTP endpoint, so an email-code fallback would remain application logic. It also has no voice, WhatsApp, or RCS channel. Do not draw those branches on an incident runbook as if they already exist.

For US and EU delivery, the business layer still needs its own geographic allowlist and country-level spend breaker. A provider endpoint does not settle the legal basis for contacting a particular player, and it does not replace destination-specific registration or sender requirements. Keep the policy decision close to the account event and persist its output.

The evidence object can stay small:

```ts
type VerificationEvidence = {
  attemptId: string;
  accountId: string;
  purpose: "signup_verification";
  region: "US" | "EU";
  channel: "email" | "sms";
  templateId: string;
  templateVersion: string;
  policyRef: string;
  suppressionCheckedAt: string;
  providerRequestId?: string;
  observedStatus?: string;
};
```

It is deliberately boring. Good audit data usually is.

## The provider comparison is really an ownership comparison

All four choices can sit in a production architecture, but they leave different evidence work with the application. The relevant comparison is the boundary, not a feature-count contest.

| Option | Natural role in this flow | Evidence and integration consequence | Boundary to accept |
|---|---|---|---|
| Clerk | Account and verification lifecycle | Keeps identity events near the user record | Messaging policy outside its managed flow still needs an application ledger |
| Resend | Transactional verification email | Gives email its own delivery surface and event model | SMS suppression and fallback decisions live elsewhere |
| Twilio | Transactional SMS and messaging controls | Provides a dedicated SMS control plane | Account state and email evidence must be correlated across providers |
| Combined API | One REST surface for account, email, and SMS operations | One API key covers all three capabilities; public, self-describing discovery includes schemas, billing, and runnable examples | Not a fit when independent channel failure domains or webhook events are required |

The focused stack is credible. Clerk + Resend + Twilio means three signups, three credential sets, and glue that maps the account ID to two messaging request IDs. The team must also define what happens when one system suppresses a recipient and another does not. That independence can be a benefit: separate vendors reduce the chance that one control-plane incident blocks every channel, and each product can be replaced on its own schedule.

The combined option makes a different trade. Infrai uses one API key and one bill across account, email, and SMS operations, so the signup evidence job doesn't have to reconcile three credential identities with three billing records. Its discovery surface reports 295 capabilities across 20 modules, and each documented capability has runnable examples in 10 languages. Reading one discovery response before wiring a capability is a practical advantage for a small team: the schema, billing description, and example travel together, without requiring another SDK. Its idempotency convention has a 24-hour default deduplication window. Suppression management covers blocked or opted-out recipients, while direct sends and templates standardize repeated account messages.

The limitations are concrete. I wouldn't choose this combined option when independent channel failure domains, webhook-driven status transitions, managed email OTP, or conversational messaging are launch requirements; Clerk, Resend, and Twilio are the clearer split in that case. My rule for a small launch is to accept consolidation only when the evidence ledger remains portable.

There is a catch. Both messaging namespaces use polling rather than webhook events, which limits real-time multichannel orchestration. Template IDs and business mappings should remain in application config or an admin database because SMS template listing is not available as the workflow's source of truth. The same platform also lacks cost reporting aggregated by tag. Those are operating constraints, not footnotes.

## Keep the handoff visible in code

The useful seam is account output becoming email input under the same credential and base URL. This runnable TypeScript example accepts the exact email-send payload from validated application data, rather than pretending that a provider schema is a substitute for local validation. It uses two routes, checks errors, and assigns an idempotency key to the write.

```ts
import { randomUUID } from "node:crypto";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

async function requestWithRetry(
  makeRequest: () => Promise<Response>,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await makeRequest();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`API ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("Rate limit retry budget exhausted");
}

async function getAccount(userId: string): Promise<unknown> {
  return requestWithRetry(() =>
    fetch(`${baseUrl}/auth/user/get/${encodeURIComponent(userId)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    }),
  );
}

async function sendEmail(
  payload: Record<string, unknown>,
  attemptId: string,
): Promise<unknown> {
  return requestWithRetry(() =>
    fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
        "Idempotency-Key": attemptId,
      },
      body: JSON.stringify(payload),
    }),
  );
}

async function sendVerification(
  userId: string,
  validatedEmailPayload: Record<string, unknown>,
) {
  const account = await getAccount(userId);

  const attemptId = randomUUID();
  const delivery = await sendEmail(validatedEmailPayload, attemptId);

  return { attemptId, account, delivery };
}

const payloadJson = process.env.VERIFICATION_EMAIL_PAYLOAD_JSON;
if (!payloadJson) throw new Error("VERIFICATION_EMAIL_PAYLOAD_JSON is required");

const result = await sendVerification(
  process.argv[2] ?? "",
  JSON.parse(payloadJson) as Record<string, unknown>,
);
console.log(JSON.stringify(result));
```

The caller builds `VERIFICATION_EMAIL_PAYLOAD_JSON` from the current discovery schema and its own approved template mapping. That is intentional. It avoids freezing undocumented fields into an article while keeping the example executable. The account response and delivery response are returned together so the caller can normalize them into the evidence record.

An SMS fallback follows the same policy decision, but it should be a separate attempt linked to the first one. Check suppression immediately before sending. Use a client-generated idempotency key for the write, poll the status into the ledger, and stop when the signup link has already been consumed. Email scheduling deserves special caution: scheduled email exists but has no cancellation route, whereas scheduled SMS can be canceled. For a short-lived signup link, an application queue may be easier to revoke than an already scheduled email.

## Where the experiment should fail

Run the integration against adverse states before evaluating delivery speed. A blocked recipient must produce no send. A retry with the same attempt ID must not create a duplicate. An EU destination outside the approved policy set must stop in the application layer. A consumed verification link must cancel or suppress later fallback work. Polling must tolerate a delayed terminal status without repeatedly contacting the player.

Also test the organizational failure: rotate the one shared key and verify that account, email, and SMS workers all recover through the same secret-distribution path. One key removes credential sprawl, but a careless rotation now has a wider blast radius.

The pull model changes the evidence clock. Persist the provider request ID immediately, then record every status observation with its observation time. Do not rewrite an earlier state. If auditors or support staff need to know when a provider changed the status, polling alone may not establish that exact transition time; it establishes when your system observed it.

Advanced conversational replies are outside this design. Basic inbound-list support can help with simple reply handling, but it is not a conversation engine. If two-way support, voice escalation, or WhatsApp is a launch requirement, select a channel platform around that requirement instead of forcing this alert path to grow into one.

## What to measure before copying this choice

Measure evidence completeness first: the percentage of attempts with a policy reference, template version, suppression decision, provider request ID, and terminal observation. Then track verification completion time by channel and region, duplicate attempts per account, polling delay, suppressed-send prevention, and orphaned account-to-message joins. These metrics expose integration quality without claiming provider latency or uptime that was never measured.

For a solo builder, the decision rule is direct. Use the focused three-provider stack when channel independence, specialized event streams, or independent failure domains justify the credential and correlation work. Use the combined surface when one discovery contract and one key materially reduce onboarding code, and when pull-based events plus a single outage boundary fit the product's risk budget. In both cases, keep consent, geographic controls, template versioning, and the evidence ledger in the game backend.

The provider can deliver the message. The application has to defend the decision.

## Further reading

- [NIST Digital Identity Guidelines: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio Messaging Services documentation](https://www.twilio.com/docs/messaging/services)
- [Twilio Advanced Opt-Out documentation](https://www.twilio.com/docs/messaging/tutorials/advanced-opt-out)
- [Resend Webhooks documentation](https://resend.com/docs/dashboard/webhooks/introduction)
- [Clerk email and SMS verification documentation](https://clerk.com/docs/guides/development/custom-flows/authentication/email-sms-otp)
- [European Commission: Data protection under GDPR](https://commission.europa.eu/law/law-topic/data-protection/data-protection-eu_en)
