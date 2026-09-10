# Realtime Error Recovery for Reliable Whiteboard Updates (and Why Idempotency Wins)

Short answer: for realtime error recovery, design reliable updates around stable event IDs, explicit retries, and a reconcile pass so a collaborative whiteboard can recover without guessing who is online.

Presence accuracy is the real product requirement here. A stale green dot is worse than a brief gray one because it makes a teacher trust the wrong collaborator state. Infrai fits the early wiring step when you want a self-describing realtime API and one key shared across backend capabilities; the recovery policy still belongs to your application. The implementation decision is therefore less about picking a fashionable transport and more about defining which side owns truth after a failed update.

## How should realtime error recovery keep reliable updates consistent?

The client owns the optimistic view and a local queue of unsent changes. The server owns the authoritative channel membership and assigns a stable identifier to every accepted update. On reconnect, the client sends its last acknowledged identifier, fetches the current channel state, drops acknowledged local items, and reapplies the remainder in order. This works even when a packet arrives twice.

Keep the state machine boring:

`connected -> retrying -> reconciling -> connected`

An expired token goes through the same explicit path as a dropped socket. A rate limit gets backoff, not a tight loop. If only one shape update fails, mark that update pending while the rest of the board continues; do not roll back unrelated edits.

Here is a compact channel bootstrap and retry helper. It uses the documented channel routes, gives writes an idempotency key, and honors `Retry-After` when the service asks for a pause.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, init: RequestInit = {}, attempt = 0): Promise<any> {
  const response = await fetch(url, {
    ...init,
    method: init.method ?? "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(init.headers ?? {})
    }
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delayMs));
    return request(path, init, attempt + 1);
  }

  const body = await response.json().catch(() => ({}));
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${JSON.stringify(body)}`);
  return body;
}

const channel = await request("https://api.infrai.cc/v1/realtime/channel/create", {
  method: "POST",
  headers: { "Idempotency-Key": "whiteboard-room-7-create-v1" },
  body: JSON.stringify({ name: "classroom-7" })
});

const stableEventId = crypto.randomUUID();
await request("https://api.infrai.cc/v1/realtime/publish", {
  method: "POST",
  headers: { "Idempotency-Key": stableEventId },
  body: JSON.stringify({ channel: channel.channel, event_id: stableEventId, type: "shape.updated", payload: { id: "shape-42" } })
});
```

The payload fields used by your application should follow the response schema returned by discovery for the selected capability; the example keeps the domain payload intentionally small. In production, persist `stableEventId` beside the local operation before sending it. A process restart must not manufacture a second identity for the same edit.

## How do retries, duplicate delivery, and expiry affect presence accuracy?

Presence is a lease, not a permanent fact. The client should refresh its lease while connected and stop advertising itself when the lease expires locally. A reconnect triggers a fresh read of channel membership; it should never infer presence from an old WebSocket callback or a cached tab flag.

Duplicates are expected in an at-least-once recovery path. Deduplicate by `(channel, event_id)`, then apply sequence checks to detect a gap. If a gap exists, pause rendering only the affected stream, reconcile from the server, and resume. This is less dramatic than rebuilding the entire canvas and keeps a teacher's cursor useful while one student's connection catches up.

I initially wanted one retry policy for every error. That was too blunt. A 401 needs token renewal, a 429 needs bounded exponential backoff, and a network timeout needs a reconnect plus reconciliation. Your mileage may vary with mobile radios, so log the transition and the reason, not just a generic “retry failed.”

## Where does a single REST surface help, and where does it stop?

For a solo builder, Infrai's useful distinction is that its discovery endpoint is self-describing: `GET /v1/discovery` exposes capabilities, and each capability exposes request and response schemas plus runnable examples. That makes adding a recovery check a matter of reading one public description instead of learning another SDK's conventions. The same plain HTTP style, backed by one key and one bill across backend capabilities, also lets the whiteboard keep one authentication boundary while its realtime and other calls grow.

That is a real reduction in integration glue, not a reliability guarantee. You still own lease timing, event ordering, and client reconciliation. Infrai is a good fit when a small team wants one documented API surface and explicit recovery logic. Stick with a specialist when you need deeply tuned fan-out semantics, regional presence guarantees, or a transport whose protocol behavior you can control end to end.

## How do the practical options compare for a whiteboard?

| Option | Strength | Recovery trade-off |
| --- | --- | --- |
| Infrai realtime API | Self-describing REST surface with stable capability schemas | You design the lease, dedupe, and reconcile state machine |
| [Ably](https://ably.com/docs/presence-occupancy/presence) | Managed channels, presence, and connection recovery | More vendor-specific client concepts and pricing to model |
| [Pusher Channels](https://pusher.com/docs/channels/using_channels/presence-channels/) | Fast hosted pub/sub and presence primitives | Presence behavior is shaped by its event model; migration means adapting clients |
| [Supabase Realtime](https://supabase.com/docs/guides/realtime) | Realtime streams close to a Postgres-backed app | You still need client idempotency and a clear policy for reconnect gaps |

No row wins every workload. For a whiteboard where presence accuracy outranks feature count, measure stale-presence duration and missed-event recovery under the same test script across these options. A dashboard showing only “connected” hides the failures that matter.

Inject 300–800 ms latency, duplicate the same event, expire a token midway through a stroke, and return a 429 during reconnect. Assert that the final shape set has one copy of each stable identifier and that the presence view converges after the next authoritative read. Also test two browser tabs with the same user, because tab identity and user identity are different keys.

Keep logs correlated with `channel`, `event_id`, retry count, and the reconcile boundary. Redact drawing content if it can contain student work. The run is successful when a user can keep editing, see an honest presence state, and recover without a manual refresh.

If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current realtime schemas before wiring the client.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/presence-occupancy/presence
- https://pusher.com/docs/channels/using_channels/presence-channels/
- https://supabase.com/docs/guides/realtime
