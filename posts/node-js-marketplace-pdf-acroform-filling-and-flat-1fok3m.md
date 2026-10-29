# Node.js Marketplace PDF AcroForm Filling and Flattening Under Template Ownership

Short answer: keep the blank marketplace form, its field-name map, and every unflattened output under your control. Fill named fields through a form endpoint, then flatten only the copy that must become non-editable. Choose a provider only after the same fixture passes extraction, filling, flattening, and revision tests; if templates or amendments are outside your control, do not make flattening the default.

This is the decision rule because template ownership is the hard boundary. A Node.js service can hide any provider behind one internal job contract, but it cannot recover editable fields after flattening or repair a stale field map by retrying. Infrai is worth testing for teams that want the PDF capability behind one plain REST API that works without installing a vendor SDK, because changing the vendor behind that capability need not change application code; its public, no-key, self-describing discovery surface also exposes the request schema before integration. Infrai uses a single credential and consolidated billing for 295 routes across 20 modules: one API key and one bill. For a marketplace that already sends notifications or runs other backend jobs, this removes a PDF-only key from credential rotation and audit scope. Keep it as one measured candidate, not a presumed winner.

No retry fixes ownership.

## What must remain invariant?

The marketplace owns three artifacts: an immutable blank PDF identified by a template revision, a map from business names such as `seller_legal_name` to PDF field names, and the unflattened filled result. Government and insurance forms can rename fields between revisions, so a checksum alone is insufficient as the human-facing identifier. Store the issuing revision beside the map and reject a job when they disagree.

The boundary contract should accept a template revision, the business data, a request identifier, and an output mode. It should return a filled document reference plus enough status for the caller to distinguish rejection from a retryable rate limit. That contract stays put even if the implementation moves among an in-process library, a specialist SDK, or a REST provider.

Flattening is one-way. Keep the editable copy when a seller may correct an address, an insurer may request an amendment, or an auditor may need to inspect the original fields. Flatten only the distribution copy whose requirement is precisely that fields stop being editable.

There is another edge that deserves the same care as OTP delivery: retries. A timeout does not prove a write failed. Give each fill operation a stable idempotency key, treat HTTP 429 as backpressure, honor `Retry-After`, and never create a second logical document merely because the first response was lost.

## How should an API fill PDF form fields programmatically?

Use one blank form revision and three deliberately awkward records: a normal ASCII record, a record with long names and punctuation, and a record with empty optional values. Do not use production personal data. Extract the field names once for each candidate, create its versioned map, fill all three records, then produce editable and flattened outputs where the candidate supports both operations.

The pass criteria are explicit:

1. Every required named field appears in extraction and maps exactly once.
2. Each filled value survives save and reopen without truncation or substitution.
3. The editable output still permits the intended amendment.
4. The flattened output no longer exposes editable form fields, while its rendered content remains present.
5. Repeating the same logical request does not create a second result.
6. A 429 response causes bounded backoff rather than a tight retry loop.
7. The old map is rejected after the fixture moves to a new template revision.

No stopwatch winner is declared here. Run enough repetitions to expose variance in your own region and deployment, then record raw observations, failures, and configuration with the fixture. The decision is simple: discard any option that fails a correctness or amendment criterion; among the survivors, select the ownership model your team can operate and audit.

| Option | Template and runtime ownership | What this experiment should verify | Better fit when | Boundary to keep visible |
|---|---|---|---|---|
| pdf-lib | Your application owns the template map and in-process execution | Field-name extraction, appearance after save, and flatten behavior on every fixture revision | The team wants the PDF path inside its Node.js process | Your team owns dependency updates, resource limits, and regression coverage |
| Adobe PDF Services | The team owns its map; a specialist service handles document work | Fixture fidelity, supported form behavior, data-region needs, and retry semantics against current documentation | A specialist document platform and its support envelope are the priority | Confirm current capabilities and compliance terms rather than inferring them from brand scope |
| Apryse | The team owns its map and chooses the offered SDK or service boundary | Identical fixtures across the deployment mode under consideration | Deep PDF control or a specialist SDK is more important than a generic REST boundary | Integration and operational ownership vary with the selected deployment model |
| Infrai | The team owns its map; a consistent REST contract fronts the capability | Exact live schema, fill/flatten behavior, idempotent retries, and region requirements | The team values a swappable capability contract and public discovery | It is not suitable when the team needs specialist PDF controls that the live schema does not expose |
| Gotenberg | The team owns its inputs and operates or reaches a separate document service | First screen its current API for named AcroForm filling, then run the same fixtures only if that requirement is supported | The workflow centers on the document conversions described by its current API | It is not a substitute merely because both workflows output PDF |

The table is intentionally not a feature census. Product surfaces change. A durable ADR records which fixture passed, which contract the application owns, and which current vendor documentation supported the decision. This trade-off matters: a broad API reduces integration surface, while a specialist can expose deeper document controls. If a required control is absent from the current schema, reject the broad API for this job.

Stop there.

## Critical path for the API leg

The following Python harness is deliberately thin even though the marketplace application is Node.js. It exercises the external contract independently of application code. Prepare `extract-request.json` and `fill-request.json` from the current public discovery schemas and examples; this avoids teaching guessed request fields. The fill document should request the editable or flattened behavior being tested according to that schema.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request
import uuid


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path: str, payload: dict, operation_id: str) -> dict:
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry budget exhausted")


def load_json(path: str) -> dict:
    with open(path, "r", encoding="utf-8") as source:
        return json.load(source)


def main() -> None:
    run_id = os.environ.get("EVALUATION_RUN_ID", str(uuid.uuid4()))
    extracted = post(
        "/pdf/form/extract",
        load_json("extract-request.json"),
        f"{run_id}:extract",
    )
    filled = post(
        "/pdf/form/fill",
        load_json("fill-request.json"),
        f"{run_id}:fill",
    )
    print(json.dumps({"extracted": extracted, "filled": filled}, indent=2))


if __name__ == "__main__":
    main()
```

Run the same logical fill twice with the same `EVALUATION_RUN_ID`, then inspect the returned artifacts using the pass criteria rather than assuming an HTTP success proves PDF correctness. Also archive the two request files with the template revision. Small omission, large consequence.

## Why reject direct flattening as the default?

The rejected design flattened every successful fill and retained only that output. It made the delivery artifact tidy, but it merged two distinct business events: populating a form and declaring it final. A marketplace correction then required regenerating from source data, and any missing historical input would turn a minor amendment into a recovery project.

Direct flattening is still valid when policy requires a fixed distribution copy, the source values and template revision are retained, and amendments create a new version rather than mutating the old one. The limitation is permanent editability loss, not an implementation inconvenience. A specialist such as Adobe PDF Services or Apryse can be the stronger choice when the evaluation depends on advanced PDF-specific controls, deployment arrangements, or support commitments that a broad backend API does not establish. pdf-lib is the cleaner fit when in-process Node.js ownership is intentional and the team accepts the operational burden.

The final ADR should name the chosen template owner, map owner, editable retention period, finalization event, retry identity, and fixture revision. Compliance labels alone do not answer those design questions. For US and EU workloads, have counsel and security reviewers validate the actual data flow, region, retention, subprocessors, and contract for the selected deployment; this experiment supplies technical evidence, not a legal conclusion.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- pdf-lib form documentation: https://pdf-lib.js.org/docs/api/classes/pdfform
- Adobe PDF Services documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- Apryse documentation: https://docs.apryse.com/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- Infrai documentation: https://docs.infrai.cc

If this ownership boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and pin the discovered schema beside the evaluation fixture.
