# DMARC Policy Monitoring in Python: Report Retention Before Quarantine and Reject

The bill for DMARC policy rollout monitoring has several parts: report ingestion, parsing, retained report bytes, investigation time, and the operational cost of holding a tenant's email cutover open. For retained data, the useful starting equation is storage = reports per day × average stored report size × retention days. If raw reports dominate stored bytes, shortening their retention moves that term directly; changing an enforcement policy doesn't automatically reduce it.

Short answer: roll out the DMARC TXT policy through `p=none`, then `p=quarantine`, then `p=reject`, using aggregate reports to establish which legitimate senders pass aligned authentication before each increase. RFC 7489 defines these policies; the sequence is an operational recommendation, not a mandatory protocol handshake. For an e-commerce system that automatically assigns tenant subdomains, provision the hostname separately from the decision to enforce its mail policy. A successful DNS update is not evidence that order confirmations or OTP messages will pass DMARC.

## What does keeping another report actually buy?

Consider a proposed retention policy of 28 days for raw aggregate reports versus 7 days. At a constant daily byte volume, the raw storage term is four times as large with the longer window. These are illustrative retention choices, not observed traffic or recommended defaults. The dominant term needs measuring: if investigation time outweighs storage, deleting reports earlier can make the overall operating burden worse.

The useful change is to give raw reports and extracted evidence different lifetimes. Retain the report interval, reporting organization, source IP, message count, evaluated policy, authentication results, and relevant identifiers in a compact representation. RFC 7489's aggregate report format supplies these fields. Whether that representation actually occupies fewer bytes depends on encoding and indexing, so measure both before claiming a storage benefit. Keep the raw document briefly enough to control duplication, but long enough to diagnose parser disagreements.

Counts matter.

A report row can represent multiple messages, so counting rows as messages distorts the denominator used to approve enforcement. Report time intervals matter too: a newly ingested document can describe mail sent before the latest TXT change. Store its reporting interval alongside ingestion time and the policy version you published. Otherwise yesterday's evidence can look like today's regression.

There is a privacy trade-off. Source addresses and authentication identifiers are operational evidence that deserve access controls and an explicit retention purpose. Aggregate reports are not a reason to retain message bodies. RFC 7489 also describes failure reporting and its privacy considerations; this design starts with aggregate reporting and deliberately leaves message-level failure collection out of the default path.

## A tenant hostname is not an authentication boundary

Take the illustrative tenant `cedar.shops.example.com`, which sends order confirmations and sign-in codes with that domain in the visible From address. Its web subdomain becoming reachable says nothing about the alignment of its email. Under RFC 7489, DMARC passes when at least one passing SPF or DKIM result is aligned with the RFC5322.From domain. Authentication success by itself is insufficient. That distinction catches a common provisioning mistake: validating a sender's SPF or DKIM result without checking the domain it authenticated. A passing signature for an unrelated domain does not establish DMARC alignment for the tenant. Forwarding is another reason to inspect the mechanisms separately. RFC 7489 discusses how forwarding can disrupt SPF and how message modification can affect DKIM; a policy transition cannot repair either relationship. Check the effective policy before creating the tenant's record. RFC 7489 describes discovery at the From domain and, when needed, its Organizational Domain. An Organizational Domain's `sp` policy can govern subdomains that lack their own DMARC record; absent `sp`, its `p` policy is used. Don't assume that every intermediate DNS label contributes another inherited policy layer. A new tenant can therefore start under an existing enforcement policy before the provisioning workflow has intentionally approved it.

Relaxed alignment is the default. It permits an Organizational Domain match, which means separate tenant subdomain labels do not inherently enforce isolation between their mail identities. Strict alignment can require exact domain matches, but changing alignment and enforcement together makes failures harder to explain. Evaluate that choice against the actual sending domains and signing setup.

Keep ownership explicit.

For each tenant, the operational record should identify who can publish DNS, who controls the sending configuration, and who can authorize an enforcement increase. A system that creates subdomains automatically still needs an accountable decision about mail.

## Should DMARC policy monitoring come before quarantine or reject?

Start with `p=none` and an aggregate reporting destination. This requests monitoring without a DMARC-specific delivery action; receiving systems can still apply their own filtering. Confirm that every known legitimate path appears in the evidence, including low-volume account recovery and order notifications. No fixed number of quiet days proves a sender is safe if the relevant path never ran.

Next request `p=quarantine`. RFC 7489 describes this as asking receivers to treat failing messages as suspicious, with local handling determining the result. It can interrupt an OTP journey even when the message is technically accepted. Watch authentication evidence together with application signals such as accepted sends, delivery notifications where available, and support reports. None of those signals alone proves inbox placement.

Advance to `p=reject` after legitimate failing paths have been repaired or explicitly retired. This requests rejection of messages that fail the policy. Receiver discretion still applies. Reject is a statement about the disposition of authentication failures, not proof that every intended customer received a message.

The following Python example renders complete replacement values for one TXT record at `_dmarc.cedar.shops.example.com`. The reporting mailbox is illustrative and must exist before use. Keeping the destination under the tenant domain avoids introducing an external reporting destination into this example.

```python
POLICIES = ("none", "quarantine", "reject")

def dmarc_value(policy: str) -> str:
    if policy not in POLICIES:
        raise ValueError("Unsupported DMARC policy")
    return (
        f"v=DMARC1; p={policy}; pct=100; "
        "rua=mailto:dmarc-reports@cedar.shops.example.com"
    )

for policy in POLICIES:
    print(dmarc_value(policy))
```

These are three successive values, not three simultaneous DMARC records. RFC 7489 specifies policy discovery behavior when multiple records remain; adding another policy record is not a safe way to stage a transition. Publish one intended value and verify what DNS returns. The example is a renderer, not a DNS publisher or an instruction to advance automatically.

Here `pct=100` makes the requested scope explicit. RFC 7489 also provides percentage-based enforcement, but the percentage applies to messages failing DMARC and does not select particular tenants or guarantee a representative sample. For an automatically provisioned fleet, approving named tenant cohorts gives the operator a clearer release boundary. It still requires deliberate DNS changes for those tenants.

## How should DNS delay affect a tenant's cutover?

Separate the time a policy is published from the time evidence can reasonably describe its effects. DNS caches and asynchronous aggregate reporting prevent a single write acknowledgment from establishing universal adoption. RFC 7489 discusses DNS TTLs and reporting intervals; it does not supply a universal propagation deadline or a guarantee that every receiver sends reports. A missing report is an evidence gap. It is not a pass.

This is where the propagation-versus-cutover decision becomes concrete. For a tenant whose legitimate mail is already aligned and represented in reports, prepare its DNS and reporting path ahead of the application launch so the remaining observation work does not begin at checkout enablement. For an unobserved sender, a fast website launch is no justification for jumping straight to rejection. Keep the mail enforcement decision pending while collecting the missing evidence.

Test the deployment controls as well as the record text: invalid policy input must fail, the published value must match the approved policy, and the operator must be able to identify the prior value. Test the report parser against RFC 7489's schema and preserve message counts. Exercise a rollback decision using known failing authentication evidence before relying on that decision during a customer-facing incident.

A rollback to a less restrictive policy is another DNS publication. Cached values can remain in use, and the change cannot recover a message already rejected. For OTP traffic, that makes a retry path and clear application status part of release planning. Keep retries bounded; replaying every failed notification can create duplicate orders-of-events for customers without repairing the authentication failure.

The final retention decision should be explicit: stop keeping raw report payloads after the chosen investigation window, retain the approved policy history and necessary extracted evidence for their separately defined purposes, and omit message-level failure reports unless there is a justified need. The cost is lost forensic detail. If a parser defect is discovered after raw reports expire, the extracted rows may be impossible to revalidate. Choose that loss deliberately, and do not advance enforcement using evidence whose interpretation is still in doubt.

## References

RFC 7489 supplies the authentication alignment rules, policy tags and discovery procedure, receiver policy handling, aggregate report format, DNS considerations, and reporting privacy discussion used here. Retention windows, tenant cohorts, and approval controls are design choices rather than protocol requirements.

## Further reading

- RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
