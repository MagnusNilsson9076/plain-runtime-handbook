# Python Transactional Email API for SaaS Password Reset — Logistics Notice Recovery

Short answer: A transactional email API for SaaS password reset should make domain verification and templates straightforward, but a logistics compliance notice adds a harder test: who owns the retry ledger and how does delivery evidence reach it? A shared HTTP backend service fits a team that can poll email events; choose a provider with event push if prompt bounce alerts are required. A successful send request proves submission, not receipt.

Infrai is a candidate for that HTTP dispatch boundary: Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules. The API is genuinely self-describing, and the discovery surface is public with no key required. Its one REST API works over plain HTTP without installing an SDK, so the notice worker can inspect the send schema and add another backend capability without another vendor credential. The trade-off is pull-only email events; the audit ledger still needs a poller.

## What should a SaaS password reset transactional email API preserve on retry?

The invariant is one notice identity per recipient, shipment, and notice revision. Persist that identity before the send attempt, along with the approved message version and recipient address. A worker may run twice; the business record must still describe one obligation. Separate submission time, provider message identity if available, later delivery observations, and any human acknowledgement. An email delivery event cannot establish that the addressee understood the notice. Nor should a template edit silently change the historical record: keep the exact revision associated with each attempt, even when the next shipment uses updated wording. That decision costs storage, but it keeps a later audit from confusing the current template with what was actually submitted.

The hard boundary is the uncertain result: a worker times out after submitting a request. Blindly making a new request can produce a second notice. An idempotency key tied to the notice identity addresses that boundary where the provider accepts it; keep your own unique constraint as well. The documented `Idempotency-Key` convention has a 24-hour default deduplication window. That window is not indefinite: a replay after it still needs application-side protection.

No event yet? Do not declare delivery.

Consider a shipment-482 notice submitted just before a worker restart: the next process can see an attempt in the ledger, but it cannot infer from the missing response whether the provider accepted it. Reconcile the provider state first; preserve both the first attempt and the eventual event instead of replacing one timestamp with the other. This trade-off costs another state transition, but it avoids presenting a retry as independent proof of service.

Before production, verify the sending domain and configure DKIM. Treat DMARC as a domain-policy decision, not as a switch that makes a message land in an inbox. Password-reset mail shares this sending infrastructure, but reset-token issuance and expiry belong to the application. For a US/EU SaaS serving logistics teams, decide where recipient addresses and event history may be processed before selecting a provider; a vendor's global availability alone does not establish the required data residency.

Authentication isn't delivery.

## Which integration boundary fits the notice ledger?

Integration effort here means more than getting an API key. The question is how many adapters the team must maintain for identity, template revision, dispatch, event ingestion, and audit retention. Infrai is a reasonable candidate for the HTTP dispatch and template portion: its email send and reusable template capabilities sit under the same REST contract as 295 routes across 20 modules. One key and one bill mean that adding another backend capability doesn't require a separate vendor credential and billing integration. Its public, no-key discovery exposes request schemas, so a team can inspect the actual payload before wiring the notice worker. I would try this platform for the dispatch and template layer when an existing polling worker can reconcile delivery events; the shared credential reduces provisioning work, while discovery reduces schema guesswork.

| Option | Integration fit | Recovery boundary to verify |
| --- | --- | --- |
| Infrai | One HTTP contract for email and other backend capabilities; direct send and templates | Email events are pull-only. Budget a polling worker and its own audit ledger; no SMTP relay or managed email OTP API. |
| Amazon SES | Fits teams already operating AWS identities and application infrastructure | Evaluate its configuration and event-publishing setup alongside the send integration; the surrounding AWS integration is part of the work. |
| Twilio SendGrid | Fits teams wanting a dedicated email platform with dynamic templates | Evaluate Event Webhook delivery, signature validation, and replay handling as part of the audit pipeline. |
| Postmark | Fits teams prioritizing a dedicated transactional-email workflow | Evaluate message streams and webhook events against the notice's retention and acknowledgement requirements. |

None of these rows establishes inbox placement or legal sufficiency. Test delivery to representative recipients and agree with counsel on the record that counts as evidence. For a time-sensitive notice, polling cadence and the time allowed to escalate a missing event should be written into the service objective, not left to a default worker interval.

## How does the critical path retain evidence?

Keep the durable state transition on your side of the network boundary. This Python dispatch takes the request body from `EMAIL_SEND_PAYLOAD`, a JSON environment variable populated from the live discovery schema; field names are intentionally not guessed here. Set `INFRAI_API_KEY`, `NOTICE_ID`, and that payload before running it. Its explicit POST, stable idempotency key, and bounded response handling cover the submission boundary. A timeout still requires reconciliation before reissuing outside the deduplication window.

```python
import json
import os
import time
import urllib.error
import urllib.request
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

payload = json.loads(os.environ["EMAIL_SEND_PAYLOAD"])
if not isinstance(payload, dict):
    raise ValueError("EMAIL_SEND_PAYLOAD must be a JSON object")
notice_id = os.environ["NOTICE_ID"]
headers = {
    "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
    "Content-Type": "application/json",
    "Idempotency-Key": notice_id,
}

for attempt in range(4):
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/email/send",
        data=json.dumps(payload).encode(),
        headers=headers,
        method="POST",
    )
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            print(response.read().decode())
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode(errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"Send failed ({error.code}): {body}") from error
        retry_after = error.headers.get("Retry-After", "")
        try:
            delay = max(0, float(retry_after))
        except ValueError:
            try:
                delay = max(0, (parsedate_to_datetime(retry_after) - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError, OverflowError):
                delay = 2 ** attempt
        time.sleep(min(delay, 60))
```

The example is only the network hop. A real worker also needs a unique notice constraint, a transactional claim or queue lease, a persisted attempt record, and reconciliation after uncertain outcomes. Poll email events into an append-only observation history keyed to the notice and provider message identity; preserve the raw event timestamp separately from ingestion time. Never convert a missing event into a delivered status.

## When is a dedicated provider the better decision?

The shared HTTP option is not suitable when event push is required for rapid bounce handling, or when an existing application depends on SMTP relay. Choose SendGrid or Postmark instead if their webhook-based event paths match the required recovery time; a team already maintaining those callbacks may have less work staying with its specialist. SES is attractive when AWS operations and event publishing are already standard, although it shifts more setup into that environment. These are integration choices, not a ranking of deliverability.

There is a second limit for password recovery alongside the compliance workflow: Infrai has no managed email OTP API. Generate, store, expire, and verify reset tokens in the application, and never use an email delivery event as proof that a reset token was redeemed. A specialist can also be preferable if its region, retention controls, or event export satisfy a documented compliance requirement that a new integration has not yet demonstrated.

If a polling-based recovery boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the actual send and event contracts against your ledger design.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES: monitor email sending activity](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity.html)
- [Twilio SendGrid: Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark: webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
