# Host Playback Position: Express Clients Converge Gradually, Avoid Hard Seeking

Use gradual correction for ordinary watch-party drift, and reserve a hard seek for discontinuities. The host should publish a small, monotonically ordered timeline sample; each viewer should project that sample to the present, ignore stale updates, then choose one of three states: hold, nudge, or seek. This gives delivery failures somewhere safe to land. A missed sample delays correction, while a later sample still moves the client toward the same host timeline.

For a media workspace, that choice matters more than shaving a few bytes from an event. Hard-seeking on every update makes network jitter visible as player movement. Gradual correction accepts brief disagreement in exchange for calmer playback. I would choose it whenever temporary drift is tolerable and the host remains authoritative; I would choose hard seeking when the media changes, the host jumps deliberately, or drift crosses a declared safety bound.

## How should clients converge when a Node.js host publishes playback position?

A playback-position message describes a moving timeline, not an immutable business event. The useful payload is therefore a checkpoint: room, media identity, host position, playback state, playback rate, sequence number, and the host's send time. A client does not need every checkpoint if each newer one contains enough state to supersede the older one. This is convergence by replacement, not a claim of exactly-once delivery.

The sequence number is the first guard. If sample 42 arrives after sample 43, discard 42. The media identity is the second: a position from the previous video must never steer the current one. The third guard is age. A client projects a playing checkpoint forward by its observed transit time before comparing it with the local player.

Missed samples are expected.

There is an important boundary here. WebRTC data channels can be configured for ordered or unordered delivery, and partial reliability can be selected with either a retransmission limit or a lifetime limit. The WebRTC specification also exposes `bufferedAmount`, which is a signal that queued data is accumulating. Those transport choices do not remove the need for application-level sequence and media checks. They only change which failures the application will see.

## A focused Express relay

The relay keeps one current sample per room and streams replacements to connected viewers. It uses two routes: one host publication endpoint and one viewer stream. The process-local map makes the deployment boundary explicit: one room must reach one shared authority if the service runs in more than one process.

```ts
import express, { type Request, type Response } from "express";

type PlaybackSample = {
  roomId: string;
  mediaId: string;
  sequence: number;
  positionMs: number;
  playing: boolean;
  rate: number;
  sentAtMs: number;
};

const app = express();
app.use(express.json({ limit: "8kb" }));

const latest = new Map<string, PlaybackSample>();
const viewers = new Map<string, Set<Response>>();

function isSample(value: unknown): value is PlaybackSample {
  if (typeof value !== "object" || value === null) return false;
  const item = value as Record<string, unknown>;
  return typeof item.roomId === "string" &&
    typeof item.mediaId === "string" &&
    Number.isSafeInteger(item.sequence) &&
    typeof item.positionMs === "number" && Number.isFinite(item.positionMs) &&
    typeof item.playing === "boolean" &&
    typeof item.rate === "number" && Number.isFinite(item.rate) &&
    typeof item.sentAtMs === "number" && Number.isFinite(item.sentAtMs);
}

function writeSample(response: Response, sample: PlaybackSample): void {
  response.write(`id: ${sample.sequence}\n`);
  response.write(`data: ${JSON.stringify(sample)}\n\n`);
}

app.post("/rooms/:roomId/playback", (request: Request, response: Response) => {
  if (!isSample(request.body) || request.body.roomId !== request.params.roomId) {
    response.status(400).json({ error: "invalid playback sample" });
    return;
  }

  const sample = request.body;
  const current = latest.get(sample.roomId);
  if (current && sample.sequence <= current.sequence) {
    response.status(202).json({ accepted: false, sequence: current.sequence });
    return;
  }

  latest.set(sample.roomId, sample);
  for (const viewer of viewers.get(sample.roomId) ?? []) writeSample(viewer, sample);
  response.status(202).json({ accepted: true, sequence: sample.sequence });
});

app.get("/rooms/:roomId/playback", (request: Request, response: Response) => {
  const roomId = request.params.roomId;
  response.status(200);
  response.set({
    "Cache-Control": "no-cache, no-transform",
    "Content-Type": "text/event-stream",
    Connection: "keep-alive"
  });
  response.flushHeaders();

  const roomViewers = viewers.get(roomId) ?? new Set<Response>();
  roomViewers.add(response);
  viewers.set(roomId, roomViewers);
  const current = latest.get(roomId);
  if (current) writeSample(response, current);

  request.on("close", () => {
    roomViewers.delete(response);
    if (roomViewers.size === 0) viewers.delete(roomId);
  });
});

app.listen(3000);
```

Authentication is outside that focused snippet, not optional in a real deployment. The publication route must establish that the caller currently owns the host role for that room. Validate finite numbers and payload size at the edge, as shown, then add rate limits appropriate to the application's update cadence.

Trust is separate.

The client correction is where the behavior becomes visible. Keep its thresholds named and testable rather than scattering magic comparisons through player callbacks.

```ts
type Correction =
  | { kind: "hold" }
  | { kind: "nudge"; playbackRate: number }
  | { kind: "seek"; positionMs: number };

function correctionFor(
  sample: PlaybackSample,
  receivedAtMs: number,
  localPositionMs: number,
  localMediaId: string
): Correction {
  if (sample.mediaId !== localMediaId) {
    return { kind: "seek", positionMs: sample.positionMs };
  }

  const elapsedMs = sample.playing
    ? Math.max(0, receivedAtMs - sample.sentAtMs) * sample.rate
    : 0;
  const targetMs = sample.positionMs + elapsedMs;
  const driftMs = targetMs - localPositionMs;
  const magnitudeMs = Math.abs(driftMs);

  if (magnitudeMs < 250) return { kind: "hold" };
  if (magnitudeMs >= 2_000) return { kind: "seek", positionMs: targetMs };

  const adjustment = driftMs > 0 ? 0.05 : -0.05;
  return { kind: "nudge", playbackRate: sample.rate + adjustment };
}
```

The values 250 ms, 2,000 ms, and 0.05 are policy inputs in this example, not universal media constants. Tune them with playback behavior and user observation, and cap the accepted host rate to what the player can actually apply. Short correction windows also need a return to the host's declared rate; otherwise a temporary nudge becomes a permanent speed error.

This scheme has a real limitation: it is not suitable when every intermediate event must be retained, audited, or applied exactly once. Gradual correction deliberately discards superseded positions, and its nudge state temporarily changes playback speed. For a scored premiere, a legal review session, or frame-accurate editing, choose a durable event log plus an explicit acknowledgment protocol, then hard-seek at controlled boundaries. That alternative costs more storage, replay logic, and operational attention, but it preserves events that a replaceable-checkpoint design is built to forget. The trade-off is direct: calmer convergence and bounded queues on one side, complete event history and stricter coordination on the other.

## Delivery guarantees should match the event

Reliable, ordered delivery sounds safest, but an old position update has little value after a newer checkpoint exists. Under congestion, insisting on every intermediate sample can preserve information the viewer no longer needs. An unordered or partially reliable data channel may fit replaceable checkpoints, provided the application rejects stale sequences and periodically emits complete state. A reliable ordered channel remains clearer for host changes, media selection, chat, or any event whose loss cannot be repaired by the next playback sample.

That distinction prevents one delivery contract from swallowing the whole room. Presence is also different. "Who is online" is leased state, so disappearance must follow a timeout or an explicit departure rather than a missing playback checkpoint. Keep membership and timeline messages separate even if they share a connection.

Backpressure needs an equally plain rule. Watch the transport's queued-byte signal where one exists, and coalesce playback checkpoints before queues grow without bound. Never coalesce a media change into a position update. One is a state transition; the other is replaceable telemetry.

Queues lie.

## Failure tests that expose false confidence

Test the state machine with controlled disorder before testing it on a perfect local network. Deliver sequences 10, 12, then 11 and verify that 11 has no effect. Drop five consecutive checkpoints and confirm that the next complete sample converges without replay. Pause a viewer, resume it with a large drift, and expect a seek rather than minutes of nudging. Change the media identity while an old sample is in flight. Disconnect and reconnect after the relay has advanced.

Also test clock trouble. A wall clock can move, and clocks on separate machines are not automatically synchronized. Record receive time with a monotonic clock for local intervals, reject absurd transit estimates, and treat the host timestamp as an estimate used for projection rather than proof of network delay. The sequence number, not the timestamp, decides freshness.

Order wins.

Observe outcomes instead of raw message volume: correction-state counts, drift magnitude before and after correction, stale samples discarded, reconnect duration, active viewers, and queued bytes. Those measurements reveal whether the selected guarantee is producing convergence or merely producing traffic. A rising queue paired with old samples is a design signal, not a reason to publish faster.

## Ship with an explicit operating contract

Before release, write down who may publish, how host ownership expires, and which store carries the latest room sample across processes. Set separate rules for replaceable playback checkpoints and durable room transitions. Confirm that a reconnect receives a full current sample, that stale sequence numbers are harmless, and that media changes force discontinuous correction. Then rehearse overload: coalesce checkpoints, bound per-viewer queues, and remove closed streams.

The final choice is narrow and practical. Use three-state gradual correction when viewers can be briefly apart and newer complete samples supersede missed ones. Use hard seeking for identity changes, deliberate jumps, and drift beyond the safety bound. The transport can help with ordering, partial reliability, and queue visibility, but convergence remains an application rule.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
