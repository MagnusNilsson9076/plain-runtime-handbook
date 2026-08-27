# Bounce-Safe B2B SaaS Welcome Email: DKIM, Custom Domains, and Suppression

Short answer: gate every B2B SaaS welcome email on a verified custom sending domain and a current suppression decision, then treat acceptance, bounce, and suppression as separate states. DKIM belongs in that gate, but authentication alone cannot prevent a known-invalid recipient from being queued again.

The hard constraint is timing. A new account needs its welcome message quickly, while a bounce may arrive after the send request has finished. If signup code interprets "queued" as "delivered," it has already lost the information needed to make a reliable retry decision. The safer design gives account creation and mail delivery different state machines, joined by a durable message identifier.

No guesswork.

## A B2B signup starts two clocks

At signup time, start with an admission rule, not a send call. A tenant may request a branded From address, but the mail path should admit that identity only after the corresponding sending domain is in the application's verified state. Keep that state explicit: domain name, verification status, active DKIM selector, last operator check, and the tenant allowed to use it. Do not infer authorization from a matching string in the From header.

SPF and DKIM solve different authentication jobs. RFC 7208 defines SPF as an authorization check for the identity used in SMTP mail handling. DKIM signs selected message content and headers under a domain-controlled key. For this system, the useful architectural point is that authentication records belong to domain configuration, while suppression belongs to recipient history. Passing one check never bypasses the other.

The pre-send order should be stable:

1. Resolve the tenant's approved sender identity.
2. Require a verified sending domain and an active DKIM configuration.
3. Normalize the recipient for lookup without inventing provider-specific mailbox rules.
4. Check the global and tenant-scoped suppression records.
5. Create one durable welcome-message operation and enqueue it once.

That fourth step must happen immediately before the durable enqueue decision, in the same transaction when the storage model permits it. A suppression imported five minutes ago is useless if a stale signup worker can still enqueue from yesterday's cached eligibility result. Cache domain configuration if necessary; don't cache a recipient's permission to receive mail across a send boundary.

The suppression record also needs a reason. At minimum, distinguish an invalid-recipient bounce from an operator block or a complaint-derived block, because removal policy and audit requirements differ. An operator should be able to answer who or what created the record, when it happened, which message led to it, and why a later removal was allowed. "Suppressed: true" isn't enough evidence for a compliance review or a delivery incident.

## How should a SaaS welcome email connect DKIM, a custom sending domain, and suppression?

Once the operation leaves the outbox, an accepted submission is not proof of inbox delivery. Represent the progression as `eligible -> queued -> accepted -> observed`, with `suppressed` as a terminal sending decision until an authorized process changes it. A negative observation can move the recipient into suppression, but it should not rewrite the historical message as though it had never been accepted. Those are separate facts.

This separation matters most during retries. Imagine account `acct_2048` queues welcome operation `welcome:acct_2048:v1`. The worker submits it and loses its lease before storing the acceptance observation. A replacement worker sees incomplete local state. If it creates a fresh operation, the recipient can receive two welcomes; if it marks the old operation delivered, the application invents an outcome. The operation identifier therefore survives worker restarts, and the transport adapter must expose enough submission identity for reconciliation. The exact reconciliation mechanism varies by transport, and I'm not sure one policy fits every provider contract. Document the contract you actually have.

Bounce ingestion needs the same idempotency. Store the raw event identifier or a stable digest, apply each event once, and preserve the original event time separately from ingestion time. Events can be delayed or repeated. If two observations disagree, retain both and derive current state with a documented precedence rule instead of overwriting the inconvenient one.

Be conservative with invalid recipients. A permanent invalid-recipient result should prevent another automated welcome attempt. A temporary delivery delay is different: bound the retry schedule, preserve the same operation identity, and stop after the application's documented limit. The classification vocabulary should be owned by the application even if an adapter translates transport-specific labels into it.

Observability follows the state model. Track queue age, time from queue to acceptance, time from acceptance to observation, bounce classifications, suppression hits, and repeated-event counts. Segment by sending domain and deployment cohort, but protect recipient addresses in logs. A rising suppression-hit count can mean the gate is working; it is not automatically evidence of a new delivery failure. Pair the count with newly created suppressions and bounce classifications before drawing a conclusion.

Short paths hide bugs.

The state model becomes useful only when admission is atomic at the outbox edge. The following Python example is intentionally transport-agnostic. It demonstrates the invariant that a verified domain check, suppression lookup, and idempotent outbox insertion happen as one local decision. It does not claim delivery; it only records that this welcome operation is eligible to be processed.

```python
import hashlib
import sqlite3
from dataclasses import dataclass


@dataclass(frozen=True)
class WelcomeRequest:
    account_id: str
    tenant_id: str
    sending_domain: str
    recipient: str


def normalize_for_lookup(address: str) -> str:
    # Domain names are case-insensitive; preserve the local part as supplied.
    local, separator, domain = address.strip().rpartition("@")
    if not separator or not local or not domain:
        raise ValueError("recipient must contain a local part and domain")
    return f"{local}@{domain.lower()}"


def admit_welcome(db: sqlite3.Connection, request: WelcomeRequest) -> bool:
    recipient = normalize_for_lookup(request.recipient)
    operation_key = hashlib.sha256(
        f"welcome:v1:{request.account_id}".encode("utf-8")
    ).hexdigest()

    with db:
        domain = db.execute(
            """
            SELECT 1
            FROM sending_domains
            WHERE tenant_id = ? AND domain = ? AND verified = 1
            """,
            (request.tenant_id, request.sending_domain.lower()),
        ).fetchone()
        if domain is None:
            return False

        blocked = db.execute(
            """
            SELECT 1
            FROM suppressions
            WHERE recipient = ?
              AND (tenant_id = ? OR tenant_id IS NULL)
            """,
            (recipient, request.tenant_id),
        ).fetchone()
        if blocked is not None:
            return False

        inserted = db.execute(
            """
            INSERT OR IGNORE INTO mail_outbox(
                operation_key, tenant_id, recipient, sending_domain, state
            ) VALUES (?, ?, ?, ?, 'queued')
            """,
            (
                operation_key,
                request.tenant_id,
                recipient,
                request.sending_domain.lower(),
            ),
        )
        return inserted.rowcount == 1
```

There is a deliberate limitation here: address normalization lowercases only the domain. Some systems choose broader canonicalization, but provider-specific transformations such as removing dots or tags can merge distinct recipients. Make that a product policy backed by evidence, not a convenience hidden in a helper.

The schema should enforce the invariant with a unique `operation_key`; code-level checking alone leaves a race between workers. Suppression creation deserves a similar unique event key so a replay cannot create conflicting audit rows. Keep the transport call outside this transaction: the outbox worker claims committed work, submits it, and records the resulting transport identity. Holding a database transaction open across a network call would couple database contention to transport latency.

## Operational ownership carries the hidden cost

By the time delayed bounce evidence arrives, the system needs a clear owner for reconciliation. That work is the hidden cost in this design, so choose ownership and required event behavior before interface style. A managed email transport is suitable when the team wants to delegate SMTP delivery while retaining sender-domain configuration, suppression policy, event ingestion, and audit state in the application. A self-operated mail transfer stack offers deeper control, but it also leaves DNS operations, queue operations, reputation monitoring, abuse response, and delivery troubleshooting with the team. The latter is not suitable when nobody owns those duties on call.

| Decision area | Prefer application-owned control when | Prefer transport-managed behavior when | The catch |
|---|---|---|---|
| Suppression | Tenant policy and audit history must remain portable | A single mail path owns all recipient history | Transport state can drift from application state unless it is reconciled |
| Event intake | The system needs a canonical bounce vocabulary | The team can consume one transport's native event model | Adapters still need replay and deduplication rules |
| Domain configuration | Many tenants bring branded identities | One company domain covers all mail | Tenant authorization must be enforced separately from DNS verification |
| Operations | The team already runs delivery infrastructure | The team wants a narrower operational surface | Responsibility for consent, content, and recipient policy never disappears |

Stick with a dedicated email-only boundary when specialized deliverability controls and direct access to detailed mail events are the dominant requirements. A broader communications abstraction can be useful for a product that coordinates email and SMS, but it is not suitable if it erases channel-specific bounce classes, consent state, or authentication evidence. The abstraction should unify operation identity and audit conventions, not flatten meaningful differences.

SMS fallback deserves an explicit warning. Email suppression does not grant permission to text a phone number, and an email bounce is not evidence of SMS consent. CTIA's messaging commitments emphasize consumer protection and responsible messaging practices; the application should keep channel-specific consent and suppression records even when one workflow can select several channels.

## Migrate with shadow decisions and two cohorts

Begin with shadow evaluation: run the domain and suppression gates against real queued candidates without changing delivery, then compare the decisions with the current path. Next, enable the new outbox for one internal domain and a small tenant cohort. Seed test recipients that exercise verified-domain rejection, global suppression, tenant suppression, duplicate enqueue, delayed observation, and repeated bounce ingestion.

The release gate is compact: domain ownership is explicit, DKIM configuration is operator-visible, suppression is checked at admission, one operation survives retries, bounce events are idempotent, and unknown delivery remains unknown. Roll back by cohort while retaining event ingestion, because observations arriving after a traffic change still belong in the audit trail.

Only widen the cohort after queue age and observation lag are stable and operators can explain every state transition from stored evidence. A polished template cannot compensate for a system that repeatedly sends to an address it already knows is invalid.

## References

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
