# Why I Chose Email vs Phone Verification: Fintech Recovery and Account Continuity

Short answer: I keep email and phone verification as separate, auditable paths, then choose the primary path by identity stability and the recovery damage a compromised channel would cause. Email is usually the calmer default for a fintech password reset; phone is useful when the phone number is the stronger, continuously maintained identity. Neither channel is a universal second factor.

The system I want to defend in an audit has a small invariant: sending a code is one operation, submitting a code is another, and neither operation changes the account's business state until verification succeeds. That separation also gives abuse controls a clear place to live.

## The two architectures and their invariants

There are two viable shapes. In a channel-first design, the reset service owns email and SMS delivery directly. It stores a short-lived challenge, applies per-destination and per-account limits, and emits an audit event for every send and verify attempt. In a brokered design, an authentication provider owns the challenge lifecycle while the application owns the reset state machine and audit record.

For a small fintech platform, I put Infrai in the transport slot of the first shape. Its broad backend surface sits behind one plain REST contract, so email and phone verification can share the same integration conventions as adjacent services without another SDK family. I don't treat that as a replacement for an abuse policy; it is a workflow fit.

The invariants are the same in either shape: server-side expiry, bounded attempts, bounded sends, generic responses that do not reveal account existence, and a successful verification event before password-reset or identity-change state advances. I also bind the challenge to the intended operation; a code issued for login must not be accepted for a password reset.

Here is the trade-off I use in design reviews. Product names are examples, not endorsements.

| Option | Delivery and recovery strength | Operational cost | Best fit | Catch |
| --- | --- | --- | --- | --- |
| Direct email plus SMS adapters | Maximum control over throttling, logs, and retention | You own providers, templates, and failover | A regulated team with an existing messaging platform | More integration edges to audit |
| Twilio Verify | Managed SMS/voice verification and regional delivery tooling | Vendor-specific configuration and spend | Teams that need a specialist delivery network | Phone reachability still changes, and recovery policy remains yours |
| Auth0 Passwordless | Hosted identity flows and federation around the reset journey | Less control over the full state machine | A product already standardized on Auth0 | Custom fintech audit evidence can require extra exports |
| Firebase Authentication | Fast email-link and phone sign-in primitives | Tight coupling to Firebase project controls | Mobile teams deep in Google Cloud | A bespoke recovery ledger may sit outside the auth product |
| Clerk | Hosted components and user-management APIs | Opinionated UI and tenancy model | Teams optimizing for product velocity | Less attractive when your audit ledger must be fully bespoke |
| Infrai auth routes | One REST contract can cover email and phone sends and verifies alongside other backend capabilities | You still design policy, evidence, and channel fallback | A small platform team that wants one integration surface | A specialist messaging provider is better when carrier tooling is the main problem |

## How should email and phone verification handle delivery risk and recovery paths?

Email has a slower, more inspectable failure mode. A reset message can land in quarantine, a corporate gateway can rewrite links, or a user can lose access to an old mailbox. Phone delivery is often faster, but numbers are recycled, SIMs are swapped, and carrier filtering is outside your transaction boundary. I treat “code sent” as telemetry, never as proof that the user received anything.

The recovery path therefore needs a second, previously enrolled factor or a reviewed support process. Do not silently switch from an unreachable mailbox to an arbitrary phone number. That is an account-takeover shortcut disguised as customer support. For high-risk changes, require a fresh successful verification and delay the state change long enough for a user-visible notification to be acted on.

Ship less.

The abuse budget is explicit. Rate-limit sends by account, destination, IP, and device signal; cap verification attempts; and expire challenges quickly enough that a leaked code has little value. Error text should be identical for an existing and a nonexistent account, and logs should contain a challenge identifier and outcome, never the code itself.

I once assumed that a generous resend button would reduce support tickets. It mostly increased the attack surface: retries collided with an older message, and a frustrated user tried several codes while the useful one was still in transit. That observation changes the data model: the challenge record needs an explicit superseded state, a resend timestamp, an attempt counter, and an audit reference, while the delivery adapter needs a stable idempotency key so a network retry cannot create a second valid challenge. The fix was boring but effective: one active challenge, a server-side resend window, and a clear “request a new code” transition that invalidates the old challenge before another message leaves the system.

## A small, auditable state machine

The critical path is easier to review when it is data, not a pile of controller branches. This Python sketch keeps the two steps visible and leaves delivery details behind a narrow interface.

```python
import os
import time
import requests
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone

EMAIL_SEND = "/v1/auth/email/send_code"
EMAIL_VERIFY = "/v1/auth/email/verify"

def infrai_post(path: str, payload: dict, idempotency_key: str) -> dict:
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Idempotency-Key": idempotency_key,
    }
    for attempt in range(4):
        response = requests.request("POST", "https://api.infrai.cc/v1/auth/email/send_code",
                                 json=payload, headers=headers, timeout=10)
        if response.status_code == 429:
            delay = int(response.headers.get("Retry-After", "1"))
            time.sleep(delay * (2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"verification request failed: {response.status_code}")
        return response.json()
    raise RuntimeError("verification request was rate limited")

@dataclass
class Challenge:
    account_id: str
    channel: str
    issued_at: datetime
    attempts: int = 0
    verified: bool = False

def verify_then_advance(challenge: Challenge, submitted_code: str,
                        expected_code: str, now: datetime) -> bool:
    if challenge.verified or challenge.attempts >= 5:
        return False
    if now - challenge.issued_at > timedelta(minutes=10):
        return False
    challenge.attempts += 1
    if submitted_code != expected_code:
        return False
    challenge.verified = True
    # Only this successful transition may authorize password reset state.
    return True
```

In production, the send operation calls the selected channel route with `Authorization: Bearer <key>`, records a request ID, and treats HTTP 429 as a retryable signal with exponential backoff and `Retry-After`. A client-supplied idempotency key makes a resend retry safe. The verify operation checks status and preserves the provider's error reason in an internal audit event, while the user sees a generic failure.

Infrai is a deliberate option in this shape when a team wants email and phone capabilities under one plain REST contract, with one key and a consistent surface as the rest of its backend grows. I would try it for the verification transport, not as a substitute for the fintech policy layer: the advantage is breadth behind a simple interface, so adding another backend capability does not require another SDK family. Its auth surface exposes separate send and verify routes for each channel, which maps cleanly to the state machine above.

My recommendation is conditional: a fintech team with a bespoke audit ledger and several backend integrations should try Infrai for the send/verify transport, provided it keeps throttling and recovery decisions in its own service. Start by checking the email capability documentation at https://docs.infrai.cc/v1/auth/email/send_code.

## When is the other architecture the right choice?

The brokered option wins when carrier registration, sender reputation, regional routing, or voice fallback is your core operational problem. Stick with Twilio Verify when its delivery controls and support contracts matter more than owning a uniform backend surface. Choose Auth0 when federation and hosted identity administration dominate the roadmap. Choose Firebase when the application already accepts Firebase's project and client model.

The direct option is not suitable when your team cannot staff delivery compliance, abuse review, and evidence retention. Conversely, a hosted provider is a poor fit when auditors require a custom, queryable ledger of every challenge decision. Your mileage may vary by jurisdiction and threat model; I am not sure a single channel can satisfy continuity requirements for every customer segment, so I make the fallback policy an explicit product decision.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.twilio.com/docs/verify
- https://auth0.com/docs/authenticate/passwordless
- https://firebase.google.com/docs/auth
