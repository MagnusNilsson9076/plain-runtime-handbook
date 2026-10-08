# Scanned Invoice OCR Batching: Controlling Throughput, Storage, and Searchable PDF Output

**TL;DR:** For a media business turning order records and scanned invoice attachments into searchable PDFs, the simplest dependable design is a bounded batch pipeline: preserve the uploaded scan, process pages behind a concurrency limit, validate the text layer, and retain only the artifacts that serve a declared recovery or audit purpose. The engine can be local or managed. Batch throughput depends more on page volume, retry behavior, and oversized jobs than on the language used by the caller.

Start with the bill. Its useful unit is not “one API call” or “one invoice”; it is one page carried through recognition, PDF assembly, validation, storage, and possibly a retry. A planning model can stay deliberately plain:

Pages are the unit.

`daily work = accepted documents × mean pages × (1 + page retry rate)`

For a hypothetical media order run of 40,000 two-page invoices with a 2% page retry rate, that is 81,600 page attempts, not 40,000 requests. This is a capacity example, not a benchmark. Substitute measurements from a representative invoice set before choosing worker counts or service limits.

The change that moves the dominant term is usually reducing repeated page work. Hash the immutable source, make submission idempotent, retry failed pages rather than whole documents when the chosen engine permits it, and place large invoices in a separate queue. Those controls matter whether the recognition engine is an executable, a container, or a remote API.

Retries count.

## What are you actually paying to keep?

An OCR pipeline can create four artifacts: the original scan, rendered page images, extracted text or structured fields, and a searchable PDF containing the scan plus a text layer. Keeping all four indefinitely makes recovery comfortable, but multiplies stored bytes and expands the set of sensitive material that retention and deletion workflows must cover.

Treat each artifact as a separate retention decision. The original is the evidence needed to reproduce recognition. The searchable PDF is the reader-facing derivative. Extracted text supports indexing and downstream checks. Rendered pages are normally intermediate work products; after validation and the recovery window, they are the first candidate for deletion.

There is a cost to that choice. If page images are discarded, a later investigation must render the original again before recognition can be replayed. If the original is discarded too, a future engine change cannot reconstruct information that the previous pass missed. **Keep the immutable source for the period justified by the document policy, but do not confuse temporary renderings with records.** The exact period is a business and compliance decision, not an OCR default.

This distinction also keeps deletions honest. An order erasure workflow must know whether the same invoice text lives in object storage, a search index, a dead-letter payload, and an application log. Logging recognized invoice bodies is especially hard to defend; identifiers, state transitions, durations, counts, and stable error codes are usually enough to operate the pipeline.

Delete deliberately.

## Throughput comes from bounded work, not a bigger request

The public interface should accept a job and return a durable identifier quickly. Processing a 200-page attachment inside one synchronous request couples caller timeouts to recognition time and encourages whole-document retries. A queue separates admission from execution, while a status record makes retries observable instead of mysterious.

A useful state machine is small: `accepted`, `processing`, `validating`, `completed`, and `failed`. State changes should be conditional so two workers cannot both publish the same derivative. Store an idempotency key derived from the order identity, source hash, and processing-policy version. Changing recognition settings then creates a new, explainable job instead of silently overwriting an old result.

Duplicates will happen.

Do not let the mean invoice hide the tail. One very large PDF can occupy a worker long enough to delay thousands of ordinary two-page invoices. Inspect page count before recognition, reject inputs beyond a documented limit, and route accepted large jobs to a queue with its own concurrency budget. This is head-of-line blocking in business clothing.

The following planner exposes the assumptions rather than pretending to predict an engine. It returns the minimum sustained page rate and the worker count implied by a measured per-worker rate. All numbers passed to it must come from load tests or an explicit traffic forecast.

```python
from dataclasses import dataclass
from math import ceil


@dataclass(frozen=True)
class BatchPlan:
    page_attempts: int
    required_pages_per_second: float
    workers: int


def plan_batch(
    documents: int,
    mean_pages: float,
    retry_fraction: float,
    window_seconds: int,
    measured_pages_per_second_per_worker: float,
) -> BatchPlan:
    if documents < 0 or mean_pages <= 0:
        raise ValueError("documents must be nonnegative and mean_pages positive")
    if not 0 <= retry_fraction < 1:
        raise ValueError("retry_fraction must be in [0, 1)")
    if window_seconds <= 0 or measured_pages_per_second_per_worker <= 0:
        raise ValueError("capacity inputs must be positive")

    attempts = ceil(documents * mean_pages * (1 + retry_fraction))
    required_rate = attempts / window_seconds
    workers = ceil(required_rate / measured_pages_per_second_per_worker)
    return BatchPlan(attempts, required_rate, workers)
```

That last input is the trap. Measure it with the page sizes, scan quality, languages, fonts, and concurrency expected in production. A single clean sample invoice says almost nothing about a mixed overnight batch. Keep a fixed evaluation corpus, but do not put real customer invoices in a source repository.

Backpressure belongs at admission as well as at workers. Set a maximum queued page estimate and return a stable “capacity unavailable” result when that budget is exhausted. Unlimited acceptance merely moves an outage into tomorrow's backlog.

The queue is finite.

## How should an OCR API turn a scanned PDF into searchable text?

PDF is a document format with a formal specification, ISO 32000-2. An OCR engine's plain-text response is therefore not itself a searchable PDF. The pipeline must assemble or update a PDF so recognized text is associated with the visible page, then verify the resulting artifact rather than trusting a success flag.

Validation needs layers. First, parse the output as a PDF and confirm that its page count matches the accepted source. Next, extract text from each page and flag empty or implausibly sparse results. Then render selected pages and compare the visual dimensions and orientation with the source. For invoice workflows, apply domain checks to expected fields such as order identifier, invoice identifier, dates, totals, and currency, but do not treat a plausible total as proof that every line was read correctly.

Text geometry matters. A page can contain extractable words yet produce a painful search and copy experience because coordinates, rotation, or reading order are wrong. Sample those behaviors in acceptance tests. Keep the original pixels visible; recognition should add a text layer, not redraw the commercial record from guessed characters.

Confidence scores, when available, are routing signals rather than truth. Define review rules from business risk: a low-confidence order identifier deserves different handling from uncertain punctuation in a mailing address. Never auto-correct an amount merely because another value looks likely. Quiet corruption is worse than a visible failed job.

Search is not proof.

## Choosing the least complex engine boundary

The practical alternatives fall into three boundaries: run recognition in the application process, operate it as an isolated internal service, or call a managed document service. Compare the boundaries with the same invoice corpus and the same acceptance tests. Product feature lists do not answer the throughput question.

This bounded-batch approach has limitations. It favors predictable completion and replay over immediate results, so it does not fit an interaction that genuinely requires recognition before the caller can proceed. Separating oversized invoices improves fairness but may finish them later, while strict admission control can reject work during a surge. Those are explicit trade-offs: for synchronous low-volume use, a direct call with a hard page limit may be simpler; for sustained batches, accepting unbounded work is the more dangerous choice.

| Boundary | Operational owner | Natural pressure point | Best evidence before selection |
|---|---|---|---|
| In-process engine | Application team | CPU and memory compete with request handling | Load test showing stable latency under batch concurrency |
| Internal OCR service | Platform or application team | Queueing, worker deployment, model data, and patching | Recovery drill plus measured pages per worker |
| Managed document API | External service plus integration owner | Quotas, payload limits, data handling terms, and retry semantics | Contract review and a throttling test with representative documents |

For the reader looking for a Tesseract alternative from a Node.js service, this framing avoids a false choice. The caller can remain Node.js while the OCR boundary changes. Use a narrow job contract containing an object reference, source digest, policy version, and callback or polling identifier; do not let engine-specific response shapes leak into order processing.

Choose in-process execution only when its resource contention is acceptable and deployment remains routine. Isolating recognition adds a service and a queue, yet gives CPU-heavy work an independent scaling and failure boundary. A managed API removes engine hosting from the team's duties, while leaving admission control, idempotency, validation, retention, and incident handling firmly inside the system. No boundary erases those obligations.

Run a timed batch test that includes normal invoices, skewed pages, blank backs, mixed page sizes, duplicates, and deliberately damaged files. Record accepted pages, attempted pages, completed documents, validation failures, retries, queue age, and the oldest job. Those measures expose delivery gaps in the same way message systems do: an acceptance event is not proof of usable output.

**Select the boundary whose measured throughput meets the batch window with recovery headroom and whose data handling matches policy.** Then pin the processing-policy version and rerun the corpus before every engine or configuration change.

## Operate the result as a document supply chain

Deployment should preserve replayability. Roll out a new policy version to a small, deterministic slice of new jobs, compare validation outcomes, and stop expansion if failure rates move outside the team's declared threshold. Do not mix old and new outputs under one unlabeled status.

Alerts should focus on user-visible delay and silent loss: oldest queued job, completion rate, validation-failure rate, retry exhaustion, and mismatch between accepted and terminal job counts. CPU saturation explains pressure but does not say whether invoices will reach search before the publishing or accounting deadline.

There is one more deliberate deletion. Keep dead-letter metadata and the source reference needed for an authorized replay, but stop keeping raw page payloads in queue records after the job reaches a terminal state. When something goes wrong after that cleanup, diagnosis relies on the immutable source, policy version, structured events, and reproducible rendering. Rebuilding takes longer. The smaller retained data surface is worth that recovery cost when the policy is explicit and the replay path is tested.

## Further reading

- ISO, “ISO 32000-2:2020, Document management — Portable document format — Part 2: PDF 2.0”: https://www.iso.org/standard/75839.html
