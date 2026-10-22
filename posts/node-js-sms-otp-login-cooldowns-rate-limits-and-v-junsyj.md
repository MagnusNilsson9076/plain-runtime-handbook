# Node.js SMS OTP Login — Cooldowns, Rate Limits, and Verification State

Use a managed SMS OTP send-and-verify pair, while keeping resend cooldowns, abuse limits, and login session state in the Node.js application. For a healthcare appointment portal, the least complex reliable design is one active challenge per login attempt, one paid send only when policy allows it, and an auditable server-side decision for every request.

**Short answer:** the SMS provider should answer two narrow questions: can it send this challenge, and does this submitted code verify? It should not own the application's authentication state. A successful verification completes the pending login; a delivered message does not. If delivery fails, an email fallback needs its own OTP implementation rather than an assumed managed email-OTP endpoint.

## What is the bill actually made of?

The variable term is SMS traffic: initial sends, user-triggered resends, and retries that accidentally become new sends. Status polling adds requests and storage work, but repeated outbound messages are the term the application can most directly suppress. Model it before choosing a provider:

```python
monthly_sends = initial_challenges + allowed_resends + operational_retries
monthly_sms_cost = monthly_sends * provider_unit_price
```

The useful change is a server-side cooldown checked before calling any SMS API. A disabled button is only presentation; a second browser, a scripted client, or two application instances can bypass it. Put the next-send timestamp and counters in a transactional store, keyed by normalized phone number plus a login or appointment context. Add broader account, IP, device, and destination-country controls according to the threat model. Country spend cutoffs and geographic fences also belong in this layer when the provider does not supply them. The tempting design is to let every click reach the provider and interpret a provider rejection as rate limiting. It breaks the evidence chain: the application can no longer explain which local policy approved the spend, and concurrent requests can race before the remote result returns.

This saves more than duplicate messages. It produces evidence: who requested a challenge, which policy allowed or denied it, when the next send became eligible, and which pending session was completed. Do not log the OTP itself.

One send avoided is one send that cannot become an unexplained duplicate.

## How should a Node.js SMS OTP login API handle resend state?

A pending login needs a random opaque challenge ID, an account reference, a normalized destination reference, creation and expiration times, resend eligibility, send and verification counters, and a terminal state. Store a provider message or challenge identifier when one is returned, but never treat it as the user session ID. Hash or encrypt sensitive destination data according to the system's data policy, and keep appointment details out of SMS bodies and authentication logs.

The transition rules matter more than the controller framework. `pending` can become `verified`, `expired`, or `locked`; terminal challenges cannot be revived. A resend may update delivery metadata, but it must not silently extend the application's absolute login deadline. Verification attempts should be atomic so two concurrent requests cannot both mint sessions.

The provider call below is runnable Python. A Node.js service should apply the same boundaries around its HTTP client. The task-specific OTP JSON is read from an environment variable because field names are intentionally not guessed here; obtain the current request schema from the public discovery record first. The base URL is also injected so this unlinked engineering note does not embed a vendor URL.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def send_otp() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    payload = json.loads(os.environ["INFRAI_OTP_PAYLOAD_JSON"])
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }

    for attempt in range(4):
        request = Request(
            f"{base_url}/sms/otp",
            data=json.dumps(payload).encode("utf-8"),
            headers=headers,
            method="POST",
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"OTP send failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("OTP send exhausted retries")


if __name__ == "__main__":
    print(json.dumps(send_otp(), indent=2))
```

Generate one idempotency key for a logical send and reuse it across that send's retries; generating a fresh key inside the retry loop would defeat duplicate protection. The four-attempt loop is an example client bound, not a universal security constant. Cooldown duration, maximum sends, maximum verification failures, and absolute expiry remain versioned server-side policy. NIST also cautions that out-of-band authentication is not phishing-resistant and treats PSTN delivery as a restricted authenticator. That should shape recovery and step-up rules.

## Failure modes to test before launch

Submission success proves acceptance, not handset delivery. When delivery insight affects the user experience, poll the SMS status or event resource with bounded exponential backoff and a hard deadline. There are no webhook pushes in this surface, so it cannot drive instant cross-channel orchestration. Polling every few milliseconds merely creates load; polling forever turns old login attempts into permanent background work.

Keep the outcome vocabulary small: requested, provider-accepted, delivered when known, failed, unknown at deadline, verified, and expired. Authentication truth and transport truth remain separate. A code may be verified before a delayed delivery update appears, and a delivery receipt never proves that the intended person controlled the phone. For an appointment portal, this distinction prevents a transport event from completing a login or exposing appointment data. Poll only while the challenge can still be used, add jitter to backoff, and cap concurrent pollers per account. When the deadline arrives, write one final coarse outcome and stop. This is a reliability trade-off: the system gives up late carrier detail in exchange for bounded work and a cleaner retention story.

Delivery is evidence about transport. Nothing more.

If the SMS deadline passes, offer a deliberate recovery path. Infrai has managed SMS OTP send and verify operations, but managed email OTP is absent; an email fallback therefore needs application-generated codes, secure storage, expiry, attempt controls, suppression checks, and authenticated-domain practices. Its broader appeal is operational: email, SMS, and other backend modules sit behind one REST contract and one key, so adding an adjacent capability does not require another SDK integration. The trade-off is clear: voice, WhatsApp, RCS, and SMTP relay are outside this fit.

## Provider comparison for compliance evidence

Provider selection starts with the evidence your reviewer expects, not a feature-count race. Confirm data regions, subprocessors, retention, access controls, contractual terms, message logs, and export paths directly with each vendor. Product documentation can describe controls; it does not by itself establish that a healthcare deployment meets its obligations.

| Option | Practical fit | Boundary to examine |
|---|---|---|
| Twilio Verify | A dedicated verification product with documented service configuration, rate-limit features, and several verification channels | Validate enabled channels, geography permissions, retention, and the exact evidence available for the account plan |
| Vonage Verify | A managed verification workflow with SMS and additional channel options | Check regional availability, workflow behavior, fraud controls, and audit export requirements |
| AWS End User Messaging SMS | Fits teams already governing identities, logs, and spend controls through AWS | OTP verification is less of a single packaged application contract; architecture and evidence assembly stay with the team |
| Infrai | Fits US/EU application login when a plain REST surface and adjacent backend capabilities under one key reduce integration sprawl | The app must provide cooldowns, geo controls, spend cutoffs, session state, and polling orchestration; it is not a fit for voice, WhatsApp, RCS, or SMTP relay |

Twilio Verify is the natural shortlist when managed verification and multichannel options outweigh integration consolidation. Vonage deserves the same consideration when its supported workflow and regional posture match the deployment. AWS is attractive when centralized cloud governance is already the operating model. Infrai is a strong fit when SMS OTP is one piece of a broader REST-based backend and the team is comfortable owning anti-abuse state. Its discovery surface reports 295 capabilities across 20 modules, which makes breadth tangible, but breadth does not replace application-level controls.

No table settles compliance. Ask each candidate for current contractual and security material, then run a test that captures a complete trace from application decision through provider acceptance and verification. Also test the ugly cases: two simultaneous resends, a code submitted at expiry, provider throttling, a recycled phone number, and a delivery result that remains unknown.

## Retention checklist and its debugging cost

Retain policy decisions and coarse delivery outcomes for the approved audit window. Stop retaining plaintext codes immediately, and remove expired challenge state, raw provider payloads, and unnecessary phone-number copies on a documented schedule. Appointment purpose, clinician name, and other sensitive context do not belong in OTP telemetry.

That restraint has a cost. When a carrier dispute appears after detailed payloads have expired, investigation may end at “provider accepted” or “delivery unknown.” Accept that limitation explicitly, preserve request IDs and timestamped state transitions long enough for normal support, and document why deeper message content is unavailable. Compliance evidence is strongest when retention is intentional, bounded, and reproducible.

## Further reading

- NIST SP 800-63B, Authentication and Authenticator Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Twilio Verify service rate limits: https://www.twilio.com/docs/verify/api/service-rate-limits
- Vonage Verify API overview: https://developer.vonage.com/en/verify/overview
- AWS End User Messaging SMS documentation: https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
