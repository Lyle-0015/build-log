# US/EU Beginner 2FA Login Stack: SMS OTP Evidence and Suppression Controls

Short answer: for a beginner SaaS that sends password-reset challenges in the US and EU, choose SMS OTP with four local evidence gates: suppression, challenge expiry, verification, and bounded status polling. The important design choice is who owns the evidence. The provider can report message state; your application must decide whether a reset is still valid, whether another send is allowed, and how the support team explains the result.

That makes this a governance problem before it becomes a vendor problem. A successful API request is not the same thing as a delivered code, and a delivered code is not the same thing as a successful reset. Keep those events separate from the first table you create.

This is the part that is easy to skip.

Consider a concrete reset timeline. At 09:00 the user requests a code for a US number, the suppression check passes, and the application records an attempt that expires at 09:05. The provider accepts the message at 09:00:02, but the first status observation is still pending at 09:00:03. At 09:01 the user presses resend, so the application creates attempt B and marks attempt A superseded. If the code from A arrives at 09:02, verification must reject it even though its message status later becomes delivered. If B is entered at 09:03, the account can proceed. If a support agent reviews the case at 09:06, the record should show two different outcomes: A was a valid send that became ineligible, while B was the current challenge. That sequence is why provider message IDs, application attempt IDs, expiry times, and verification results deserve separate fields. A single delivery flag cannot explain it, and a single phone-number record cannot reconstruct it cleanly.

## Should a beginner 2FA login stack use SMS OTP?

Start with one record per reset attempt. Give it an internal attempt ID, creation time, expiry time, destination country, suppression decision, provider message ID, verification result, and the latest provider status. Store message metadata beside the attempt so you can attribute usage to the password-reset feature later; there is no tag-aggregated cost reporting API for this selection.

The record is deliberately boring. That is useful. A support agent can answer four different questions without guessing:

- Was the number blocked before the application tried to send?
- Did the provider accept a message for this attempt?
- What delivery state was last observed?
- Did the user verify before the application expiry time?

Do not collapse these into `sendSucceeded`. A carrier can accept a message while the challenge expires, and a user can enter a valid code after a newer reset attempt has superseded it. The application should reject that old attempt even if its eventual delivery status looks healthy.

I would also make the attempt ID the unit of support review, rather than the phone number. A phone number can have several outstanding requests; a support investigation needs the timeline of one request. This is a small governance decision with a large practical payoff.

## Which four gates keep the flow explainable?

The first gate is suppression. Check the destination before sending and record the result, including a skip. A blocked number should not look like a carrier outage, and a skipped send should not quietly turn into a resend loop.

The second gate is expiry. Set the reset deadline in your own database and compare it at verification time. A short code lifetime is a security policy, not a delivery-status field. Status polling must never extend it.

The third gate is verification. Treat the provider's OTP operation as the direct path to a working challenge flow, but keep the application's session transition separate. The login service should mark the reset attempt verified only after it has checked that the attempt is current, unexpired, and bound to the intended account.

The fourth gate is status observation. Poll lightly, stop after a fixed budget, and record the last observed state. Polling is a way to improve the support view; it is not a delivery guarantee. Webhook events are not available here, so a design that requires push-driven orchestration has a different constraint.

For a small support team, this four-gate model is more useful than a long feature checklist. It says which facts must survive a dispute and which decisions belong in application code.

## How should a bounded status poll behave?

The status adapter should be narrow enough that it can be replaced. The example below reads the key from the environment, uses the documented v1 base URL, sets an explicit method, honors `Retry-After` on rate limiting, and surfaces the response body for other failures. Three attempts is an application policy, not a provider promise; the caller should also enforce an overall time budget.

```ts
export async function readSmsStatus(messageId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) {
    throw new Error("INFRAI_API_KEY is required");
  }
  if (!baseUrl) {
    throw new Error("INFRAI_BASE_URL is required");
  }

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/sms/status/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 2 ** attempt * 250;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`SMS status failed (${response.status}): ${detail}`);
    }

    return response.json();
  }

  throw new Error("SMS status rate limit persisted after three attempts");
}
```

The send and verify boundaries need the same discipline. Use a client-owned attempt identity and an idempotency key for create or publish operations so a browser double-click or a retried request does not create competing reset attempts. The platform convention specifies an `Idempotency-Key` header and a default 24-hour deduplication window, but the application still decides which retry is safe.

Keep the poll schedule explicit in configuration. For example, a UI can observe at 1, 2, and 4 seconds, then stop. Those numbers are a product choice, not a claim about carrier performance. If the user requests another code, create a new attempt and make the old one ineligible for verification.

## How do the real options differ on governance?

The right comparison is the work left in your application after the first message is sent. Twilio Verify is a focused managed verification service with a substantial documentation surface. Vonage Verify is a natural candidate for a team already using Vonage communications. Bird puts verification inside a broader communications workspace. A REST aggregator can reduce the number of backend credentials, but it leaves more policy and measurement work to the application.

| Option | Governance fit | Boundary to validate |
| --- | --- | --- |
| Twilio Verify | Focused managed verification workflow | Confirm country rules, retry semantics, and the evidence exposed to support operators |
| Vonage Verify | Verification for teams already close to Vonage communications | Confirm regional coverage, status meanings, and the reset-attempt model |
| Bird | Verification within a wider communications workspace | Confirm that its workflow exposes the suppression and delivery controls the product needs |
| Infrai | Plain REST operations that can sit behind a replaceable adapter | Own expiry, abuse controls, country policy, and local cost attribution |

Infrai is a reasonable fit when one key and one bill across backend services matter to a small team, and when plain SMS is enough. A second advantage is its interface shape: one REST API means a TypeScript service, a worker in another runtime, or a small operational script can use HTTP without installing an SDK. Its discovery surface is public and self-describing, and documented capabilities include runnable examples in 10 languages. That reduces the friction of inspecting a contract or moving this adapter into a different runtime later.

The tradeoff is ownership. A focused verification product may package more verification-specific policy, while this choice leaves expiry, abuse controls, and feature-level reporting with the application.

The platform also covers 295 routes across 20 modules behind the same key. That breadth can help when the support product adds an adjacent backend operation, while the adapter remains a small boundary around SMS. It is not a substitute for local policy. The platform does not provide built-in fraud controls or tag-level cost reporting, and it does not turn polling into event delivery.

## When is this selection the wrong one?

Use a different design if the product needs voice, WhatsApp, or RCS. Those channels are not available here. Choose a provider with push events if the workflow depends on webhook-driven orchestration; both relevant namespaces use pull-based events, so polling has a ceiling.

This selection is not suitable for a team that needs built-in fraud controls or tag-aggregated cost reporting. A focused verification provider may be the better fit when those controls are requirements rather than later improvements.

The same warning applies to fraud and spend controls. Geographic fencing and per-country pricing circuit breakers belong in the business layer. Record country, feature, attempt, and outcome in your database, then enforce the policy before the send. Do not infer a fraud decision from delivery status.

Email is not a ready-made fallback in this selection: there is no hosted email OTP interface, and scheduled email has no cancellation route. Domestic email vendor readiness is also not a basis for domestic compliance claims. If the product requires those properties, model the fallback as a separate capability with its own review.

## Decision rule

For a beginner US/EU SaaS password-reset flow, use SMS OTP with suppression checks and lightweight status polling when plain SMS is acceptable and the team is willing to own expiry, abuse controls, and reporting. Keep the provider behind an adapter, retain one record per attempt, and compare terminal evidence rather than marketing feature counts.

Before shipping, test the four gates in each target country: a suppressed number, an accepted message, a late status, and an expired challenge. Measure the application's own outcomes, including verification after expiry and resend frequency. The useful question is not whether a provider can send one code. It is whether your support team can explain what happened without reopening a reset that should remain closed.

## References

- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify API documentation: https://developer.vonage.com/en/verify/overview
- Bird Verify documentation: https://docs.bird.com/api/verify
- MDN Fetch API: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
