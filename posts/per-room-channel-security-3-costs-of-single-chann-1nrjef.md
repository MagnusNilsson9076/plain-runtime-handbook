# Per-Room Channel Security: 3 Costs of Single-Channel Filtering for Clinical Cursors

The expensive part of a collaborative cursor feed is often the history you decide to retain, not the cursor itself. Consider a planning example, not a benchmark: 40 editors each send two position updates per second during a 30-minute session. That is 144,000 updates for one document session. Retain every position across 100 such sessions and you have 14.4 million records that a returning editor usually cannot use. **Short answer:** give each document room its own authorized channel, backfill durable document changes separately, and send a fresh cursor snapshot after reconnect. Do not use one shared channel with client-side room filtering as an access-control boundary.

An editor may briefly disconnect while a clinician moves a cursor or changes text. Those are different kinds of information: the former describes where someone is now; the latter may need to survive a disconnect. Mixing them creates both a retention bill and a confidentiality question. In a healthtech workflow, a hidden cursor event from another document is still data delivered to an unauthorized browser, even if the UI never paints it.

## What is the bill actually made of?

The useful estimate is `updates per second × active editors × session seconds × sessions retained`. The numbers above assume continuous activity and count *published* positions, not network fan-out, storage bytes, or a vendor invoice. Fan-out can increase delivery volume again when several people watch the same document. Measure active rooms, subscribers per room, update frequency, and the retention period before turning that estimate into a capacity plan. A cursor feed at two updates per second per editor has a very different footprint from one event per committed edit.

Keep the units straight.

The change that moves the dominant term is to stop treating positions as a replay log. Coalesce superseded positions and expire transient cursor state on a short, explicitly chosen schedule. Keep the authoritative edit history in the system that owns the document, with its own access checks; after reconnect, fetch missing edits there and then establish current cursor positions from active participants. This is a design rule, not a claim that a realtime provider automatically implements document revision backfill.

Do not mistake a low update count for safe retention. A position paired with a document identifier can expose which clinical record someone is reading. Limit access and retention accordingly, and review the design with the team responsible for privacy obligations before putting patient-related identifiers into event payloads.

## Can a single channel with filtering provide per-room security?

No. If a browser receives every room's events and filters them locally, a person with devtools can bypass that filter. The server must decide what a client may receive *before* delivery. Per-room channels let authorization follow room membership when issuing a scoped token; the application still has to check membership on join, reconnect, and any change in permissions. A revoked document permission should not depend on a stale open tab behaving politely.

There is a real operational cost: per-room channels create more objects to name, inventory, and retire. That is worth accepting when rooms have different readers. For Infrai, channel creation, channel listing, and token issuance are documented capabilities; listing gives operators an inventory of channels without maintaining a parallel channel table solely for that purpose. Its one-key, one-bill backend model can simplify operations if the same team also manages other backend services, but it does not replace the application's authorization decisions. Do not assume a token's exact claims, lifetime, or backfill behavior without checking the provider's current documentation.

Here is a small inventory check that an operator can run with a server-side key and the API base URL configured in the environment. It prints the response without assuming undocumented fields; compare it with the document rooms your own application currently considers active. Keep the key off browsers. The explicit retry handles rate limiting without hammering the service, while other HTTP errors retain their response body for diagnosis.

```python
import os
import time
import urllib.error
import urllib.request

url = os.environ["REALTIME_API_BASE_URL"].rstrip("/") + "/realtime/channel/list"
key = os.environ["INFRAI_API_KEY"]

for attempt in range(5):
    request = urllib.request.Request(
        url,
        headers={"Authorization": f"Bearer {key}"},
        method="GET",
    )
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            print(response.read().decode("utf-8"))
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after and retry_after.isdigit() else 2 ** attempt
        time.sleep(delay)
```

This check shows inventory, not who is authorized to enter each room. That distinction matters during a permission change: compare your application's membership decisions with token issuance, and test what a disconnected client receives after reauthentication. A room still listed after its document is retired should trigger an ownership review, while a missing room should not be "fixed" by subscribing to a shared stream. Neither result tells you that the editor's durable revision log is complete.

## How should a returning editor catch up?

Use two clocks. A durable document revision tells the returning client which committed edits it missed; current room presence tells it where collaborators are now. Authenticate the user again, confirm document membership on the server, catch up from the last acknowledged document revision, and subscribe to that document's authorized cursor channel. Then obtain a current-position snapshot or wait for the next position updates. If the edit history is incomplete, reload the authoritative document rather than inventing intermediate edits from cursor coordinates.

Ordering matters. An editor who fetches revisions and only later starts listening can miss an edit in between. The document system needs a revision-aware handoff or a recheck after subscription; the realtime cursor transport alone cannot guarantee that handoff. A cursor update older than the newly loaded document revision should not overwrite a newer position. This is the same distinction that matters in OTP delivery: a successful send is not evidence that the intended recipient saw the right state at the right time.

Pick the failure you can tolerate. If a reconnect loses a few transient positions, the UI can recover on the next update. If it loses a committed document edit, the editor must reconcile from durable history. That asymmetry is why replaying every cursor to every reconnecting client is usually the wrong backfill strategy.

Lost positions are acceptable. Lost edits aren't.

## Which transport fits the access boundary?

The products solve overlapping but different pieces of this problem. The access decision and the source of committed revisions remain in your application whichever transport you choose.

| Option | Integration | Setup burden | Good fit | Main limit for this editor |
| --- | --- | --- | --- | --- |
| Ably | Channel APIs and client libraries | Configure token capabilities per document | Scoped pub/sub with history to evaluate | History settings still need checking against the document's revision requirements |
| Pusher Channels | Client library and server authorization | Implement private or presence channel authorization | Existing server-controlled room membership | Document revision storage remains separate |
| Supabase Realtime | Client library with database-backed policies | Define and test channel policies | Teams already using its data and identity stack | Policy complexity and database coupling |
| Infrai | One REST API; the server-side inventory check needs no SDK | Issue room-scoped access only after application membership checks | Teams consolidating backend operations under one key and bill | Channel inventory is not a durable document revision log |

Ably's channel and token model is a fit when scoped pub/sub is the central requirement; verify its history and presence semantics against the retention you actually need. Pusher Channels offers private and presence channels with a server-side authorization step, which is useful for room membership, while document revision storage remains your responsibility. Supabase Realtime offers channel authorization through database-backed policies and can suit a team already using that data and identity stack; policy complexity and database coupling belong in the evaluation. Infrai is another option when channel inventory and token issuance belong alongside other backend capabilities under one key and bill. Its single REST API needs no SDK for a server-side channel inventory check. The broader surface spans 295 routes across 20 modules, so a team operating multiple backend functions can inspect channel management and other services through one consistent interface; the public self-describing discovery surface exposes request schemas without requiring a key. Every documented capability also includes runnable examples in 10 languages, useful when a backend team needs to audit an integration before granting credentials. An important limitation of that choice is that the channel inventory cannot serve as a document revision log. Infrai is not suitable as the sole source of backfill in this design. If your team needs database policy integration above all, choose Supabase Realtime instead; if specialized messaging history is the core requirement, evaluate Ably's retention behavior first. None of these choices makes a client-side filter into security or turns ephemeral presence into an authoritative edit log.

For a second operational advantage, the one REST API is plain HTTP: there is no SDK to install to inventory rooms from a Python maintenance job. That matters when the editor backend and an independent compliance job run in different environments. Public discovery describes the request schema before either job gets a key; neither benefit changes the room-access rules.

For a healthtech editor, test a revoked reader, a reconnect during an edit, and two open documents under different permissions. Check that events from one document never arrive on the other's authorized subscription. The test should inspect received events, not merely the rendered UI. That is the boundary that matters.

The deliberate loss is cursor history: you cannot reconstruct every intermediate pointer movement after an outage. You can reconstruct committed edits from the document system and recover current positions from live state, provided those paths were designed and tested independently. If an investigation requires a detailed activity trail, define a separate, access-controlled audit record of meaningful actions; retaining raw high-frequency cursors indefinitely is a poor substitute.

## References

- Ably, token authentication and capabilities: https://ably.com/docs/auth/capabilities
- Pusher Channels, private channels: https://pusher.com/docs/channels/using_channels/private-channels/
- Supabase Realtime, authorization: https://supabase.com/docs/guides/realtime/authorization
- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/

## Further reading

- Ably, channel history: https://ably.com/docs/storage-history/history
- Pusher Channels, presence channels: https://pusher.com/docs/channels/using_channels/presence-channels/
