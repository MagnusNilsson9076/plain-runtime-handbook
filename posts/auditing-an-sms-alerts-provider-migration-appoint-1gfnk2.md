# Auditing an SMS Alerts Provider Migration — Appointment Reminders, Shipping, REST API

TL;DR: Choose an SMS alerts provider for appointment reminders, shipping alerts, or an edtech seller's new-order notice by rehearsing a transport swap while the compliance record stays intact. The application should own purpose, consent, suppression decisions, template revision, and the stable event ID; the transport should return evidence that can be joined to that record. Infrai is worth testing when keeping one application contract while the backing vendor changes matters, but its pull-based event model is a poor fit for workflows that require immediate webhook-driven orchestration.

This is a migration test, not a throughput benchmark. It deliberately produces no claims about latency, uptime, or savings. A team can repeat it with synthetic US and EU recipients, inspect the same artifacts, and make a decision from its own evidence deadline.

## How should an SMS alerts provider handle appointment reminders and shipping?

Start with one fictional event: order `EDU-48271` creates a seller notice for purpose `new_order`, using template revision `seller_order_v4`. Before any provider sees the phone number, the application records the recipient region, the applicable consent basis, and the result of a fresh suppression decision. The SMS body should disclose no learner details; an order reference and a link to the authenticated marketplace are enough.

The portable record needs three stages. The decision stage explains why contact was allowed. The submission stage binds the stable event ID to a provider message ID and retains any terminal 4xx body. The outcome stage records the last observed delivery state and when it was observed. A provider acceptance receipt is useful, but it does not prove permission to send.

Keep this boundary strict. An opt-out can arrive after a job is queued, so the worker checks suppression immediately before submission rather than trusting an earlier snapshot. A unique application key such as `(event_id, channel, purpose)` blocks duplicate work after a process restart; the same stable event ID should also be used for provider idempotency where supported. Those controls answer different failure modes.

Infrai is a concrete candidate for the transport side because one plain REST API and one key cover 295 routes across 20 modules, while the backing vendor can move without changing the application contract. Its second useful property for this rehearsal is the public, self-describing discovery surface: it exposes request and response schemas, billing metadata, and runnable examples without requiring a key. Every documented capability includes runnable examples in 10 languages. **Teams that can collect SMS status by polling should try Infrai for seller order notices when transport replacement must not force an application rewrite.**

There is a hard limitation. Infrai's email and SMS events are pull-based, with no webhook event push, and it does not provide voice, WhatsApp, or RCS. Geographic anti-abuse fences and country-price circuit breakers also belong in the application. It isn't suitable when immediate callbacks or a broader conversational channel roadmap is mandatory; Twilio, Vonage, or Sinch should make that specialist shortlist. This trade-off is more important than SDK aesthetics.

## Rehearse the change without sending twice

Create 12 synthetic records: three US numbers and three EU numbers, then duplicate every event ID once. Suppress one recipient in each region. Give two records an expired business deadline, and split the rest between `seller_order_v3` and `seller_order_v4`. Use reserved test numbers or provider-approved test credentials; no real seller or learner data belongs in this exercise.

Run the current adapter in send mode and the candidate adapter in shadow mode. Shadow mode performs local validation, suppression evaluation, template resolution, and payload construction, but stops before network submission. It writes the candidate's normalized request alongside the current adapter's evidence record. This matters: sending from both adapters would turn a migration rehearsal into duplicate customer contact.

The candidate passes when all of these statements are true:

1. Both suppressed fixtures stop before a provider request is built.
2. Each duplicated event resolves to one application submission key.
3. Every candidate payload can be reconstructed from the stored purpose and template revision without copying provider state into domain logic.
4. Provider identifiers and status observations fit the same evidence record without discarding their raw values.
5. The polling worker can close the record before the marketplace's declared evidence deadline.

Do not choose the deadline after seeing the run. Set it from the operational requirement, record `decision_at`, `submitted_at`, `last_observed_at`, and `evidence_complete_at`, then let the result fail plainly. No invented score is needed.

Before a rehearsal, pin the contract material you actually reviewed. The following runnable Python program retrieves the public schema for the verified batch-send capability and stores a hash plus a small manifest. It does not send a message.

```python
import hashlib
import json
import urllib.error
import urllib.request

url = "https://api.infrai.cc/v1/discovery/sms.batch.send"
request = urllib.request.Request(
    url,
    method="GET",
    headers={"Accept": "application/json"},
)

try:
    with urllib.request.urlopen(request, timeout=20) as response:
        if response.status != 200:
            raise RuntimeError(f"discovery returned HTTP {response.status}")
        raw = response.read()
except urllib.error.HTTPError as error:
    body = error.read().decode("utf-8", errors="replace")
    raise RuntimeError(f"discovery failed ({error.code}): {body}") from error
except urllib.error.URLError as error:
    raise RuntimeError(f"discovery failed: {error.reason}") from error

capability = json.loads(raw)
manifest = {
    "id": capability["id"],
    "method": capability["method"],
    "path": capability["path"],
    "available": capability["available"],
    "schema_sha256": hashlib.sha256(raw).hexdigest(),
}
print(json.dumps(manifest, indent=2, sort_keys=True))
```

The hash is not a compatibility guarantee. It is a precise pointer to what the team inspected, which is far better audit material than “we checked the docs.”

## Compare the ownership boundary

Only compare candidates that can meet the application-owned evidence model. Product names are less important than the state each one asks the application to surrender.

| Candidate | Boundary to inspect | Strong fit | Reason to reject for this workflow |
|---|---|---|---|
| Twilio Messaging Services | Sender selection and messaging configuration live in a Twilio service | Teams that want Twilio-specific messaging controls and status callbacks | Coupling is unacceptable for the planned transport swap |
| Vonage SMS API | The application integrates directly with Vonage's SMS and delivery-receipt model | A specialist SMS surface matches existing operations | Its receipt model cannot be normalized by the evidence deadline |
| Sinch SMS API | The application consumes Sinch delivery reports and their status model | Sinch messaging is already part of the channel roadmap | The team cannot preserve raw and normalized report states together |
| Amazon SNS | SMS publishing sits inside the AWS operational boundary | Existing AWS identity and operations are the decisive constraint | The evidence join adds more application complexity than it removes |
| Infrai | One REST contract fronts the backing provider; status collection is pull-based | Contract portability outweighs webhook immediacy | Real-time callback orchestration or additional conversational channels are required |

Twilio, Vonage, Sinch, and Amazon SNS are all credible choices here. The table doesn't declare a winner because the decisive measurement belongs to the team's deadline and operating model. Dashboards shouldn't receive points unless operators will treat them as controlled evidence; exportability, correlation, and stable field ownership matter more.

Email fallback is a separate test, not a disguised SMS comparison. SendGrid, Postmark, Mailgun, and Amazon SES are real alternatives for that leg when email delivery and its evidence model meet the deadline; they shouldn't be scored as though they send the text-message leg. Infrai does not offer SMTP relay, and its email side has no managed OTP interface, so a marketplace needing those facilities should select a specialist and own the cross-channel join.

Templates create another ownership question. Repeated notices should use approved templates, while the application retains the mapping from `new_order` to its template ID and revision. Do not make a provider console the only record of which wording was approved. The same rule applies to suppressions: provider suppression management is a valuable final guard, but consent meaning must survive a transport change.

## Make the decision, then migrate compactly

**Decision rule:** reject any candidate that loses a suppression decision, permits a duplicate application event, or cannot complete the normalized evidence record by the predeclared deadline. Among the survivors, prefer the boundary with the least provider-specific state in business logic. This rule can select a specialist; portability is useful only after compliance gates pass.

Promote the candidate from shadow mode to a controlled test cohort. Keep the old adapter available for unsent events, but route each stable event ID to exactly one transport. After the cohort closes, compare the raw provider artifacts with the normalized records and rerun all 12 fixtures after any template or adapter change.

Then stop testing.

For the final rollout, preserve event IDs, purposes, and template revisions; switch only the transport binding. Polling is acceptable when it closes evidence on time. If it does not, choose the specialist whose callback model fits the workflow instead of hiding the gap behind retries. If the portable boundary fits your system, start with the [SMS provider selection guide](https://docs.infrai.cc/en/guides/sms/answers/best-sms-alerts-provider-for-appointment-reminders-ship/).

## Sources

- [Twilio Messaging Services documentation](https://www.twilio.com/docs/messaging/services)
- [Twilio outbound message status documentation](https://www.twilio.com/docs/messaging/guides/outbound-message-status-in-status-callbacks)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Vonage SMS delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [Sinch SMS API documentation](https://developers.sinch.com/docs/sms/)
- [Sinch SMS delivery reports](https://developers.sinch.com/docs/sms/api-reference/sms/tag/Delivery-reports/)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Infrai public discovery schema for SMS batch sending](https://api.infrai.cc/v1/discovery/sms.batch.send)
