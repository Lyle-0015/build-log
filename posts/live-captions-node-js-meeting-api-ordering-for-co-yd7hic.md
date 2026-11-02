# Live Captions: Node.js Meeting API Ordering for Commerce Cursors

Short answer: transcribe meeting audio into immutable caption segments, publish those segments through a separate realtime stream, and treat editor cursors as ephemeral state. The reconnect rule is the important part. A returning client should replay committed segments from a cursor, then receive new ones; it should not ask the speech pipeline to reconstruct history.

That shape fits an e-commerce merchandising editor where a buyer and a merchandiser discuss a catalog change while pointing at the same product fields. The caption stream helps accessibility and review. The cursor stream tells the other participant where attention is. They have different durability requirements, so coupling them creates needless recovery work.

Keep the streams separate.

## How should a Node.js meeting API publish live captions?

A caption segment has an ordering key, a meeting or document scope, text, and a publication status. Once published, its identity must remain stable. A cursor has a user identity, a current location, and a freshness window. Replaying an old cursor can put a collaborator on the wrong SKU; replaying a committed caption is usually useful.

I would store the caption record before broadcasting its notification. The broadcaster can retry the notification because the client deduplicates by segment ID. That is a small decision with a large operational payoff: a dropped connection becomes a read problem, not a speech-recognition problem.

The media path can use WebRTC 1.0 where the browser needs peer media or low-latency transport. The application event path still needs an explicit ordering and recovery contract. WebRTC data channels provide a transport, but they do not define the document's durable history or the meaning of a missing application event.

Transcription and publication need separate state.

Use three states: `draft`, `published`, and `superseded`. Draft text may change as more audio context arrives. Publication is the boundary visible to every attendee. Supersession handles a correction without mutating the meaning of an already delivered event. The exact policy depends on the speech service, but the client contract should stay boring.

A Node.js service can expose a narrow publisher interface. It does not need to know which speech engine produced the text.

```ts
type Caption = {
  id: string;
  meetingId: string;
  sequence: number;
  text: string;
  status: "published" | "superseded";
  replaces?: string;
};

interface CaptionStore {
  append(caption: Caption): Promise<void>;
  after(meetingId: string, sequence: number): Promise<Caption[]>;
}

async function publishCaption(
  store: CaptionStore,
  broadcast: (meetingId: string, caption: Caption) => Promise<void>,
  caption: Caption,
): Promise<void> {
  await store.append(caption);
  await broadcast(caption.meetingId, caption);
}
```

The sequence is scoped to the meeting, not to a WebSocket connection. A client that reconnects sends its last applied sequence. The server reads later records, sends them in order, and only then resumes live delivery. If the live subscription and the backfill overlap, the client applies the same ID once.

Collaborative cursors need a different retention policy.

Cursors should use the same authorization scope as captions but a different retention policy. Publish them as replaceable presence events, keyed by participant and document. Do not put every cursor movement into the caption log. That log is an audit-like timeline; presence is a current observation.

For a catalog editor, a cursor event might contain `documentId`, `participantId`, `fieldPath`, and a monotonically increasing client revision. On reconnect, the client fetches the current presence snapshot and resumes caption backfill separately. This avoids an awkward question: should a four-minute-old cursor be replayed before a caption about the product it once highlighted? No.

The browser can render captions as they arrive and show a cursor only while its lease is fresh. A server timestamp helps reconcile clients, but it should not be used as the ordering key when two events can share a clock value. Use the assigned sequence for the durable stream.

## What should be measured before shipping?

The useful test is a failure matrix, not a happy-path demo. Disconnect a client before it receives a broadcast, during backfill, and immediately after it applies the final backfilled record. Verify that the resulting caption IDs are complete and unique, then verify that a stale cursor disappears without changing caption history.

Measure publication delay, backfill size, duplicate suppression, abandoned draft count, and the age of the oldest presence event rendered. Keep separate counters for speech input failures and event-delivery failures. Otherwise a dashboard can report “captions delayed” while hiding that the actual problem is a blocked subscriber.

A short retention window for presence and a longer retention window for published captions is usually easier to reason about than one shared stream. The trade-off is storage and replay policy, so set both explicitly. The right values come from meeting length, reconnect behavior, and the review needs of the commerce team, not from a generic realtime default. I would accept a little more storage here because it makes recovery inspectable and keeps cursor churn out of the durable record.

## Sources

- https://www.w3.org/TR/webrtc/
- https://www.rfc-editor.org/rfc/rfc6455
- https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API
- https://www.rfc-editor.org/rfc/rfc9110
