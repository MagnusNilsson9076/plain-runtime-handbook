# Logistics Email Authentication: Retaining SPF, DKIM, and DMARC Records as Release Evidence

The operating bill for logistics email authentication includes publishing SPF, DKIM, and DMARC records, verifying domains, and retaining evidence of release decisions. In a proposed workload of 100 domains, checking three authentication record groups produces 300 observations per sweep. Keep 20 full sweeps and that becomes 6,000 observations. Those are planning assumptions, not a vendor benchmark; they identify repeated evidence as a term the internal admin console can control.

TL;DR: Publish SPF, DKIM, and DMARC as TXT records, then verify the sending domain through the mail service. Start DMARC in monitoring mode. For shipment notifications and delivery OTPs, make that verification result part of the release decision, and retain selected evidence rather than every identical observation. SPF alone is insufficient evidence of a usable sending domain.

One successful DNS write is a small receipt. It cannot carry the whole deliverability argument.

## What do SPF, DKIM, and DMARC records each cover?

The console needs to distinguish three responsibilities. SPF authorizes senders. DKIM signs messages. DMARC evaluates authentication and alignment with the visible From domain and publishes a policy for handling failures. Calling DMARC merely a tie-breaker between SPF and DKIM hides the alignment condition that matters to receivers. These mechanisms support authentication; they don't promise inbox placement.

For this setup, publish all three as TXT records. There is no separate record type to select for each mechanism. The controls should expose the purpose of the record as well as its DNS type: a logistics operator looking at three entries labeled TXT still needs to know which supports sender authorization, which carries DKIM verification material, and which sets DMARC policy. Do not invent the contents. Use the sending service's required values and the domain's approved sender inventory.

Verify after publishing.

A DNS lookup establishes what the lookup returned. A mail-side domain check establishes what the sending service accepts for that domain. Keep those observations separate in the release record, because substituting one for the other makes a green console badge ambiguous. A domain used for shipment notices and a domain used for delivery OTPs should each have an identifiable verification decision; a shared logistics tenant name is not sufficient evidence.

Begin DMARC with a monitoring policy, commonly expressed as p=none, while reviewing the legitimate sending paths. Moving immediately to rejection can discard transactional mail from paths that have not been accounted for. Monitoring also has a limitation: it requests observation without asking receivers to reject failing messages. Treat it as a rollout stage with an owner, not as proof that enforcement is complete.

## Reduce repeated evidence before changing providers

Return to the proposed 100-domain workload. If the three record groups remain unchanged across 20 sweeps, retaining one baseline per domain requires 300 record observations instead of 6,000. That arithmetic describes an unchanged interval only. Actual changes and new verification outcomes require additional evidence. It says nothing about bytes stored, API charges, or a particular provider's billing unit.

The useful change is to separate checking from keeping. Continue checks according to the release workflow, but keep a baseline, changed configurations, and the mail-side result used to authorize sending. An unchanged observation can update a last-checked timestamp and counter. It need not create another permanent copy of the same TXT material. This reduces duplicate evidence without treating a previous verification as a permanent guarantee.

For a proposed retention policy, keep routine unchanged-check summaries for 7 days and release decisions for 30 days. These are design choices for discussion with operations and compliance, not legal requirements or product limits. A shipment dispute may outlast either interval. If a contractual investigation window is longer, that requirement has to change the policy before deletion begins.

Keep the evidence small.

A release record can identify the domain, configuration fingerprint, observation time, approving actor, and mail verification result. It does not need the recipient address, OTP value, or shipment message body. This boundary matters more than squeezing a few bytes out of a TXT string: copying delivery payloads into a DNS audit trail creates another sensitive dataset without helping explain which authentication configuration was approved.

## Inspect the integration contract before writing records

Before implementing the console's write and verification requests, inspect the public discovery manifest. This Python 3.11 example retrieves that integration contract. Set INFRAI_BASE_URL to the service's documented API base, including its version prefix, and INFRAI_API_KEY to your key. The manifest is publicly readable; the example uses the same environment-based credential pattern as the console's authenticated requests. It makes no DNS changes and does not claim to verify a domain.

```python
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    backoff = float(2 ** attempt)
    if value is None:
        return backoff
    try:
        requested = float(value)
    except ValueError:
        try:
            requested = (
                parsedate_to_datetime(value) - datetime.now(timezone.utc)
            ).total_seconds()
        except (TypeError, ValueError, OverflowError):
            return backoff
    return max(backoff, requested)


def main() -> None:
    base = os.environ["INFRAI_BASE_URL"].rstrip("/")
    key = os.environ["INFRAI_API_KEY"]
    if not base.startswith("https://"):
        raise ValueError("An HTTPS API base is required")
    for attempt in range(5):
        request = Request(
            f"{base}/discovery",
            headers={"Authorization": f"Bearer {key}"},
            method="GET",
        )
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                manifest = json.load(response)
            print(json.dumps(manifest, indent=2))
            return
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))


if __name__ == "__main__":
    main()
```

This is a read-only request, so retrying cannot duplicate a DNS write. The retry loop honors Retry-After and uses exponential backoff; unsuccessful responses surface their status and body. Inspect the declared request schemas before adding writes. A discovery result is an integration artifact, not deliverability evidence.

For retention, compare consecutive configuration fingerprints and mail verification results per domain. Preserve a changed acceptance result even if the TXT values are unchanged, and preserve a return to an earlier configuration. Don't suppress an explicit approval event just because its configuration matches the preceding event. Hashing helps compare configurations; it neither proves who approved them nor anonymizes predictable values. Keep approval events in a durable decision trail.

## Compare ownership of evidence, not just DNS features

Cloudflare DNS, Amazon Route 53, and Amazon SES are real options, but they occupy different parts of this workflow. Cloudflare and Route 53 provide DNS record management. SES provides sending-identity verification on the mail side. A fair selection has to account for that boundary; a DNS API and a sending service are not interchangeable.

| Option | Relevant role | Evidence boundary for this console |
| --- | --- | --- |
| Cloudflare DNS | Manage the authentication TXT records in a Cloudflare-managed zone | Record-management success still needs a separate mail-side verification result. |
| Amazon Route 53 | Manage the authentication TXT records in a Route 53 hosted zone | A DNS change does not establish that the sending service accepts the domain. |
| Amazon SES | Verify a sending identity and configure email authentication | Its mail-side evidence does not replace the DNS management responsibility for the authoritative zone. |

For a team already operating its zones in Cloudflare, keeping DNS changes there can preserve an established ownership boundary. Route 53 serves the corresponding DNS role for teams using its hosted zones. SES is relevant when it is the sending service whose acceptance the release must establish. Choose the authoritative DNS integration and the sending-service integration together; requiring a zone move solely to simplify a console can introduce work unrelated to deliverability evidence.

Infrai combines one key and one bill with a unified REST API: plain HTTP, no SDK required, and language-agnostic requests. For this console, shared credentials and billing reduce dashboard administration and invoice reconciliation. The Python backend can use its standard library for both DNS management and mail-side verification, avoiding separate SDK dependencies for those two steps. Public discovery is self-describing and requires no key; it supplies full request and response schemas, with runnable examples in 10 languages for every documented capability. That lets the implementer inspect the DNS-write and mail-verification contracts before wiring the release gate, rather than infer request fields from prose. The platform spans 295 routes across 20 modules; consolidation is useful when the same console also needs other backend services, while public schemas address a separate problem: implementing the verification workflow with explicit contracts.

No delivery benchmark decides this comparison. The practical acceptance test is a configuration change followed by the actual sending service's domain verification, with enough retained context to explain the result. Teams that already have clear ownership and adequate evidence in separate integrations may gain little from consolidating them.

## Decide what an investigation will lose

Retain the configuration baseline and the verification result associated with each release decision. Record the actor and time under the console's own audit policy. Before tightening DMARC enforcement, review the sending paths that monitoring has exposed, including the transactional paths carrying shipment notifications and delivery OTPs. Authentication coverage is the gate; a tidy DNS editor is only part of the mechanism.

The proposed policy deliberately stops keeping full copies of unchanged checks and avoids storing message payloads in this audit trail. After routine summaries expire, an investigator cannot reconstruct every intermediate lookup or its precise timing. A configuration fingerprint cannot recover the original record values either, so preserve the corresponding baseline if reconstruction matters. Those are concrete losses. Accept them only if release evidence and the separate operational records satisfy the investigation window; otherwise extend retention before shipping the policy.

## References

The authentication specifications and provider documentation establish the semantic and integration boundaries used above.

## Further reading

- SPF specification: https://datatracker.ietf.org/doc/html/rfc7208
- DKIM specification: https://datatracker.ietf.org/doc/html/rfc6376
- DMARC specification: https://datatracker.ietf.org/doc/html/rfc7489
- Cloudflare DNS record management: https://developers.cloudflare.com/dns/manage-dns-records/
- Amazon Route 53 resource record sets: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html
- Amazon SES verified identities: https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html
