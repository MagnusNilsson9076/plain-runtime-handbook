# Cheap LLM API Gateway Cost: 5 One-Key Quality Gates for Invoice Extraction

A cheap LLM API gateway with one key should route supplier invoices through a policy layer that can reject, retry, or escalate an extraction before optimizing for the fastest response. The deciding constraint is not the advertised token price; it is whether the system can return five required fields with traceable evidence inside the publishing workflow's latency budget.

TL;DR: keep one application credential at the gateway boundary, but do not confuse one key with interchangeable behavior. Normalize requests and accounting there. Preserve model-specific capabilities behind adapters, estimate tokens before admission, cache only content-addressed and policy-compatible results, and batch only work that has no interactive deadline. For media invoice intake, quality wins until an extraction passes schema, arithmetic, evidence, and residency checks. Latency decides among the candidates that remain.

This architecture decision record covers one job: extracting invoice number, invoice date, currency, subtotal, and total from supplier invoices. A cheap gateway is acceptable only if it makes routing decisions observable and reversible.

## Should a cheap LLM API gateway use one key for invoices?

Five invariants define the system more usefully than a feature checklist. Every accepted result conforms to a versioned schema. Every extracted value carries page-level evidence or an explicit `null`; plausible invention is a failure. Subtotal and total are parsed as decimal values in the stated currency, never binary floating point. Retries are bounded and idempotent. Finally, the execution region and retention policy are recorded with the result.

These invariants separate correctness from convenience. A unified request shape can reduce application branching, yet a gateway cannot make distinct tokenizers, context rules, caching semantics, or batch contracts identical. The adapter owns those differences. The domain service owns acceptance.

One key is not one policy.

Invoices arrive from email and shared upload channels, so duplicate delivery belongs in the design. Derive an idempotency key from the document digest, extraction-policy version, and model-route version. Do not derive it from a filename: `invoice.pdf` is not an identity. Store the original object's immutable digest beside the extraction so a later retry cannot silently operate on changed bytes.

A token estimate is an admission-control signal, not a bill. Models may tokenize the same content differently, and image or document inputs may be metered under model-specific rules. Record the estimate, the adapter that produced it, actual usage returned by execution, and the difference. Missing actual usage should remain missing rather than becoming an invented zero.

No estimate fixes weak evidence.

Residency is another hard boundary. A label such as `eu` is insufficient evidence by itself. The route record needs the processing region selected by policy, destination adapter, retention mode, policy version, and timestamp. Transfers of personal data from the European Economic Area require an appropriate legal basis and safeguards; routing metadata supports an audit, but does not create those safeguards [1].

## Which path should handle each invoice?

Classify work before selecting a runtime path. An editor waiting to approve tomorrow's issue has an interactive deadline. A nightly supplier archive does not. Both need the same acceptance gates, but only the second can trade queue delay for batch efficiency.

| Path | Use when | Quality control | Latency consequence | Accounting boundary |
| --- | --- | --- | --- | --- |
| Synchronous | A human is waiting and the document fits declared limits | Validate, then retry or escalate on a typed failure | Queue and inference time are on the user path | Reserve estimated usage; reconcile actual usage |
| Deferred batch | The deadline is measured in hours | Run the identical validator when results arrive | Completion is intentionally asynchronous | Attribute usage per document, not only per batch |
| Verified cache | Digest, policy, route, and residency scope match | Reuse only a previously accepted result | Avoids inference while retaining provenance | Count a cache hit separately from model usage |
| Manual review | Evidence is absent, arithmetic conflicts, or retries end | A reviewer resolves or rejects fields | Slow by design | Attribute work to the failed extraction class |

Never race several models by default merely to take the first valid-looking JSON. That multiplies processing, complicates deletion and residency obligations, and rewards speed before evidence has been checked. Hedging can be justified for a narrow interactive tail-latency target, but it needs an explicit cancellation policy and separate accounting.

The trade-off is explicit.

Cache keys deserve the same suspicion as OTP retry keys. A digest of document bytes alone is too broad. If the schema changes from `invoice_date` as free text to an ISO date, an old response is no longer equivalent. If residency scope changes, reuse may violate current policy even when the fields look identical. Cache accepted results only, encrypt them under the source's data classification, and apply the source document's deletion lifecycle.

## Critical path and failure boundaries

The critical path below is intentionally small. The runtime client is a generic interface; policy remains visible in application code. The example assumes no commercial endpoint, model identifier, or proprietary response shape.

```python
from dataclasses import dataclass
from datetime import date
from decimal import Decimal, InvalidOperation
from hashlib import sha256
from typing import Protocol


FIELDS = ("invoice_number", "invoice_date", "currency", "subtotal", "total")


@dataclass(frozen=True)
class Policy:
    version: str
    route_version: str
    residency: str
    max_attempts: int = 2


@dataclass(frozen=True)
class Result:
    values: dict[str, str | None]
    evidence_pages: dict[str, int | None]
    actual_input_tokens: int | None
    actual_output_tokens: int | None
    processing_region: str


class Runtime(Protocol):
    def estimate_tokens(self, document: bytes) -> int: ...

    def extract(self, document: bytes, policy: Policy, key: str) -> Result: ...


def request_key(document: bytes, policy: Policy) -> str:
    identity = b"|".join((
        sha256(document).hexdigest().encode(),
        policy.version.encode(),
        policy.route_version.encode(),
        policy.residency.encode(),
    ))
    return sha256(identity).hexdigest()


def validate(result: Result, policy: Policy) -> None:
    if result.processing_region != policy.residency:
        raise ValueError("processing region did not match policy")
    missing = [name for name in FIELDS if not result.values.get(name)]
    if missing:
        raise ValueError(f"missing required fields: {missing}")
    no_evidence = [name for name in FIELDS if result.evidence_pages.get(name) is None]
    if no_evidence:
        raise ValueError(f"fields without page evidence: {no_evidence}")
    try:
        date.fromisoformat(result.values["invoice_date"] or "")
        subtotal = Decimal(result.values["subtotal"] or "")
        total = Decimal(result.values["total"] or "")
    except (ValueError, InvalidOperation) as error:
        raise ValueError("invalid date or amount representation") from error
    if subtotal < 0 or total < subtotal:
        raise ValueError("invoice arithmetic requires review")


def extract_invoice(document: bytes, runtime: Runtime, policy: Policy):
    estimate = runtime.estimate_tokens(document)
    key = request_key(document, policy)
    last_error = None
    for _ in range(policy.max_attempts):
        result = runtime.extract(document, policy, key)
        try:
            validate(result, policy)
            return result, estimate
        except ValueError as error:
            last_error = error
    raise ValueError("manual review required") from last_error
```

Two details are easy to miss. `Decimal` protects money comparisons from binary floating-point representation, while the bounded loop prevents a malformed document from becoming an unbounded spend loop. The code does not assert that `total == subtotal`; tax, shipping, credits, and rounding can legitimately separate them. It rejects only an obviously inconsistent relationship for this simplified schema and sends richer arithmetic cases to review.

Production code should classify errors. Transport timeout, rate limit, malformed output, policy mismatch, and evidence failure have different retry behavior. Retry a transient transport failure with bounded backoff. Do not retry a residency mismatch on the same route. A schema failure may justify one repair attempt, but repeated repair should end in review, not a recursive prompt chain.

Streaming rarely improves this job. Server-sent events use a persistent HTTP connection in the `text/event-stream` format and are one-way from server to client [2]. They can expose progress to an interactive UI, but partial invoice JSON must not be committed as a valid extraction. Buffer it, validate the completed object, then publish one domain event.

Fast fragments are still fragments.

## Measure decisions, not a blended average

A single mean latency and one cost total cannot explain whether the runtime is working. Segment observations by synchronous, batch, cache, retry, and manual-review outcomes. For each attempt, retain document digest, policy version, route version, region, estimated usage, actual usage when reported, validation result, retry count, queue duration, execution duration, and final disposition. Keep raw invoice content out of general-purpose logs [3].

Useful quality measures follow the workflow: required-field acceptance rate, evidence coverage, arithmetic-review rate, human correction rate by field, and duplicate-suppression rate. Latency needs percentiles for accepted synchronous requests and age-to-completion for deferred work. Cost accounting should include execution, retries, duplicate work, storage, review labor, and gateway operation without pretending they share one universal unit price.

Set thresholds from the publishing deadline and review capacity. A team may define a five-minute interactive objective and a two-attempt ceiling as policy inputs; those are design examples, not industry benchmarks. Replay a labeled, access-controlled invoice set whenever the adapter, prompt, schema, or route changes. Compare field corrections and tail latency, then approve the change through the same versioned policy used in production.

The one-key requirement belongs here too. Applications should receive a scoped internal credential, while the gateway keeps upstream credentials isolated, rotated, and unavailable to callers. Scope that credential by environment and tenant, rate-limit it, and make revocation independent. One shared secret across development, batch workers, and production turns convenience into a broad incident boundary.

## Why reject direct application integrations?

The rejected option lets each application call multiple model APIs directly and implement token estimation, retries, caching, batch polling, and regional rules itself. It can produce the lowest initial latency because there is no gateway hop, and it preserves immediate access to provider-specific features. That is valid for a short evaluation, one route, or a tightly owned service whose policy surface is genuinely small.

The gateway approach has real limitations. It is not suitable when an experiment depends on a newly released, provider-specific capability that the adapter cannot represent, or when one small service has no shared policy to centralize. In those cases, use a direct integration behind the same domain-level validator, keep its credential scoped to that service, and record usage and residency with the extraction. The direct route should remain an explicit exception rather than quietly changing the default path.

It loses for this invoice workflow because acceptance and compliance rules would be copied into interactive intake, nightly archives, and reprocessing jobs. Credential rotation becomes an application concern. Usage reconciliation acquires incompatible paths. Most dangerously, a developer can add a faster route without adding its evidence, deletion, and residency behavior to the decision record.

The accepted design adds an operational component and a small network hop. Treat that honestly: deploy it in required processing regions, bound its queues, export route-level telemetry, and test fail-closed policy behavior. If the gateway cannot prove where a document may go, it should not send the document. **Quality and policy eligibility choose the candidate set; latency chooses within it.**

## References

1. European Commission, "What rules apply if my organisation transfers data outside the EU?" https://commission.europa.eu/law/law-topic/data-protection/rules-business-and-organisations/obligations/what-rules-apply-if-my-organisation-transfers-data-outside-eu_en
2. MDN Web Docs, "Using server-sent events." https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
3. OWASP Cheat Sheet Series, "Logging Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
4. Python documentation, "decimal — Decimal fixed-point and floating-point arithmetic." https://docs.python.org/3/library/decimal.html
