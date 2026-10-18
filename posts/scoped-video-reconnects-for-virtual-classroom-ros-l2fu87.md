# Scoped Video Reconnects for Virtual Classroom Rosters Without a Socket Server

TL;DR: Treat video, roster presence, and hand raises as three related but different data flows. Let a managed realtime service carry transient signals, but keep a small authoritative event log behind your Node.js application so reconnecting clients can backfill missed changes. Mint short-lived, room-scoped video credentials only after checking the user's property and training-session membership. This avoids operating a socket server without pretending a live connection is durable storage.

For a property-management company, the concrete room might be a live safety briefing for maintenance contractors across 40 buildings. The costly failure is not a slightly delayed emoji. It is a supervisor seeing 17 attendees while the compliance record says 18, or a technician's raised hand vanishing during a network switch. The design below makes those disagreements explicit and recoverable.

## How Should a Node.js Virtual Classroom Realtime API Recover State?

Start by classifying state by consequence. WebRTC provides browser APIs and a connection model for real-time audio, video, and data; it does not turn an application roster into a durable business record. A media participant, a realtime subscriber, and an authorized training attendee are separate concepts even when the UI renders them in one panel.

I would use three lifetimes:

| State | Owner | Reconnect behavior |
|---|---|---|
| Camera and microphone tracks | Video room | Rejoin with a fresh scoped credential when required |
| Online indicator and typing-like signals | Realtime presence | Recompute after reconnect; tolerate brief uncertainty |
| Hand-raise transitions and attendance checkpoints | Application event log | Backfill from the last applied sequence, then resume live delivery |

That division is the central trade-off. Persisting every presence pulse creates write volume with little business value. Keeping hand raises only in presence makes them disappear precisely when a weak connection matters. For a solo team, a boring append-only table is cheaper to reason about than clever client reconciliation.

Keep the roster definition narrow as well. “Authorized for session” comes from application data. “Connected now” comes from presence. “Joined media” comes from the video layer. Displaying these as separate states prevents a temporary disconnect from silently rewriting attendance.

## The Data Flow Before the Code

The browser first asks the application backend to join a training session. The backend verifies the signed-in user, property boundary, role, and session window. It then mints a short-lived video credential limited to one room and returns the current durable sequence. The browser connects to media and subscribes to a managed realtime channel; it never receives a credential that can publish to arbitrary rooms.

When a technician raises a hand, the browser sends a command to the application backend. The backend authorizes it, appends an event with a monotonically increasing sequence for that session, and publishes the committed event. On reconnect, the client requests all events after its last applied sequence before accepting newer live events. The database is authoritative; realtime delivery is an acceleration path.

Order matters here. Subscribe first, record the subscription boundary, fetch the missing range, and then drain buffered live events in sequence. Fetching first and subscribing second leaves a gap where an event can be committed but never observed by that client.

The gap is real.

Suppose the client remembers sequence 812. It fetches after 812 and receives an empty response; a hand raise is then committed as 813, just before the client establishes its live subscription. Neither request failed, yet the screen remains stale. Reversing the order closes that window: a live event arriving during backfill waits in a local buffer, the range query establishes what was committed through its response, and the client applies the combined stream by sequence. This costs a little client complexity. I would pay it rather than make every instructor refresh the roster after a network change.

## A Small Node.js Boundary That Can Move Between Providers

The useful abstraction is not a universal realtime API. It is a narrow boundary around the semantics this room needs. The following TypeScript is intentionally generic: adapters can sit behind the interfaces, while authorization and recovery remain application code.

```ts
type Role = "instructor" | "technician" | "observer";

type JoinRequest = {
  userId: string;
  propertyId: string;
  sessionId: string;
};

type VideoGrant = {
  roomId: string;
  subject: string;
  role: Role;
  expiresAtEpochSeconds: number;
};

type ClassroomEvent = {
  sessionId: string;
  sequence: number;
  kind: "hand.raised" | "hand.lowered" | "attendance.checked";
  actorId: string;
  occurredAt: string;
};

interface MembershipStore {
  findActiveMembership(input: JoinRequest): Promise<{ role: Role } | null>;
}

interface EventStore {
  latestSequence(sessionId: string): Promise<number>;
  after(sessionId: string, sequence: number): Promise<ClassroomEvent[]>;
  appendHandState(input: {
    sessionId: string;
    actorId: string;
    raised: boolean;
    idempotencyKey: string;
  }): Promise<ClassroomEvent>;
}

interface VideoCredentialIssuer {
  issue(grant: VideoGrant): Promise<string>;
}

interface RealtimePublisher {
  publish(channel: string, event: ClassroomEvent): Promise<void>;
}

const roomChannel = (propertyId: string, sessionId: string) =>
  `property:${propertyId}:training:${sessionId}`;
```

The join handler should expose only what the client needs. Ten minutes is an example credential lifetime for this design, not a universal recommendation; set it from the session risk and the issuer's refresh behavior. The important control is scope.

```ts
async function joinTrainingRoom(
  request: JoinRequest,
  dependencies: {
    memberships: MembershipStore;
    events: EventStore;
    video: VideoCredentialIssuer;
  },
) {
  const membership = await dependencies.memberships.findActiveMembership(request);
  if (!membership) throw new Error("Training session access denied");

  const expiresAtEpochSeconds = Math.floor(Date.now() / 1000) + 10 * 60;
  const videoToken = await dependencies.video.issue({
    roomId: request.sessionId,
    subject: request.userId,
    role: membership.role,
    expiresAtEpochSeconds,
  });

  return {
    videoToken,
    channel: roomChannel(request.propertyId, request.sessionId),
    latestSequence: await dependencies.events.latestSequence(request.sessionId),
  };
}

async function raiseHand(
  input: JoinRequest & { raised: boolean; idempotencyKey: string },
  dependencies: {
    memberships: MembershipStore;
    events: EventStore;
    realtime: RealtimePublisher;
  },
) {
  const membership = await dependencies.memberships.findActiveMembership(input);
  if (!membership || membership.role === "observer") {
    throw new Error("Hand-raise access denied");
  }

  const event = await dependencies.events.appendHandState(input);
  await dependencies.realtime.publish(
    roomChannel(input.propertyId, input.sessionId),
    event,
  );
  return event;
}
```

There is a deliberate failure mode in that sequence: the database commit can succeed while publish fails. Do not roll back the hand raise just because fan-out had a bad moment. Return the committed event to the caller, retry publication from an outbox or worker, and let other clients recover it through backfill. This is one extra table and worker path, but it turns delivery failure into delay rather than data loss.

The `idempotencyKey` is equally practical. Mobile clients retry. The event store should enforce uniqueness for the session and actor so one tap cannot become two transitions after a timeout.

## Reconnect and Backfill Are One Protocol

A client needs a single rule: apply an event only when its sequence is the next expected value. Duplicate sequences are ignored. A higher-than-expected sequence pauses application and triggers backfill. This is more dependable than sorting by client timestamps, which can disagree across devices.

```ts
type FetchBackfill = (
  sessionId: string,
  afterSequence: number,
) => Promise<ClassroomEvent[]>;

class OrderedClassroomState {
  private lastApplied = 0;
  private buffered = new Map<number, ClassroomEvent>();

  constructor(private readonly fetchBackfill: FetchBackfill) {}

  async receive(event: ClassroomEvent): Promise<ClassroomEvent[]> {
    if (event.sequence <= this.lastApplied) return [];
    this.buffered.set(event.sequence, event);

    if (event.sequence > this.lastApplied + 1) {
      const missing = await this.fetchBackfill(event.sessionId, this.lastApplied);
      for (const item of missing) this.buffered.set(item.sequence, item);
    }

    const applied: ClassroomEvent[] = [];
    while (this.buffered.has(this.lastApplied + 1)) {
      const next = this.buffered.get(this.lastApplied + 1)!;
      this.buffered.delete(next.sequence);
      this.lastApplied = next.sequence;
      applied.push(next);
    }
    return applied;
  }
}
```

This sample keeps state in memory, so a page reload begins from a server-provided checkpoint or locally stored last sequence. Production code also needs cancellation, bounded buffers, retry backoff, and a snapshot path when the event range is too large. Those are operational controls, not reasons to move authorization into the browser.

Do not use roster count as the recovery cursor. Presence changes can collapse, expire, or arrive around the same time. A committed sequence gives the client an exact question to ask: “What happened after 814?”

## Test the Gaps, Not the Happy Path

The basic test uses three clients: instructor, technician, and observer. Disconnect the technician after sequence 812, commit a hand raise at 813 and an attendance checkpoint at 814, then reconnect while 815 is being published. The technician must apply 813, 814, and 815 exactly once and in order. The observer must be rejected when attempting a hand-raise command, even if it can view the room.

Then test credential boundaries. A token minted for property A must not join property B's room. A removed member must fail the next join or refresh. Expiry must not erase already committed classroom events. These checks belong in adapter contract tests because replacing a media or realtime component must not alter application policy.

Measure the system at its joins: token issuance failures by reason, event-commit-to-publish delay, reconnect backfill size, sequence-gap frequency, and the age of the oldest unpublished outbox row. Avoid logging tokens or raw authorization headers. Useful telemetry identifies a stuck delivery path without creating a credential leak.

Cost follows fan-out and retention more than the number of interface methods. Before choosing a managed service, estimate concurrent room members, presence churn, events per hand action, reconnect frequency, and retained event volume. Compare billing units against that workload and verify limits with the current documentation. A provider-neutral adapter helps migration, but it does not make different delivery guarantees identical.

## Ship With Explicit Recovery Rules

Before release, read the join path from left to right: identity is authenticated; property and session membership are checked server-side; the video credential names one room, one subject, one role, and an expiry; realtime publish rights are no broader than the UI action; durable mutations are idempotent; and every committed event receives a session sequence. Then read the reconnect path backward: live events can buffer, missing ranges can be fetched, duplicates are harmless, gaps are visible, and an outbox can repair commit-versus-publish failures.

That is enough architecture for a first production classroom. The browser handles WebRTC media, a managed channel removes the need to operate a socket fleet, and the application keeps the small amount of state whose loss would affect instruction or compliance. **The durable log is the truth; presence and live delivery are views of what is happening now.**

## Sources

References:

- W3C, “WebRTC 1.0: Real-Time Communication Between Browsers”: https://www.w3.org/TR/webrtc/
