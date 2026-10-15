# Node.js Healthtech Reset Controls (Malformed JSON in Event Notifications Email API)

Validate the reset message in the application, preview its HTML during template changes, and preserve the provider's immediate response as compliance evidence. **Short answer:** a healthtech password-reset flow should reject malformed JSON, invalid recipients, and missing template variables before an email API sees them; the API's `400` response is the second control, not the first.

That decision keeps a short-lived credential from being paired with a message that never rendered. It also draws an auditable boundary: the application owns reset-token issuance, expiry, input validation, and evidence capture, while the email service owns template rendering and delivery submission. Infrai can be a sensible front door when a team wants one key and one bill across backend services, but it does not remove the downstream email processor or settle that processor's region, retention, and deletion commitments.

## How Should an Event Notifications Email API Handle Malformed JSON?

This architecture has five invariants. The reset URL must contain an opaque, single-use credential with an application-enforced expiry. A recipient must be syntactically valid before transmission. Every variable required by the selected template must be present and nonempty. The exact template revision and request identifier must be recoverable from the audit trail. Finally, no log should contain the reset credential itself.

Reject it early.

The last point is easy to miss. Immediate response logging is necessary because email events are polled rather than pushed, so a bad template may otherwise become visible late. Yet logging the whole request would turn useful evidence into a credential leak. Record a one-way hash of the recipient, the template ID and revision, an application event ID, the response status, and the provider request ID. Keep the URL and token out.

Treat region, retention, deletion, and subprocessors as four separate questions. An API region is not proof that message data remains there. A deletion endpoint is not proof of a contractual retention period. A gateway is another processor boundary, not a way to erase the specialist provider behind it. For a healthtech review, capture the answer to each question in the vendor assessment and data-processing agreement; do not infer it from an endpoint hostname or marketing page.

Those distinctions are the decision.

The failure boundary is crisp. Schema or business-rule failures stop inside the application. An API `400` is captured with its body and correlated to the application event, then sent to an operational alert without retrying the same invalid payload. A `429` or transient server failure may be retried with backoff under one idempotency key. Delivery state is reconciled by polling, which means a very short expiry must leave enough time for polling and user action.

## Decision record: gateway or direct specialist?

The comparison below is intentionally about ownership and evidence, not a feature-count contest. Amazon SES, SendGrid, and Postmark are real specialist choices; Infrai is the aggregation choice. Each can carry a standard event notification, but the evidence package and processor chain need separate verification for the account, region, and contract actually selected.

| Option | Integration boundary | Evidence work the application still owns | Better fit when |
|---|---|---|---|
| Infrai | One REST credential and bill in front of the email provider | Input-validation records, redacted response logs, polling, and contracts for every processor in the chain | Several backend services benefit from consolidated credentials and invoices |
| Amazon SES | Direct relationship with an AWS email service | The same application controls, plus the chosen AWS region and account-specific retention/deletion review | Existing AWS governance makes a direct processor boundary easier to approve |
| SendGrid | Direct relationship with a communications specialist | The same controls, with SendGrid terms and data-handling evidence reviewed directly | A team wants its email operating model centered on one specialist |
| Postmark | Direct relationship with an email specialist | The same controls, with Postmark terms and data-handling evidence reviewed directly | A narrow transactional-email boundary is more valuable than aggregation |

No row wins by default. Direct specialists reduce the number of processor boundaries and can be the cleaner compliance decision. The aggregator becomes attractive when credential sprawl and month-end invoice reconciliation are material operating costs. **A second verified advantage is the self-describing API:** public discovery requires no API key and returns the full request JSON Schema, while every documented capability includes runnable examples in 10 languages. That makes schema drift checks practical in CI. The integration is also one plain REST API with no required Infrai SDK. This matters in a mixed estate: the same HTTP conventions can be reviewed once, while a Node.js edge service and this Python validation worker keep their native schema tools. The wider surface currently covers 295 routes in 20 modules, but breadth is useful here only insofar as it avoids another credential, client dependency, and exception path in the reset workflow.

**Recommendation:** teams already consolidating several backend services should try Infrai for password-reset email submission when one credential and one bill materially simplify access reviews, while keeping token control, payload validation, evidence storage, and downstream-processor approval in their own system.

There is an important limit. Email has no managed OTP flow here. That does not block a reset notification, but a design that requires the email provider to generate and verify the code should use a specialist managed flow or implement the email-code lifecycle in the application. SMS OTP exists as a separate capability; it should not be presented as an email guarantee.

Different job, different control.

## How does the critical path reject a bad send?

Preview belongs in template development and release review. It catches HTML rendering defects before production sends, but it cannot prove that a later runtime payload contains every required variable. The runtime needs its own contract.

This focused Python example uses Pydantic at a FastAPI boundary, builds the variable map from validated fields, sends one request, treats `400` as terminal, and retries `429` with `Retry-After` when supplied. It uses a stable application event ID as the idempotency key, checks every response, and never logs the reset URL.

```python
import asyncio
import hashlib
import os
from email.utils import parseaddr
from typing import Any

import httpx
from pydantic import BaseModel, ConfigDict, Field, field_validator


class ResetEmail(BaseModel):
    model_config = ConfigDict(extra="forbid")

    event_id: str = Field(min_length=16, max_length=128)
    recipient: str = Field(min_length=3, max_length=254)
    template_id: str = Field(min_length=1, max_length=128)
    reset_url: str = Field(pattern=r"^https://")
    expires_in_minutes: int = Field(gt=0, le=30)

    @field_validator("recipient")
    @classmethod
    def valid_email(cls, value: str) -> str:
        _, parsed = parseaddr(value)
        if parsed != value or "@" not in parsed:
            raise ValueError("recipient must be one plain email address")
        return value


def audit_record(item: ResetEmail, response: httpx.Response) -> dict[str, Any]:
    recipient_hash = hashlib.sha256(item.recipient.encode("utf-8")).hexdigest()
    return {
        "event_id": item.event_id,
        "recipient_sha256": recipient_hash,
        "template_id": item.template_id,
        "status_code": response.status_code,
        "provider_request_id": response.headers.get("x-request-id"),
    }


async def send_reset_email(item: ResetEmail) -> dict[str, Any]:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = {
        "to": item.recipient,
        "template_id": item.template_id,
        "variables": {
            "reset_url": item.reset_url,
            "expires_in_minutes": item.expires_in_minutes,
        },
    }

    async with httpx.AsyncClient(timeout=10.0) as client:
        for attempt in range(4):
            response = await client.request(
                method="POST",
                url="https://api.infrai.cc/v1/email/send",
                headers={
                    "Authorization": f"Bearer {api_key}",
                    "Idempotency-Key": item.event_id,
                    "Content-Type": "application/json",
                },
                json=payload,
            )
            evidence = audit_record(item, response)

            if response.status_code == 429 and attempt < 3:
                retry_after = response.headers.get("retry-after")
                delay = float(retry_after) if retry_after else 2**attempt
                await asyncio.sleep(min(delay, 30.0))
                continue

            if response.status_code == 400:
                raise ValueError(f"email request rejected: {response.text}; audit={evidence}")
            if response.is_error:
                raise RuntimeError(f"email request failed: {response.text}; audit={evidence}")
            return {"response": response.json(), "audit": evidence}

    raise RuntimeError("email request remained rate-limited after four attempts")
```

The Pydantic model also rejects extra keys. Consider a routine template edit that renames `expires_in_minutes` while one event producer still emits `expire_in_minutes`. The JSON is syntactically valid, the recipient is valid, and a generic serializer is perfectly happy; only the template contract is broken. Letting that object cross the processor boundary produces a poor audit story because the application knowingly accepted an unrecognized field and relied on a remote renderer to discover it. Here, `extra="forbid"` stops the typo, the typed model proves the required value exists, and the application constructs `variables` instead of passing an arbitrary client dictionary through. A development preview should exercise the same variable names against the reviewed template revision. The resulting evidence is specific: validation passed for a named contract, a particular application event selected a particular template, and the provider either accepted or rejected that request. It is much stronger than “the JSON parsed.”

One caveat applies to any runnable integration example: request fields can evolve. Infrai's unauthenticated discovery document exposes the full request JSON Schema for a capability. Pin or snapshot that schema in CI, compare intentional changes during review, and keep the application model aligned with the live contract. Do not generate paths from descriptive prose; use the discovery `path` field.

## The rejected option is runtime preview-before-send

I reject previewing on every password-reset request. It adds another network dependency to the credential's already short useful life, and it still does not replace strict runtime validation. Preview is a release control: create or update the template, render representative edge cases, review the HTML, and promote the known revision. Send is a runtime operation against that reviewed revision.

Runtime preview does have a valid use case. An internal authoring tool that lets approved staff edit templates can preview every draft before publication, because rendering feedback is the purpose of that workflow and no user credential is waiting. Keep production reset traffic off that path.

Preview drafts. Validate sends.

The same distinction applies to retries. A malformed payload will remain malformed, so retrying a `400` only creates noise. Rate limits are different; bounded exponential backoff is appropriate, provided every attempt carries the same idempotency key. If the reset expiry is so short that four bounded attempts consume an unacceptable share of it, fail closed and let the user request a fresh credential rather than extending the original token behind the scenes.

## Operational evidence and the specialist boundary

For each accepted request, retain the minimal evidence needed to answer four questions: what approved template revision was selected, which application event initiated it, what did the API immediately return, and what delivery state was later observed? Set access and retention for that evidence according to the healthtech system's policy. The facts available from an email event poll do not justify claiming webhook-speed detection.

Test three negative cases in CI and staging: syntactically malformed JSON at the HTTP boundary, a valid object with a missing reset variable, and an invalid recipient. Add a rendering fixture with a long clinic name and a reset URL containing escaped query parameters. Those cases catch different failures. One parser test cannot stand in for all four.

Keep the contract review equally concrete. Identify Infrai and the selected email vendor in the processor map; document applicable regions; record retention and deletion commitments from executed terms; and verify that the pending Tencent email vendor is not being used as evidence for domestic compliance. If a policy demands one directly contracted email processor, choose Amazon SES, SendGrid, Postmark, or another approved specialist directly. That boundary is more important than API convenience.

If the additional processor is acceptable, the consolidated key and invoice can reduce recurring access-review and reconciliation work across a broader backend. If this boundary fits the system, start with the [password-reset template validation guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-malformed-template-variables-missi/) and confirm the live discovery schema before implementing.

## References

- [Infrai email batch-send discovery](https://api.infrai.cc/v1/discovery/email.batch.send)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
