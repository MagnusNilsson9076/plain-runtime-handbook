# Email Service Evidence for Password Reset, Welcome Deliverability, and Bounce Tracking

Short answer: for password reset and welcome email, choose an API-first service only after it proves dedicated-domain sending, suppression enforcement, and bounce tracking with evidence your fintech team can retain; no SMTP is fine for a new backend, but it is the wrong constraint for an SMTP-only legacy system.

The important decision is not which provider promises the best inbox placement. Nobody can promise that. Reputation, authentication, message content, recipient behavior, and mailbox policy all sit outside the send API. The decision is whether your system can demonstrate what it attempted, what happened, and why it did or did not send again.

That is a compliance problem wearing a deliverability costume.

Keep it auditable.

## What governance evidence should a fintech email path retain?

A password reset has a security deadline. A welcome message has a customer-experience deadline. Both also create an auditable communication event. For every request, retain an application operation ID, recipient hash or protected address, template version, consent or account-recovery reason, sending domain, provider message ID, normalized outcome, and observation time. Keep the raw delivery event separately, with access controls and a retention policy.

Do not put reset tokens in logs, bounce notes, analytics labels, or support exports. A message ID is evidence of a transmission attempt; it is not evidence that a recipient saw the message. That distinction belongs in the data model and in the runbook.

The minimum state machine should distinguish \`queued\`, \`submitted\`, \`delivered\`, \`deferred\`, \`permanently_failed\`, and \`suppressed\`. A temporary deferral can enter a bounded retry policy. A permanent bounce should update suppression before the next request reaches the transport adapter. A suppressed address should produce a recorded decision, not a mysterious success response that leaves the account service believing an email was sent.

The same boundary handles duplicate clicks. A user can request three reset messages in a few seconds; a client can retry after a timeout; a worker can lose its connection after the remote system accepts a request. Make the application operation ID unique and make the outbox write idempotent. Delivery status may arrive later, out of order, or more than once, so status updates need the same property.

Here is a deliberately small policy boundary. It is not an email provider client. That is the point: suppression and duplicate control should remain testable even if the transport changes.

```python
import sqlite3


SCHEMA = """
CREATE TABLE suppression (
    address TEXT PRIMARY KEY,
    reason TEXT NOT NULL,
    evidence_id TEXT NOT NULL
);
CREATE TABLE outbox (
    operation_id TEXT PRIMARY KEY,
    address TEXT NOT NULL,
    template TEXT NOT NULL,
    state TEXT NOT NULL CHECK (state = 'queued')
);
"""


def queue_message(connection, operation_id, address, template):
    with connection:
        if connection.execute(
            "SELECT 1 FROM suppression WHERE address = ?",
            (address,),
        ).fetchone():
            return "suppressed"

        result = connection.execute(
            """
            INSERT OR IGNORE INTO outbox(operation_id, address, template, state)
            VALUES (?, ?, ?, 'queued')
            """,
            (operation_id, address, template),
        )
        return "queued" if result.rowcount else "duplicate"


database = sqlite3.connect(":memory:")
database.executescript(SCHEMA)
assert queue_message(
    database, "reset:acct-71", "user@example.com", "password_reset_v4"
) == "queued"
assert queue_message(
    database, "reset:acct-71", "user@example.com", "password_reset_v4"
) == "duplicate"

database.execute(
    "INSERT INTO suppression(address, reason, evidence_id) VALUES (?, ?, ?)",
    ("invalid@example.com", "permanent_bounce", "evt-884"),
)
assert queue_message(
    database, "welcome:acct-72", "invalid@example.com", "welcome_v2"
) == "suppressed"
```

The worker that drains this outbox still has serious work: authenticate the HTTP request, enforce a timeout, record the response identifier, and map delivery observations into the internal state machine. The account service should not need to know the transport's field names or retry quirks. I've chased a `429` through a queue before: the first retry was harmless, but an unbounded retry loop turned a provider limit into our own traffic storm. Back off, honor `Retry-After` when supplied, cap the attempt budget, and leave an operator-readable reason when the budget expires. That record is more useful than a green worker metric.

## Can a dedicated domain and API-first email path handle password reset and welcome mail without SMTP?

Use a dedicated subdomain for transactional traffic, verify the required authentication records, and separate it from marketing mail. A dedicated domain does not grant inbox placement. It gives reputation and operational ownership a clearer boundary, which makes evidence easier to interpret. Warm it gradually, send stable content, and watch complaint, bounce, deferral, and delivery signals together.

An API-first path is a good fit for a new fintech backend when the application already has an HTTP client, needs one integration across languages, and does not need an SMTP relay. Plain HTTP keeps the adapter explicit: request, response, identifier, retry classification, and audit record. No SDK has to be installed before a small service can send a message.

The boundary is equally clear. No SMTP is unsuitable for a legacy framework, appliance, or vendor package that only exposes an SMTP configuration screen. Use an SMTP-capable path there, or isolate that dependency behind a small service instead of quietly rewriting the account system. A dedicated domain also does not solve a weak authentication setup, an untrusted template, or an address list that ignores suppression.

For a reset message, test more than a 200 response. Assert that the link expires, that the token is single-use, that the response does not reveal account existence, that repeated requests have a policy, and that the message renders in plain text as well as HTML. Welcome mail deserves the same template discipline, though its retry and latency budget can differ.

A useful acceptance record looks like this:

| Check | Evidence to retain | Failure response |
| --- | --- | --- |
| Domain authentication | DNS change, verification result, timestamp | Stop production traffic |
| Permanent bounce | Raw event, normalized reason, suppression ID | Block future automatic sends |
| Temporary deferral | Event sequence and retry decision | Retry within a bounded budget |
| Duplicate request | Operation ID and one outbox row | Return the existing decision |
| Reset rendering | Expiry, redaction, plain-text capture | Fix template before rollout |
| Operator lookup | Request ID to message ID to outcome | Page the owning team |

The table is intentionally boring. Compliance evidence should be boring.

## How do bounce failures change a retry test?

A dashboard count is not enough. The reviewer needs a chain from business intent to transport result: an account event caused a send decision; the policy boundary checked suppression; the adapter submitted one request; the service returned an identifier; feedback changed the normalized state; and a later attempt respected that state. Store timestamps in a consistent format and preserve the correlation key across queues.

Bounce categories need an operational meaning. Permanent failures should normally suppress the address until a deliberate, evidenced revalidation. Temporary failures should not become permanent merely because one worker retried too aggressively. Unknown feedback should be quarantined for review rather than treated as permission to keep sending.

This is where teams often create a compliance hole: they retain the event but not the decision. The raw payload says an address bounced. It does not say which rule blocked the next reset request, who changed the suppression record, or whether the change was later reversed. Log the decision and its actor or automation version.

I would also test the negative path first. Insert a known invalid recipient, request a welcome message, and prove that no transport submission occurs. Then deliver a valid reset, replay its feedback twice, and prove that the final state and audit record remain stable. A passing happy-path test tells you almost nothing about suppression.

Your mileage may vary on the retention period. Legal requirements, account records, regional policy, and the sensitivity of raw addresses determine the answer; document that decision instead of copying a number from another system.

## What should the integration contract expose before a service is selected?

Only if the integration boundary and the evidence contract both pass a production-shaped trial. Compare candidates on questions that can be observed: can the service send from the verified dedicated domain, return a stable message ID, expose bounce and delivery feedback, enforce suppression, handle rate limits, and support the required data-retention controls? Treat pricing as a procurement input, not as proof of deliverability.

A provider with webhooks may suit a workflow that needs low-latency event delivery. A provider with polling may suit a simpler backend that can tolerate an explicit observation delay. Neither choice removes the need for idempotency. With webhooks, authenticate and replay-protect inbound events. With polling, persist a cursor, use overlap at page boundaries, and deduplicate event IDs. Don't call a delayed observation a delivery guarantee.

Managed OTP is another boundary. A general email send API does not automatically provide code generation, expiry, guess limits, resend throttling, or account-enumeration protection. Build those controls in the authentication service or select a documented authentication capability. Do not infer it from a welcome-email feature.

SMS fallback adds consent, opt-out, sender identity, geography, and message-content obligations. CTIA guidance is a useful US reference, but a fintech team still needs jurisdiction-specific review. Keep email and SMS policy decisions separate so an email bounce does not accidentally authorize an SMS send.

The right answer may be to stay with the current SMTP path. Stick with it when a mature relay is already isolated, its evidence is sufficient, and replacing it would create more migration risk than control. Move to HTTP when the backend benefits from a direct API, the old relay hides useful status, or the team needs a transport adapter that can be tested in ordinary application code.

## How does a rollout prove suppression before production?

Start with a shadow decision: run suppression classification and template checks without sending. Next, use a small verified audience and record every request, response identifier, and feedback transition. Exercise malformed addresses, duplicate operation IDs, rate-limit responses, delayed events, replayed events, expired reset links, and an operator's request for the full evidence chain.

Set explicit release gates. No production send before authentication is verified. No retry policy before permanent and temporary failures are distinguished. No launch before a suppressed address is proven not to reach the adapter. No compliance sign-off before the raw event, normalized state, and decision record can be joined without manual database archaeology.

Keep the provider adapter replaceable, but do not make the audit model vague. The account team owns intent and authorization. The communication worker owns submission and retry. The feedback consumer owns state transitions. Compliance and security own retention, access, and review criteria.

That division makes the eventual provider choice smaller than it first appears. The difficult part is deciding what the system must prove.

## References

- Amazon SES official documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- CTIA messaging interoperability and compliance best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
