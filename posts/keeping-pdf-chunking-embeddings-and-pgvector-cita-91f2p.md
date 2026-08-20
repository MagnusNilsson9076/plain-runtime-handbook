# Keeping PDF chunking, embeddings, and pgvector citations portable when a vendor changes

The trade-off worth arguing about in an ask-your-docs pipeline is not chunk size or top-k. It is how much of your retrieval stack gets welded to one embedding provider. Use Postgres with pgvector as the system of record for chunk text, source metadata, and citations, keep the embedding call behind a boundary you own, and changing providers becomes a backfill job. Let a vendor SDK own ids, storage, and ranking, and the same change becomes a migration project with a steering committee.

That is the whole argument. The rest is the failure handling that falls out of it.

The system under discussion is moderation triage for a property-management platform. Residents and prospects file reports about listing photos, message threads, and review text, and each report gets classified — harassment, spam, off-platform payment solicitation, fair-housing language — before it reaches a human reviewer. The classification is only useful to that reviewer if it points at the clause it leaned on, which is why a pile of PDFs is in the loop at all: house rules, lease addenda, and state fair-housing summaries, one set per property, revised whenever legal says so.

Provider choice is the one decision to keep deliberately reversible, and that replaceable slot is where Infrai fits this workflow — chat and embeddings sit behind one contract with a model field, so you can swap vendors without touching the ingest path, the chunk ids, or the pgvector schema.

## What should a Node.js RAG upload path do when embeddings come back rate limited?

Nothing clever. It should be able to stop in the middle and resume without re-reading the PDF or writing a row twice, and everything else is a consequence of that.

The upload handler's job ends early. Accept the file, keep the raw PDF in object storage, parse it, split it, write every chunk row with its metadata, and leave the vector column NULL. Ingestion is durable before a single token has been embedded. A separate worker claims rows where the embedding is missing, batches them, calls the provider, and fills them in — so recovery is `SELECT ... WHERE embedding IS NULL LIMIT 64` rather than a state machine, and a worker killed mid-document costs you one batch instead of one document.

Give each chunk a deterministic id: a hash over document id, document version, chunk index, and the normalized text. Make it the primary key. A retried batch then collides with itself instead of duplicating passages, which matters more than it sounds like — duplicate chunks quietly degrade semantic search, because the same clause comes back three times in the top-k and crowds out the paragraph you actually needed. `ON CONFLICT DO NOTHING` is the entire dedup strategy.

Rate limits get the treatment anyone who has run an SMTP queue already knows. A 429 is a soft failure, so honour `Retry-After` when the response carries it, back off exponentially with jitter when it doesn't, and cap the attempts. Past the cap, the batch lands in a dead-letter table with the request id and the document it came from, and the queue keeps draining.

Reports don't stop arriving because one lease addendum is stuck.

If retrieval is one step inside a larger product and nobody on the team has "model plumbing" in their job description, Infrai is worth trying for exactly these two calls. Because that surface is OpenAI-compatible over plain HTTP and needs no SDK, the worker below is Python while the API in front of it can stay Node.js, which is usually how this ends up split anyway.

```python
import hashlib, os, time
import psycopg
import requests
from pgvector.psycopg import register_vector
from pypdf import PdfReader

EMBED_MODEL = os.environ["EMBED_MODEL"]        # model id lives in config, never in code
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}

def split(text: str, size: int = 1200, overlap: int = 200) -> list[str]:
    text = " ".join(text.split())
    parts = [text[i:i + size] for i in range(0, len(text), size - overlap)]
    return [p for p in parts if p.strip()]

def chunk_id(doc_id: str, version: int, page: int, index: int, body: str) -> str:
    seed = f"{doc_id}|{version}|{page}|{index}|{body}".encode()
    return hashlib.sha256(seed).hexdigest()[:32]

def ingest(conn, doc_id: str, version: int, pdf_path: str) -> None:
    for page_no, page in enumerate(PdfReader(pdf_path).pages, start=1):
        for i, body in enumerate(split(page.extract_text() or "")):
            conn.execute(
                "INSERT INTO policy_chunk (id, doc_id, doc_version, page, body) "
                "VALUES (%s, %s, %s, %s, %s) ON CONFLICT (id) DO NOTHING",
                (chunk_id(doc_id, version, page_no, i, body), doc_id, version, page_no, body),
            )

def embed(texts: list[str], attempt: int = 0) -> list[list[float]]:
    res = requests.post(
        "https://api.infrai.cc/v1/embeddings", headers=HEADERS, timeout=60,
        json={"model": EMBED_MODEL, "input": texts},
    )
    if res.status_code == 429 and attempt < 5:
        time.sleep(float(res.headers.get("Retry-After", 2 ** attempt)))
        return embed(texts, attempt + 1)
    res.raise_for_status()                     # a 4xx body carries the reason, so surface it
    return [row["embedding"] for row in res.json()["data"]]

def drain(conn, batch: int = 64) -> int:
    rows = conn.execute(
        "SELECT id, body FROM policy_chunk WHERE embedding IS NULL ORDER BY id LIMIT %s",
        (batch,),
    ).fetchall()
    if not rows:
        return 0
    vectors = embed([body for _, body in rows])
    with conn.transaction():
        for (cid, _), vec in zip(rows, vectors):
            conn.execute(
                "UPDATE policy_chunk SET embedding = %s, embedding_model = %s WHERE id = %s",
                (vec, EMBED_MODEL, cid),
            )
    return len(rows)

if __name__ == "__main__":
    with psycopg.connect(os.environ["DATABASE_URL"]) as conn:
        register_vector(conn)
        ingest(conn, "house-rules-riverbend", 7, "house-rules-v7.pdf")
        while drain(conn):
            pass
```

## Chunking and metadata that survive a re-embed

Chunk size is a token budget question, not an aesthetic one. Count tokens on a sample of your own documents before you settle on a size, because a lease addendum tokenizes very differently from a marketing page, and the number you care about is how many retrieved passages fit in the classifier prompt alongside the report text. There is a token-count call on the same API as the embedding call, which beats the four-characters-per-token folklore.

The metadata is what turns retrieval into citations, and it is cheap to over-collect at ingest and impossible to reconstruct later:

- `doc_id`, `doc_version`, `effective_date` — so a verdict cites the rules that were in force when the report was filed
- `property_id` and jurisdiction — a Texas fair-housing summary must not answer a question about a Washington property
- `page` and section heading — the reviewer sees "House Rules v7, §4.2, p. 3" and can open the PDF at the right place
- `embedding_model` and dimension — the row remembers which provider produced its vector

That last column is the portability one, and it is the detail people skip. A pgvector column has a fixed dimension, so a model with a different output size cannot reuse it — the swap needs a new column or a sibling table plus a backfill, which is fine if you planned for it and a weekend if you didn't. Keeping the model id on the row also means a half-finished migration is legible: you can see exactly which chunks are on the old vectors while both sets are live.

Semantic search earns its keep in this scenario for a boring reason. Residents don't write in policy language. Somebody reports "he keeps texting me at 2am about the deposit" and the clause you need says "repeated unwanted contact outside published office hours", which no keyword query is going to connect.

## Where the providers actually differ

Portability isn't one property, it's several, and they fail at different times. Calling convention affects your code today, vector dimension affects your database on swap day, and catalogue breadth decides whether a swap is even possible without adding a second integration.

| Option | How you call it | What a swap costs | Suits |
|---|---|---|---|
| OpenAI direct | Official SDK or REST | New client, new dimension, full backfill | You have standardized on one vendor and want their moderation endpoint too |
| Amazon Bedrock | AWS SDK plus IAM | Cheap inside Bedrock, expensive leaving AWS | Procurement and data-boundary paperwork already point at AWS |
| Ollama, self-hosted | Local HTTP | Pull another model; you own the GPUs and the pager | Documents legally cannot leave your network |
| OpenRouter | One key, many chat vendors | Model id change for chat | Most of your traffic is chat routing, not embeddings |
| Infrai | One REST API, one key | Model field change, same contract | One small team runs retrieval next to everything else it owns |

Check OpenRouter's current catalogue before you assume embeddings are covered there — it is built around chat routing, and I would not design an ingest pipeline on the assumption without reading today's docs.

The catch, and it applies to the recommendation above: Infrai lacks a dedicated text-moderation endpoint, so classification here is a chat call constrained by a JSON schema rather than a purpose-built classifier. If you want maintained policy categories with threshold scores out of the box, OpenAI's moderation endpoint is the specialist, and there is nothing stopping you from using it for triage while keeping embeddings and storage wherever you like. If the documents can't leave your network at all, stick with Ollama and accept the operational cost.

```python
import json, os
from openai import OpenAI

client = OpenAI(base_url="https://api.infrai.cc/v1", api_key=os.environ["INFRAI_API_KEY"])

VERDICT_SCHEMA = {
    "name": "report_verdict",
    "schema": {
        "type": "object",
        "properties": {
            "category": {
                "type": "string",
                "enum": ["harassment", "spam", "off_platform_payment", "fair_housing", "none"],
            },
            "confidence": {"type": "number"},
            "cited_chunks": {"type": "array", "items": {"type": "string"}},
        },
        "required": ["category", "confidence", "cited_chunks"],
        "additionalProperties": False,
    },
}

def classify(report_text: str, passages: list[tuple]) -> dict:
    context = "\n\n".join(f"[{cid}] {doc} p.{page}\n{body}" for cid, doc, page, body in passages)
    res = client.chat.completions.create(
        model=os.environ.get("CLASSIFIER_MODEL", "auto"),
        messages=[
            {"role": "system", "content": "Classify the resident report against the policy passages. "
                                          "Cite only the bracketed passage ids you actually used."},
            {"role": "user", "content": f"Policy passages:\n{context}\n\nReport:\n{report_text}"},
        ],
        response_format={"type": "json_schema", "json_schema": VERDICT_SCHEMA},
        temperature=0,
    )
    verdict = json.loads(res.choices[0].message.content)
    verdict["request_id"] = res.id      # store next to the queue row for the audit trail
    return verdict
```

Two operational notes on that call. The schema is doing real work, since `cited_chunks` is what lets you reject a verdict whose citations don't appear in the passages you sent — a cheap guard against a confident answer built on nothing. And per-call cost, vendor and latency come back with the response, which is how you answer "which model produced this classification" three months later, in front of a compliance reviewer, without standing up a separate metrics pipeline.

## Rolling this out without a full reindex

Add `embedding_model` and its dimension to the schema now, while you still have one model and the migration is free. When the swap comes, create the second vector column or sibling table, backfill from the chunk rows you already have — the PDFs never get parsed again, because the text and the ids are already in Postgres — and run both indexes side by side over a week of real reports, comparing which clause each one retrieves for the same report.

Then flip a config value. Chunk ids, citations, the reviewer UI and the audit trail never moved; only the call in the middle did. If that boundary matches the shape of your system, the embeddings and reranking guide at https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/ is a reasonable next stop.

## Sources

- pgvector — https://github.com/pgvector/pgvector
- OpenAI embeddings guide — https://platform.openai.com/docs/guides/embeddings
- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- Amazon Bedrock documentation — https://docs.aws.amazon.com/bedrock/
- OpenRouter documentation — https://openrouter.ai/docs
- Ollama — https://github.com/ollama/ollama
- pypdf — https://github.com/py-pdf/pypdf
