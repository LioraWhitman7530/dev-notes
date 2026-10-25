# Node.js Live Poll Results API: 4 Trust Boundaries for Webinar Viewers

Short answer: issue a short-lived, server-signed token for one player, one video room, and an explicit set of actions; keep authoritative poll tallies behind the application service, then broadcast versioned snapshots rather than trusting client-submitted totals. For thousands of viewers, the deciding constraint is not raw socket count. It is whether a copied token, delayed result, or reconnect can cross a trust boundary without detection.

Treat room admission and poll participation as related but separate grants. A Node.js control plane can create rooms and mint claims, while a media plane carries WebRTC sessions and a result plane distributes poll snapshots. The browser receives only the authority it needs. Never place a reusable service credential in a game client.

TL;DR: define four boundaries before choosing a transport or provider: player to application, application to token issuer, client to media room, and poll command to authoritative tally. Set an SLO for accepted-vote-to-visible-snapshot latency, measure it at the viewer, and preserve a pull-based snapshot path for recovery.

## How should a live webinar poll results API establish trust?

A signature answers a narrow question: did an expected issuer produce these bytes, and were the bytes changed? It does not prove that the holder is the intended player, that the token belongs to this match, or that publishing media and submitting a poll vote should share the same authority. Those decisions come from claims, server-side policy, and how the client may use the result.

The dangerous shortcut is a room-wide bearer token with broad publish and subscribe rights. Once copied from browser storage, logs, an analytics payload, or a support screenshot, it may be replayed until expiry. A gaming room makes the blast radius concrete: a spectator should not become a speaker, a player in match `match-4821` should not enter `match-4822`, and a vote token should not authorize a media track. Short expiry reduces the window; it does not repair excessive scope.

Assume it will leak.

Model four boundaries and reject ambiguity at each one:

1. The player authenticates to the Node.js application, which resolves the stable subject and current match membership.
2. The application asks an internal issuer for a narrowly scoped grant; only this server-to-server hop has signing authority.
3. The media service verifies issuer, audience, expiry, room, subject, and permitted actions before admitting a WebRTC participant.
4. The poll service accepts an idempotent vote command, computes the tally from authoritative state, and emits a monotonically versioned snapshot.

Fast is irrelevant if unauthorized.

RFC 7519 defines registered JWT claims including `iss`, `sub`, `aud`, `exp`, `nbf`, and `jti`; it also requires audience rejection when the processing party is absent from the `aud` claim. Use those semantics rather than inventing look-alike fields. Room and action claims remain application policy, so document their types and reject unknown actions instead of silently widening access.

## Implement the narrow grant, then separate commands from results

A production issuer needs protected key material, rotation, and an algorithm policy. The focused Go example below shows the validation decision that every implementation, including a Node.js one, must preserve. It deliberately receives already verified claims: parsing a token is not signature verification, and coupling the example to one JWT package would hide that distinction.

```go
package roomauth

import (
    "errors"
    "slices"
    "time"
)

type Claims struct {
    Issuer string
    Subject string
    Audience []string
    Expires time.Time
    NotBefore time.Time
    TokenID string
    RoomID string
    Actions []string
}

type Request struct {
    PlayerID string
    RoomID string
    Action string
    Now time.Time
}

func Authorize(c Claims, r Request) error {
    if c.Issuer != "https://auth.game.example" ||
        !slices.Contains(c.Audience, "video-room") {
        return errors.New("wrong issuer or audience")
    }
    if r.Now.Before(c.NotBefore) || !r.Now.Before(c.Expires) {
        return errors.New("token outside validity window")
    }
    if c.Subject == "" || c.TokenID == "" ||
        c.Subject != r.PlayerID || c.RoomID != r.RoomID {
        return errors.New("subject or room mismatch")
    }
    if !slices.Contains(c.Actions, r.Action) {
        return errors.New("action not granted")
    }
    return nil
}
```

Signature verification must happen before `Authorize`, with an allowed algorithm selected by configuration rather than accepted from untrusted input. Keep token lifetime short enough to bound replay, but do not turn expiry into a synchronized disconnect storm: refresh through the authenticated control plane, add bounded jitter before refresh, and let the media verifier accept only the intended audience. A `jti` can support targeted revocation, although maintaining a revocation set adds state and an availability dependency to an otherwise local verification path. That is a real trade-off, not a free security toggle.

Polls need a different contract. A vote is a command carrying a poll identifier, selected option, authenticated subject, and idempotency key. The client never sends `optionA: 7421` as truth. The tally service deduplicates commands, updates authoritative state, increments a version, and publishes a snapshot such as `{poll_id, version, counts, closes_at}`. On reconnect, a client fetches the current snapshot and applies only events with a greater version. A missed notification is recoverable because the delivery channel is not pretending to be a database.

Order matters.

WebRTC defines browser APIs for real-time media and data exchange, but the signaling mechanism is outside the specification. That leaves room for an application-controlled signaling service, yet it also means WebRTC does not define this authorization policy. Put that policy in the control plane and test it directly.

## Capacity-plan the result path before opening the room

Start with offered load, not a provider checkbox. For an illustrative room with 10,000 connected viewers and one full poll snapshot every two seconds, the application creates 5,000 viewer deliveries per second. If each encoded snapshot is 600 bytes before transport overhead, the payload rate is about 3 MB/s across egress, plus protocol overhead, retries, encryption, connection maintenance, and regional replication. These are planning inputs, not benchmark results. Measure the actual encoding and transport.

A poll that changes rapidly does not require one public event per accepted vote. Coalesce updates into a bounded interval, publish the newest complete snapshot, and attach its version. This trades some freshness for predictable fan-out. Define that choice as an SLO: a team might target 99% of accepted votes becoming visible to connected viewers within two seconds, while separately measuring correctness and reconnect recovery. The target is a product decision; the percentile must come from telemetry.

Backpressure belongs in the design. Track per-connection queued bytes or messages, disconnect or resynchronize consumers that exceed a documented limit, and avoid an unbounded channel between tally workers and socket writers. A slow spectator must not retain every obsolete intermediate tally. For snapshots, replacing an older queued version with the newest one is usually safer than growing memory until the process is killed.

Bound every queue.

Node.js suits an I/O-heavy control and fan-out layer when handlers remain nonblocking, but the event loop is still a capacity boundary. Watch event-loop delay, heap use, connection count, outbound queue depth, snapshot age, authorization rejection reason, and reconnect rate. CPU-heavy aggregation should not share the latency budget of token issuance and socket heartbeats without isolation. Scale tests must include token refresh and reconnect bursts, not only steady clients.

## Choose ownership with an on-call table

The choice is broader than WebSocket versus WebRTC. WebRTC is the media path; poll results can use a separate server-to-client channel because the server owns the tally and recovery snapshot. WebSocket provides two-way messaging, while server-sent events provide a server-to-client event stream over HTTP. Either can carry versioned results. The operational question is which layer the team is prepared to own.

| Layer | Build and operate | Managed service | Decision pressure |
|---|---|---|---|
| Token issuer | Full claim policy and key lifecycle control | Faster integration, constrained claim model | Can room, subject, audience, action, expiry, and key rotation be enforced? |
| Media plane | Maximum topology control, substantial NAT traversal and on-call work | Less media infrastructure to operate | Are admission hooks and participant permissions precise enough? |
| Result fan-out | Direct control of batching, replay, and overload behavior | Outsourced connection fleet | Can clients recover by version after gaps and reconnects? |
| Tally store | Custom consistency and deduplication semantics | Service-specific limits and coupling | Where is the authoritative vote, and how is an idempotency key retained? |

A managed option can reduce connection and media operations, while self-hosting can reduce policy constraints and dependency on a proprietary control surface. Neither label resolves the decision. Run the same abuse, expiry, overload, and recovery tests against each candidate, then price the complete workload only after the trust model passes. Include engineering ownership, key rotation, regional failure handling, observability, and exit work in that comparison; a message-rate quote is not a cost model.

Avoid selecting on the prettiest client SDK. The browser is the least trusted part of this system, so the decisive APIs are server-side token minting, deterministic authorization, idempotent vote ingestion, snapshot retrieval, and telemetry export. A thin client adapter is easier to replace than provider-specific identity or tally semantics embedded throughout the application.

## Verify denial paths and rehearse rollback

Before deployment, build a matrix with subject, room, audience, action, validity window, and expected result. Include a valid spectator subscription, a spectator attempting to publish, a player crossing rooms, a token presented to the wrong audience, an expired token, a token that is not active yet, a duplicate vote command, a skipped snapshot version, and a reconnect after several versions. Each denial needs a stable reason code for metrics, but client errors should not reveal signing or membership details.

Then load-test the shape of the event, not a toy echo. Generate the planned mix of viewers, votes, refreshes, reconnects, slow consumers, and snapshot fetches. Record accepted-vote time at the authority and applied-snapshot time at the client probe. Averages conceal the viewers most likely to distrust a live result; use latency distributions and count clients whose displayed version trails the authoritative version.

Rollout should be reversible. Start with internal rooms, then a small cohort, and compare old and new result paths using the same authoritative snapshots. Keep the pull endpoint available throughout the migration. If snapshot age, authorization failures, queue depth, or reconnect recovery breaches its threshold, stop new admissions to the new path, direct clients to fetch current state, and drain existing connections. Do not roll back the tally store by replaying client displays; those displays are projections, not authority.

Key compromise is a separate runbook. Rotate the active signing key, remove trust in the compromised key after a bounded overlap chosen by policy, revoke affected token identifiers when the architecture supports it, and force authenticated reminting. Practice this with synthetic keys. A rotation plan that has never been exercised is documentation, not a control.

The acceptance rule is blunt: choose the design that can prove a client has the minimum authority for one room, can recover the latest tally after losing any transient event, and can meet the declared latency SLO under refresh and reconnect load. Everything else is implementation detail.

## References

- https://www.w3.org/TR/webrtc/
- https://www.rfc-editor.org/rfc/rfc7519
- https://www.rfc-editor.org/rfc/rfc6455
- https://html.spec.whatwg.org/multipage/server-sent-events.html
- https://www.rfc-editor.org/rfc/rfc8725
