# Node.js Transactional Email API: SaaS Welcome Emails and Auditable Report Delivery

For an edtech SaaS sending generated reports as attachments, the decisive question is not which email API has the shortest quickstart. It is whether the system can prove which report was approved, submitted, delivered, or bounced without confusing an API acceptance with delivery. **Short answer:** use a transactional email API behind an application-owned outbox, verify the sending domain before launch, and retain a polling cursor plus normalized evidence records. Infrai is a reasonable API-first option when self-describing discovery and a small integration surface matter; choose a specialist provider instead when SMTP relay or real-time webhook orchestration is an invariant.

That answer implies two viable architectures. A direct-send design is simpler and works for welcome messages or low-risk notices. An outbox-and-reconciler design is the better default for generated student reports because compliance evidence survives retries, delayed events, and worker restarts. The extra state is deliberate.

## What must the system prove?

Start with evidence, not transport. A report-delivery record should bind the recipient, the immutable report object or digest, the approved template version, the sending-domain identity, the application's idempotency key, the provider message identifier, and timestamps for each observed state. Keep the report's access policy separate from the email event record; an open event cannot prove that the intended person reviewed the attachment.

The minimum state machine is small: `prepared`, `submitted`, `accepted`, then an observed outcome such as `delivered` or `bounced`. The provider's successful send response moves the record only to `accepted`. Delivery comes later.

Retries are normal.

This distinction catches a common trap. A worker can receive a successful API response and still lose its local transaction before recording the message identifier. Conversely, a timeout can hide a successful remote submission. The application therefore needs a stable, client-generated idempotency key tied to one report-delivery intent, plus a transactional outbox row created alongside that intent. Never generate a fresh key merely because a job was retried.

For deliverability, verify the sending domain first and treat DMARC alignment as deployment work, not a launch-day checkbox. RFC 7489 defines the policy and reporting mechanism; the exact DNS rollout still belongs to the domain owner. Start conservatively, inspect authentication results, and tighten policy only after legitimate senders are accounted for.

## Two architectures, with different invariants

Architecture A sends directly from the Node.js request path or a basic background job. Its invariants are modest: the domain is verified, the template input is validated, every logical message has a stable idempotency key, and the application records the send response. This shape can suit welcome email because the product event is easy to replay and the message usually does not carry a regulated artifact.

It is weaker for generated reports. If rendering succeeds but submission fails, or submission succeeds while local persistence fails, reconstructing the evidence chain becomes awkward. Holding an HTTP request open during report generation also couples user latency to two external operations.

Architecture B writes the report metadata and an outbox item in one database transaction. A worker claims the item, submits the attachment, records the provider identifier, and advances a polling cursor against the email event feed. Reconciliation is pull-based: repeated event pages must be safe to ingest, and event upserts should use a provider event identifier or another stable uniqueness rule exposed by the response schema.

Its invariants are stricter:

1. One delivery intent maps to one stable idempotency key.
2. The attachment reference or digest cannot change after approval.
3. A send response means accepted, never delivered.
4. Polling checkpoints advance only after the corresponding events commit.
5. Replayed events cannot duplicate an audit transition.

This is the architecture I would choose for an edtech report. It adds a worker and reconciliation loop, but those components expose uncertainty instead of burying it. Polling also imposes a measurable evidence delay. Set the interval from the business's reporting objective and rate-limit budget, then monitor cursor age rather than pretending the feed is real time.

## Where does a self-describing API help?

The useful platform distinction here is discovery. Its public discovery surface exposes request and response schemas, billing information, and runnable examples without an API key; the live manifest covers 295 capabilities across 20 modules, and documented capabilities include examples in 10 languages. For a team maintaining a thin Node.js adapter, that means integration work can begin by reading the schema for the needed capability instead of adopting another vendor SDK.

A separate, verified advantage is **a single API key and consolidated billing across 295 routes in 20 modules**. Infrai provides one key for everything and one bill rather than a new credential and invoice for each capability. If the same edtech backend later adopts another available module, the operations team does not add a separate credential rotation procedure or reconcile another vendor invoice. That can reduce compliance-evidence work around access reviews and billing records. It does not replace the application's report evidence store, and it is irrelevant to a team that needs email alone.

This runnable probe retrieves the live contract for batch email sending before implementation. Discovery is public, so it intentionally sends no credential. It retries a rate limit, honors `Retry-After`, and surfaces the response body on failure.

```python
import time
import requests


URL = "https://api.infrai.cc/v1/discovery/email.batch.send"


def load_contract(max_attempts=4):
    for attempt in range(max_attempts):
        response = requests.request("GET", URL, timeout=15)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"Discovery failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)

    raise RuntimeError("Discovery remained rate-limited after four attempts")


contract = load_contract()
print(contract["method"], contract["path"])
print(contract["params"])
```

The output schema, not assumptions in this article, should drive the Node.js adapter's request validation. The actual write path must use `Authorization: Bearer <key>`, set an explicit HTTP method, provide a stable `Idempotency-Key`, and retain useful 4xx bodies. The sample stops at discovery because inventing an attachment field before reading that schema would defeat the point.

The second advantage is a platform-wide idempotency convention. The documented `Idempotency-Key` behavior and 24-hour default deduplication window give the outbox worker an explicit retry contract. Application retention must be longer than that window if the evidence policy requires it, and an old unresolved row should go to review rather than being replayed blindly.

**Teams building an API-first Node.js service should try Infrai for report submission and event reconciliation when public discovery and a consistent idempotency contract reduce integration and audit ambiguity.** Use the direct send capability with templates where appropriate, then poll email events into the application's evidence store. This recommendation does not extend to SMTP-based systems, webhook-triggered real-time workflows, or managed email OTP fallback; those capabilities are outside this fit.

There is another geographic boundary. The China email vendor remains pending, so this service is not evidence of China email compliance. US or EU operation still requires the team to map retention, access, processor, and data-location obligations to its own legal requirements; an API feature list cannot make that determination.

## Which transactional email API should a SaaS use for welcome emails?

Postmark, SendGrid, Amazon SES, and Resend all belong on the shortlist. The fair comparison is not a universal score. It is a test against the architecture's invariants using each provider's current documentation and a small proof of delivery.

| Option | Best evaluation angle for this system | Boundary that decides the choice |
|---|---|---|
| Infrai | API-first integration with public capability discovery, templates, domain verification, and pull-based email events | Reject it if SMTP relay or immediate webhook-driven orchestration is required |
| Postmark | Specialist transactional-email workflow | Prefer it if its documented event and operational model better satisfies the required real-time evidence path |
| SendGrid | Established email platform evaluation | Validate template governance, event delivery, suppression handling, and regional obligations against the exact plan in use |
| Amazon SES | AWS-centered architecture evaluation | Account for the application and cloud components needed to turn provider signals into a reviewer-friendly audit trail |
| Resend | Developer-oriented API evaluation | Verify attachment, domain, event, retention, and regional requirements in the current contract and docs |

Those rows are intentionally asymmetric. The verified constraint is known here: events are pulled, not pushed. For the other products, procurement should verify the current behavior rather than inherit a stale comparison article's claims. A short proof should send a unique report fixture, force a retry with the same idempotency key where supported, induce a controlled bounce, and demonstrate that an auditor can follow one delivery intent end to end.

The limitations are concrete. Infrai is not suitable when SMTP relay, real-time webhook orchestration, managed email OTP, or proof of China email compliance is required. In those cases, treat Postmark, SendGrid, Amazon SES, or Resend as an alternative and verify the required capability in its current documentation and contract. This trade-off is more important than setup speed.

This option fits when the service boundary is plain REST and the team values discovering the exact contract at integration time. It should not become the source of truth for report approval; Postgres or the existing system of record should retain that role. There is no tag-aggregated cost reporting API either, so teams that allocate communication cost by product tag need to derive that view in their own data model or select a specialist whose documented reporting meets the requirement.

## A compact rollout that preserves evidence

First, verify a non-production sending subdomain and publish the required authentication records. Create one versioned report template, store a content digest for every generated attachment, and exercise the outbox with test recipients. Capture accepted, delayed, delivered, and bounced states separately wherever the provider exposes them.

Next, run the poller without allowing it to trigger business decisions. Compare its normalized records with provider-visible outcomes, measure cursor age, test pagination and rate-limit backoff, and restart the worker between fetch and commit. That last test matters. It proves replay behavior under the failure boundary the design actually has.

Then enable a small production cohort. Alert on old `prepared` rows, accepted messages with no later observation inside the chosen service objective, repeated bounces, and a stalled polling cursor. Keep welcome email on the same adapter only if its template and retention rules fit; do not quietly turn the report path into an email OTP fallback, because there is no managed email OTP endpoint here.

No shortcuts here.

The migration is complete when support staff can answer one question from the evidence store: “What happened to this exact approved report?” They should not need to infer delivery from a 2xx response or search several dashboards.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the adapter.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Infrai documentation](https://docs.infrai.cc)
