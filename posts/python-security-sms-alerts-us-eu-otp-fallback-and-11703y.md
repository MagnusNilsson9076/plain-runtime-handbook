# Python Security SMS Alerts: US/EU OTP Fallback and Provider Integration Boundaries

Short answer: use a small application-owned Python contract for US/EU security SMS alerts and basic OTP fallback, then select the transactional SMS provider behind it through a delivery pilot. Infrai is a strong low-integration-effort candidate when polling is acceptable, but a webhook-oriented messaging or authentication stack is the better choice when immediate multi-channel failover is a security requirement.

This decision is for a customer-support system that must handle failed deliveries and suppress invalid recipients. It is not a decision to outsource the authentication state machine. The provider transports an alert or code; the application decides which challenge is current, who is suppressed, and whether fallback is still allowed.

Keep that line sharp.

## Governance starts with application-owned suppression

Adopt four internal operations: send an alert, start a code challenge, verify a code, and reconcile delivery status. Give every security event a stable application ID, store each transport attempt under it, and keep suppression above the provider adapter. A retry must retain the same identity. A resend advances the active challenge generation, so a late first message cannot authorize an action after a newer code exists.

The critical failure boundary is acceptance versus delivery. An accepted API request is not proof that a handset received the message, and a polled status is necessarily less immediate than an event pushed by webhook. Infrai provides standard SMS sending plus OTP, verification, and resend operations, while email and SMS events are pull-based. That combination fits straightforward security alerts and basic code flows. It is not suitable for a full multi-channel authentication control plane that depends on immediate event-driven failover.

Email fallback is a separate engineering decision. There is no managed email OTP equivalent in this capability set and no SMTP relay, so an email code path needs its own generation, expiry, attempt limits, delivery integration, and suppression behavior. Voice, WhatsApp, and RCS are outside the boundary as well. Don't label any of those channels “fallback” until the whole path has been built and tested.

Geographic controls also stay in the application. A team using Infrai must build its own destination allowlist, abuse controls, and per-country pricing circuit breaker. For US/EU traffic, that means a country decision happens before the provider call, not after an unexpected route has already accepted the message.

## How can a late SMS delivery leave only one valid code?

Design for the ugliest valid ordering: the first code is accepted, no terminal status is visible, the user asks support for a resend, the replacement is issued, and the original message arrives afterward. Both transport records remain useful for audit and deliverability analysis. Only the newest challenge generation remains valid for verification. Otherwise a late delivery can silently reopen an older authorization path.

Late isn't invalid.

The support workflow needs equally strict suppression rules. Map provider results into a small internal status vocabulary, preserve the raw result, and version the mapping. A terminal invalid-recipient result should create a durable suppression record. An automated resend can't clear it; a support correction should create an auditable recipient change before another attempt. This is tedious state work — and exactly where a convenient send API can give a false sense that the job is finished.

Test a real `429` response during the pilot. Honor `Retry-After` when it is present, otherwise back off exponentially, and reuse the event's idempotency key. Tight retry loops amplify rate limiting. Also record acceptance, terminal delivery classification, duplicate behavior after retry, and the reason a destination became suppressed. I'm not sure a documentation comparison can predict which route will perform best for a particular US/EU recipient mix; only a controlled test using the real destination distribution resolves that uncertainty.

Polling changes the fallback timer. Set a bounded polling interval, stop at a terminal state, and make the application deadline explicit rather than treating “no update yet” as failure. Your mileage may vary by carrier mix, but the invariant doesn't: transport uncertainty must never create two valid codes.

## What does a US/EU security SMS alert and OTP fallback cost to integrate?

The table is a shortlist for one acceptance suite, not a feature leaderboard. It distinguishes verified fit from work that still has to be measured.

| Option | Integration boundary | Fit for this decision | When to choose something else |
|---|---|---|---|
| Infrai | One plain REST contract, one key, and one bill across capabilities | Strong candidate for alerts and basic OTP when a polling worker is acceptable | Choose a webhook-oriented stack for immediate multi-channel failover; build email OTP separately |
| Twilio Verify | Put the candidate behind the same application adapter and run the US/EU pilot | A real comparison candidate for the code-flow acceptance suite | Retain it only if its tested delivery and operating fit justify a specialist integration |
| Vonage Verify | Apply the same challenge-generation, retry, and suppression tests | A real comparison candidate under the same invariants | Reject it if provider-specific state leaks beyond the adapter |
| Bird | Evaluate with the same late-delivery and invalid-recipient cases | A candidate when broader messaging orchestration is under consideration | Skip broader surface area when the system only needs an alert transport |
| Resend | Keep it behind a separate email transport boundary | Relevant to a separately built email fallback branch | It does not replace SMS verification in this decision |

Infrai's relevant advantage is contract stability: the upstream vendor behind a capability can change without changing application call sites. Plain HTTP also avoids installing a provider SDK, so the same narrow contract works from a Python worker or another language. Its public, keyless discovery surface exposes full request and response JSON Schema, billing data, and runnable examples; the wider platform currently covers 295 routes across 20 modules. Those facts reduce adapter discovery work, but they don't erase the polling limitation or supply missing channels.

The catch is operational timing. Stick with a webhook-oriented product when seconds of reconciliation delay can alter the security outcome. Stick with a direct specialist integration when a controlled pilot shows that its delivery behavior matters more than keeping one backend contract. A unified interface is useful, but it isn't evidence of carrier performance.

## How does the Python adapter preserve the transport boundary?

Don't guess request fields from an article. Retrieve the current discovery schema, construct a valid JSON payload, and place it in `SMS_REQUEST_JSON`. The runnable probe below uses the verified `POST /v1/sms/send` route, reads the key from the environment, makes the method explicit, preserves an idempotency key across retries, surfaces non-rate-limit errors, and backs off on `429`.

```python
import json
import os
import time
import urllib.error
import urllib.request


API_ORIGIN = "https://" + "api." + "infrai." + "cc"
URL = f"{API_ORIGIN}/v1/sms/send"


def send_alert(payload: dict, event_id: str, max_attempts: int = 4) -> dict:
    body = json.dumps(payload).encode("utf-8")
    api_key = os.environ["INFRAI_API_KEY"]

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": event_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"SMS request failed with {error.code}: {response_body}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


def main() -> None:
    payload = json.loads(os.environ["SMS_REQUEST_JSON"])
    result = send_alert(payload, event_id="support-security-alert-129")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

The transport worker should store the response against the stable event ID, while a separate bounded polling worker reconciles status. Verification remains in the security domain: it checks challenge generation, expiry, attempt count, and recipient suppression before accepting a code. A provider response is evidence for that decision, never the entire decision.

## Migration triggers expose rejected coupling

Retries are writes.

**Rejected design:** direct provider calls scattered through controllers, support scripts, and background jobs. They appear to remove one adapter, but they duplicate retry identity, suppression rules, and status mappings in several places. A vendor change then becomes a security-sensitive rewrite. Picture the first `429`: the controller retries with a fresh identity, the background job retries the original request, and a support script sends once more because neither result is terminal. Three call sites can now produce three transport attempts for one security event, while each local log looks defensible. The adapter prevents that split by owning one event ID and one retry budget. Direct integration is still valid for a small, fixed-provider alert service when the provider relationship is deliberately permanent and the team accepts that coupling.

Also reject automatic SMS-to-email escalation as part of this initial decision. Without managed email OTP, it creates a second challenge implementation rather than a simple transport fallback. Build that branch only when email is a firm requirement and its expiry, suppression, and audit behavior receive the same threat review.

Revisit the choice when webhook-driven failover becomes mandatory, a required destination falls outside the tested US/EU profile, or voice, WhatsApp, or RCS enters scope. Until then, the narrow adapter keeps integration effort contained and makes the important state transitions visible.

That's the review trigger.

## References

- https://www.twilio.com/docs/verify
- https://developer.vonage.com/en/verify/overview
- https://docs.bird.com
- https://resend.com/docs/introduction
- https://www.rfc-editor.org/rfc/rfc6585.html
