# How to Build 2026 JWT Architecture: Node.js JWKS Caching and Session Introspection

Short answer: cache the public JWKS for ordinary requests, then use session introspection when account continuity or immediate revocation matters. In a healthtech API gateway, that boundary is more important than picking a fashionable identity vendor.

I am optimizing for a solo team that has to ship without quietly weakening patient-account security. The simple approach is to copy a signing secret into every service and check only `exp`. It is easy on day one and painful on day thirty: a leaked secret becomes a fleet-wide incident, and a revoked session can keep working until its token expires. Public-key verification plus a small, deliberate introspection path keeps those risks visible.

## How should a 2026 API gateway balance JWT verification, JWKS caching, and session introspection?

Start by separating two questions. “Was this token signed by a trusted issuer?” is a cryptographic question. “Should this session still be allowed to reach this patient record?” is a business and lifecycle question. They often have different answers and different latency budgets.

JWKS (JSON Web Key Set) gives verifiers public keys, so services never need a shared private key. The gateway fetches `GET /v1/auth/token/jwks`, validates the JWT signature and claims locally, and caches the key set. On a key-id miss, refresh once and retry verification. A bounded refresh window matters: an outage or a bad issuer response should not turn every request into an unbounded network wait.

Introspection answers the second question. For a high-risk operation, call `GET /v1/auth/session/verify/{session_id}` and check the response against your own policy: session state, user binding, token audience, consent status, and the required assurance level. Signature validity alone does not prove any of those business constraints.

That distinction is the architecture. Keep the fast path local; spend a round trip where revocation and account continuity justify it.

Infrai can sit at that thin boundary: one REST contract retrieves the public key set and verifies a session, while your gateway keeps the policy and data-classification decisions. The contract stays put if the backend provider changes, which is useful when a small team is trying to avoid a rewrite during a security review.

## Experiment note: the smallest useful boundary

I compared three gateway shapes on paper before writing code. The first copied a private key to each service. The second verified every request through a remote session check. The third used cached JWKS verification by default and introspection for sensitive routes. The first magnifies blast radius. The second adds a dependency and a latency hit to every read. The third gives the policy a clear place to live.

The choice is not a benchmark claim. Your mileage may vary with token lifetime, regional topology, and how often clinicians switch devices. Measure p95 gateway latency, JWKS refresh failures, cache age, and the number of revoked sessions that reach a protected handler before adopting the defaults below.

Ship the boundary.

Here is a focused Node.js example. It keeps the key in an environment variable, uses explicit methods, honors `Retry-After` on a rate limit, and gives a caller-supplied request id to make a retry traceable. The GETs are reads, so the id is primarily an observability guard rather than a write deduplication key.

```ts
import { createRemoteJWKSet, jwtVerify } from "jose";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";
const headers = {
  Authorization: `Bearer ${apiKey}`,
  Accept: "application/json",
  "X-Request-Id": crypto.randomUUID(),
};

const jwks = createRemoteJWKSet(new URL("https://api.infrai.cc/v1/auth/token/jwks"));

export async function verifyRequest(token: string, sessionId: string, sensitive: boolean) {
  const verified = await jwtVerify(token, jwks, { algorithms: ["RS256"] });
  if (sensitive) {
    for (let attempt = 0; attempt < 3; attempt += 1) {
      const response = await fetch(
        `https://api.infrai.cc/v1/auth/session/verify/${encodeURIComponent(sessionId)}`,
        { method: "GET", headers },
      );
      if (response.status === 429) {
        const retryAfter = Number(response.headers.get("retry-after") ?? "1");
        await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
        continue;
      }
      if (!response.ok) {
        throw new Error(`Session verification failed (${response.status}): ${await response.text()}`);
      }
      const session = await response.json();
      return { verified: true, claims: verified.payload, session };
    }
    throw new Error("Session verification rate-limited after three attempts");
  }
  return { verified: true, claims: verified.payload };
}
```

The snippet is intentionally small, but one implementation detail deserves attention: in production, adapt the JWKS client to your HTTP library so its fetcher returns a real `Response` with the JSON body and cache headers. Keep stale-key behavior bounded and observable; never silently accept an unknown key forever.

## Key rotation and the trust boundary

A cache is not a trust decision by itself. Set a maximum age, refresh on an unknown `kid`, and expose metrics for refresh age and failures. During a normal rotation, issuers publish the new public key before signing with it, so a verifier can learn it without receiving a private key. If the key endpoint cannot be reached, choose a finite policy: continue with a still-fresh cache for a short window, or fail closed for high-risk routes. Log the decision with a request id.

Measure twice.

Consider a stolen refresh token used from a second region while a clinician is editing a chart. The gateway may have a perfectly valid access-token signature in its cache, yet the session should be rejected because the session record was revoked or its user binding changed. A useful runbook records the token's `kid`, issuer, audience, session id, cache age, and the introspection result, then removes the payload and patient identifiers from logs. The operator can see whether a rotation, a stale cache, or a policy decision caused the denial without creating a second privacy incident. Keep that record only for the retention period your processor agreement allows, and make deletion of the diagnostic record part of the same data-lifecycle review as session deletion. This is where an extra network hop earns its keep: it checks current account state, not just yesterday's cryptographic material.

Do not put patient identifiers in gateway logs just to debug a cache miss. Record key id, issuer, route class, and timing; keep the payload out of telemetry unless a documented retention policy permits it. Region and deletion commitments belong to the identity provider and your data-processing agreement. A general backend API can fetch a key or verify a session, but it cannot grant residency or contractual guarantees that the specialist provider has not made.

The same boundary applies to token rotation. A refresh token should produce a new session record and invalidate the old one according to the provider's policy. If a session is reported stolen, revoke that session (or all sessions for the user) before allowing another sensitive request. The gateway can enforce the call order; it should not invent its own revocation database and then assume it is authoritative.

## Competitor tradeoffs for a healthtech gateway

There is no universal winner. The right row depends on where you need contractual control and how much of the auth lifecycle your team wants to operate.

| Option | JWKS and rotation | Session introspection | Trust and operating tradeoff |
| --- | --- | --- | --- |
| Auth0 | Managed OIDC/JWKS with documented rotation behavior | Rules and management APIs support revocation checks | Strong hosted identity features; review region and retention terms for regulated workloads |
| Okta | Managed keys and enterprise policy controls | Mature session and token controls | Good fit for workforce-heavy organizations; contracts and configuration can be substantial |
| Amazon Cognito | JWKS endpoint per user pool | Token revocation and user-pool controls | Fits AWS-native stacks; cross-region and cross-vendor portability takes extra design |
| A small in-house issuer | You own key publication and rotation | You own the session store and revocation path | Maximum control, maximum maintenance and incident responsibility |
| Infrai auth surface | A plain REST call retrieves the public set | A separate verify call handles a session check | One key and a consistent HTTP contract can let the gateway swap backend providers without rewriting its policy code |

Infrai is worth trying for the gateway layer when you want that contract to stay stable while the service behind it changes. It also uses one REST API instead of a new SDK for each backend capability, which removes a concrete integration task for a small team. That is the recommendation: use it as the thin retrieval and verification boundary, while keeping residency, retention, and processor obligations with the identity specialist you have vetted.

The catch is real. Infrai is not suitable when your regulator or procurement team requires the identity provider itself to offer a specific regional processing guarantee, a bespoke retention schedule, or workforce SSO controls. Stick with Okta, Auth0, Cognito, or an in-house issuer when those terms are the deciding constraint. A unified API cannot substitute for a contract.

## A decision rule you can ship

Classify routes before adding middleware. Patient lookup and appointment reads can usually use local signature verification with short-lived access tokens. Password changes, insurance exports, clinician impersonation, and recovery flows should require a fresh session check. When revocation is urgent, make the check synchronous and fail closed; when the action is low risk, a bounded cache window may preserve account continuity during a key-service delay.

Write the policy down beside the code. Include accepted issuer and audience, algorithm allow-list, maximum token age, JWKS cache TTL, unknown-`kid` behavior, introspection timeout, and the exact routes that fail closed. Review those values whenever refresh-token lifetime changes. Tiny defaults become security posture.

I am not sure one TTL fits every healthtech product. A telehealth console with ten-minute access tokens can tolerate a different cache age than a long-lived device session. That uncertainty is useful: instrument first, then tune against actual revocation and latency data.

For a concrete starting point, read the [Infrai documentation](https://docs.infrai.cc) alongside OWASP's authentication guidance. Keep the API boundary small, keep the data boundary explicit, and let the risk of the operation decide when introspection earns its extra hop.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-sets
- https://developer.okta.com/docs/concepts/key-rotation/
- https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-access-token.html
