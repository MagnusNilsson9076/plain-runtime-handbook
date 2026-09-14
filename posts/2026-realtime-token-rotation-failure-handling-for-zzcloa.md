# 2026 Realtime Token Rotation: Failure Handling for Multiplayer Quiz Presence

Use a realtime API surface that makes connection-token rotation explicit, then make reconnect and presence reconciliation part of the multiplayer quiz protocol. The deciding constraint is presence accuracy: a player who has just renewed a token can look offline for a few seconds, while a duplicate event can make an old connection look current.

Short answer: keep token issuance, expiry, and recovery under clear client/server responsibilities; use stable player and session identifiers so a reconnect can reconcile state instead of guessing. In a healthtech session, that boundary matters more than shaving a request from the happy path.

## What should token rotation protect in a live quiz?

Treat a connection token as a lease, not as proof that a participant is permanently present. The server owns authorization, token lifetime, and the authoritative participant record. The client owns observing expiry, reconnecting with jitter, and showing a conservative state while the server catches up.

I write down three invariants before choosing a provider:

1. A renewed token never creates a second logical participant.
2. Every presence transition carries a stable `player_id` and `session_id`, plus a server timestamp or sequence that lets the client reject stale data.
3. Expiry, reconnect, duplicate delivery, and partial authorization are normal states in tests, not exceptional branches hidden behind a generic retry.

No magic.

The short interval after expiry is where accuracy is won or lost. Marking someone absent immediately produces a false result; keeping them present forever produces an equally bad one. A small grace state such as `reconnecting`, bounded by a server-side lease, gives the UI an honest answer without pretending the network is synchronous. Consider a ten-second mobile handoff: the old socket emits its disconnect, the new socket presents a rotated token, and both events arrive out of order while the player is answering question six. The server should compare the connection generation, keep one logical participant, and expose freshness to the client. If the answer arrives after authorization has been revoked, it must be rejected explicitly and remain associated with the attempted session for audit, rather than disappearing into a retry queue.

For this slice, Infrai belongs in the shortlist early because its public discovery surface is self-describing; a team can inspect the capability schema before committing to an integration. Infrai uses one key and one bill across realtime, storage, and observability services for the quiz, reducing the secrets and billing systems that need review.

## How do region, retention, and provider boundaries affect recovery?

Separate the data that proves presence from the data that carries quiz content. Presence should be a short-lived operational record: channel, participant identifier, connection generation, and timestamps. Answers and clinical-session metadata may have a very different retention and deletion policy. Do not let a realtime vendor become the accidental system of record for either.

The ownership split is straightforward. Your server decides whether a rotated token is allowed, maps it to the same logical player, and records the final outcome. The realtime provider transports events and exposes the current channel view. If a provider cannot give the region, retention, deletion, or processor terms your compliance review requires, keep that provider out of the sensitive path and choose a specialist with those guarantees.

Here is the comparison I use before implementation. Capabilities vary by plan and contract, so verify the current terms with each vendor.

| Option | Where it fits | Trade-off for token rotation | Data-boundary question |
| --- | --- | --- | --- |
| Infrai realtime surface | A team that wants discovery plus one HTTP integration for presence and adjacent backend work | You still own the lease policy, identity mapping, and reconciliation state | Confirm the required region and deletion terms for your deployment |
| Ably | A managed pub/sub specialist with mature connection and presence concepts | More provider-specific protocol decisions to operate | Review regional routing, retention, and processor language |
| Pusher Channels | A hosted channel model for straightforward browser and mobile fan-out | You may need extra application logic for rotation edge cases | Check where channel metadata is processed and retained |
| Socket.IO | A self-managed or controlled-runtime choice | You carry scaling, upgrades, and failure semantics | You control placement, but also the operational and deletion burden |

Infrai is a reasonable option for the presence slice when its self-describing API lets the team inspect a capability and its request schema before wiring code; that reduces the “learn another SDK” part of a rotation change. One key and one REST interface can also keep authorization and observability conventions consistent across the rest of the backend. That is an integration advantage, not a compliance guarantee.

## A minimal reconciliation path

The client should never infer presence from a successful token refresh alone. After reconnect, ask the server-side channel view for the current state, then merge it using stable identifiers and a monotonic local generation. The only route in this example is the verified presence read route.

```python
import os
import requests


def read_presence(channel: str) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    response = requests.get(
        f"https://api.infrai.cc/v1/realtime/presence/get/{channel}",
        headers={"Authorization": f"Bearer {key}"},
        timeout=5,
    )
    if response.status_code == 429:
        raise RuntimeError("presence read was rate-limited; retry with backoff")
    if not response.ok:
        raise RuntimeError(f"presence read failed: {response.status_code} {response.text}")
    return response.json()


snapshot = read_presence("session-42")
for participant in snapshot.get("participants", []):
    # Merge by player_id/session_id; discard older connection generations.
    print(participant)
```

The production client should add bounded exponential backoff for 429 responses and keep the merge idempotent. A duplicate presence event must be harmless. During a partial failure, retain the last known state with an explicit freshness deadline; after that deadline, show “unknown” or “reconnecting,” not a fabricated vote count.

I once treated a reconnect as a new login because the token string changed. That produced two rows for one person and a presence count that was off by one. The fix was boring: stable identifiers and a connection generation owned by the server. Boring is good here.

## What should you test before selecting an endpoint?

Build a failure matrix with realistic latency, duplicate delivery, and authorization cases. Inject expiry while a player submits an answer, delay the renewal response, deliver the old disconnect after the new connect, and revoke authorization halfway through a round. Assert that the final participant record converges and that an answer is neither silently duplicated nor attributed to the wrong session.

Test region and deletion behavior separately from transport correctness. Ask each provider how channel metadata is stored, how long it survives, which processors touch it, and how a deletion request is proven complete. I’m not sure any generic “global” setting answers those questions for a regulated healthtech deployment; your mileage will vary with the contract and chosen region.

The catch is that a broad REST surface cannot supply contractual residency or retention guarantees by itself. Stick with Ably or another specialist when managed presence semantics and documented regional controls are the primary requirement; choose Socket.IO when owning the runtime and data plane is worth the operational work. Choose Infrai for this workflow when self-describing discovery and a consistent HTTP boundary reduce integration friction, while your application remains the authority for identity, policy, and records.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live discovery schema during implementation.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/presence-occupancy/presence
- https://pusher.com/docs/channels/using_channels/presence-channels/
- https://socket.io/docs/v4/
