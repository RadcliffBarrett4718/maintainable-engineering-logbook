# Hosted Vector API Setup: Fresh Clinical Evidence for Two-Person Teams

A healthtech page monitor has an awkward constraint: a tiny edit to eligibility, dosage, or contact guidance must become searchable quickly without making unchanged text look new. **The simplest setup for a two-person team is the one that can replace every chunk from one page atomically, filter by tenant and revision, and expose enough state to prove that an alert refers to the current page.** Choose the operating model only after that test. A generic hosted vector API reduces infrastructure ownership; Pinecone packages vector search as a managed service; Weaviate offers managed and self-hosted deployment paths; and PostgreSQL with pgvector keeps vectors beside relational state. None removes the need to design chunk identity and freshness.

TL;DR: start with page-level versioning, deterministic chunk IDs, metadata filters, and a small reconciliation worker. Run one representative-page trial against each acceptable operating model. Prefer fewer moving parts when the candidates meet the same freshness invariant; reject any candidate that cannot demonstrate complete replacement, tenant isolation, and deletion behavior under retry.

## What should a two-person team require from a hosted vector API?

Retrieval-augmented generation joins a retriever with a generator so answers can use external evidence. For a monitored health page, however, retrieval quality begins before a query arrives. The crawler first has to decide whether a revision matters, split it without losing clinical context, publish the new evidence, and retire the old evidence. An alert can be syntactically correct and still quote yesterday's paragraph.

That is the dangerous gap.

HTML hashes alone are too noisy because navigation, timestamps, and consent banners may change while the medical content does not. Whole-page embeddings move in the other direction: they suppress detail and make it hard to show exactly what changed. The useful unit is usually a stable content region such as a section under a heading, with smaller chunks derived inside that region. Preserve the heading path in every child chunk. A sentence about “call your clinician” means something different beneath “Missed dose” than beneath “Adverse reactions.”

Chunk size is therefore a semantic and operational decision, not an arbitrary token target. Smaller chunks localize diffs and reduce the text that must be re-embedded after an edit. They also increase record count and can detach qualifiers from instructions. Larger chunks preserve context but cause more write amplification and may return unrelated sentences. Use the smallest chunk that retains the condition, action, and exceptions together, then measure retrieval with actual changed passages.

## Make freshness an explicit contract

The index should never be the sole record of what was fetched. Keep a relational manifest for each tenant and canonical URL: normalized content hash, source revision, fetch time, publication state, and the IDs expected in the vector index. The vector records carry the same `tenant_id`, `page_id`, `revision`, `region`, and content hash. That duplicated metadata is intentional. It lets retrieval reject stale revisions and lets reconciliation find missing or surplus records.

A practical publication sequence is:

1. Fetch the page conditionally when the server supplies validators such as `ETag` or `Last-Modified`. HTTP conditional requests can avoid transferring an unchanged representation, but a validator is not a substitute for normalizing the clinical content.
2. Parse only allowed content regions, canonicalize whitespace and links, and compute a content hash.
3. Derive chunks and deterministic IDs from tenant, page, revision, region, and chunk ordinal.
4. Write the new revision's vectors idempotently. Verify the expected count.
5. Mark that revision current in the manifest, then remove older records asynchronously.

This order favors availability while preventing retrieval from treating a half-written revision as current. Retrieval filters on the manifest's active revision as well as the tenant. During cleanup, old vectors may exist physically, but they are ineligible. If the vector system cannot filter on both fields, the application must enforce the boundary before any text reaches the generator.

Freshness needs numbers, even when there is no universal target. Track `source_observed_at`, `index_committed_at`, and `alert_emitted_at` separately. Their differences identify crawl lag, indexing lag, and notification lag. Set the service objective from clinical risk and source behavior rather than from a vendor's nominal write latency. A two-minute index is useless behind a six-hour crawler.

## A narrow Python contract keeps the choice reversible

Avoid leaking a provider's response objects through the application. The interface only needs replace, query, and delete semantics plus filters. The worker below shows the critical part: IDs are deterministic, writes are retriable, and activation happens only after all records are accepted. `VectorStore` and `Manifest` are deliberately generic interfaces; their implementations are where managed API, self-hosted service, or PostgreSQL details belong.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Protocol, Sequence


@dataclass(frozen=True)
class Chunk:
    region: str
    ordinal: int
    text: str


class VectorStore(Protocol):
    def upsert(self, records: Sequence[dict]) -> None: ...
    def delete_older(self, tenant_id: str, page_id: str, revision: str) -> None: ...


class Manifest(Protocol):
    def activate(self, tenant_id: str, page_id: str, revision: str,
                 chunk_ids: Sequence[str]) -> None: ...


def publish_revision(store: VectorStore, manifest: Manifest, *, tenant_id: str,
                     page_id: str, revision: str, chunks: Sequence[Chunk],
                     embeddings: Sequence[Sequence[float]]) -> None:
    if len(chunks) != len(embeddings):
        raise ValueError("each chunk must have exactly one embedding")

    records = []
    for chunk, vector in zip(chunks, embeddings, strict=True):
        identity = f"{tenant_id}:{page_id}:{revision}:{chunk.region}:{chunk.ordinal}"
        chunk_id = sha256(identity.encode("utf-8")).hexdigest()
        records.append({
            "id": chunk_id,
            "vector": list(vector),
            "text": chunk.text,
            "metadata": {
                "tenant_id": tenant_id,
                "page_id": page_id,
                "revision": revision,
                "region": chunk.region,
            },
        })

    store.upsert(records)
    manifest.activate(tenant_id, page_id, revision, [r["id"] for r in records])
    store.delete_older(tenant_id, page_id, revision)
```

This is not a distributed transaction. A crash after `upsert` and before `activate` leaves an inactive revision that reconciliation can retry or delete. A crash after activation but before cleanup leaves old, filtered-out records. Both outcomes are safer than deleting the old revision first. The same pattern also makes duplicate delivery from a queue harmless, provided upsert by ID is idempotent.

Be strict at the tenant boundary. Do not accept `tenant_id` from generated text or from a model tool call; derive it from authenticated application context and include it in every storage operation. For additional defense, separate namespaces, collections, or database row-level security may help, but those mechanisms do not excuse a missing filter in the application contract.

## Compare operating models after the invariant

Once the pipeline is explicit, the product comparison becomes smaller and more honest. Pinecone documents namespaces as a multitenancy mechanism and metadata filters for narrowing search. Weaviate documents multi-tenancy on collections and supports both a managed cloud service and self-hosted deployment. pgvector is a PostgreSQL extension that provides exact and approximate nearest-neighbor search while retaining PostgreSQL features such as transactions and joins. A generic hosted vector API may offer a thinner version of the same primitives, but its label says nothing about deletion guarantees, filter behavior, backups, or observability; those must be verified in its contract and trial.

| Operating model | What the team operates | Useful fit for this monitor | Boundary to verify |
| --- | --- | --- | --- |
| Managed vector service | Ingestion, manifests, alerts, and application policy | Minimal database operations when vector search is already isolated | Read-after-write behavior, filtered deletes, export, and per-tenant controls |
| Managed or self-hosted vector database | The same application pipeline; infrastructure burden depends on deployment | A path to more schema and deployment control | Upgrade ownership, backups, multi-tenant configuration, and resource isolation |
| PostgreSQL plus vector extension | One relational system, extension lifecycle, and index tuning | Manifest and vectors can share transaction and query boundaries | Query latency under mixed relational/vector load and approximate-index behavior |
| Thin hosted vector API | Application owns more lifecycle semantics | Small surface area when required filters and deletes are complete | Durability, reconciliation visibility, quotas, and recovery procedures |

The comparison is intentionally not a ranking. Each option has limitations. For two engineers, a managed vector service limits infrastructure work but adds a separate data system, contract, and control plane. Self-hosting offers deployment control at the cost of owning upgrades, backups, capacity, and recovery. PostgreSQL with pgvector avoids another datastore, yet mixed transactional and vector workloads share resources and require index tuning. A thin hosted API is not suitable when it cannot expose revision filters, deterministic deletion, or recovery evidence; choose a fuller service or a relational implementation instead. These are concrete trade-offs, not feature-score deductions.

Compliance review also cares about data location, encryption, access logs, retention, deletion, subprocessors, and incident procedures. Those are deployment-specific facts. Confirm them in the applicable service agreement and architecture; do not infer them from “managed,” “serverless,” or “self-hosted.”

Test behavior rather than brochure nouns. Seed two tenants with overlapping language, update one page from revision A to B, interrupt publication between each step, replay the job, and delete the page. Then assert that queries never cross tenants, never return inactive A after B is active, and return no page content after deletion completes. Add malformed HTML, a removed heading, reordered sections, and a page that returns `304 Not Modified`. This compact suite exposes more risk than a broad feature checklist.

## Roll out with a shadow index, then simplify

Build a corpus from a small set of representative pages: one long policy, one table-heavy eligibility page, one frequently edited advisory, and one page whose navigation changes often. Record expected diffs and retrieval results. Run candidate backends in shadow mode from the same normalized chunks, without sending duplicate alerts. Compare revision completeness, stale-hit rate, tenant-filter correctness, indexing lag, and recovery after injected failures.

Migrate by page, not by individual chunk. Dual-write a page revision, validate its count and sample queries, switch the manifest's read target, and retain the prior target only for a bounded rollback window consistent with retention policy. Stop dual writes when the chosen path has survived retries, deletes, and restore testing.

Then simplify aggressively. Keep one source manifest, one chunking implementation, one alert deduplication key, and one retrieval contract. The durable decision is not a brand choice. **It is a system in which every alert can name the tenant, page, source revision, observed time, and exact evidence that caused it.**

## References and Sources

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- HTTP Semantics, conditional requests and validators: https://www.rfc-editor.org/rfc/rfc9110.html
- Pinecone multitenancy: https://docs.pinecone.io/guides/index-data/implement-multitenancy
- Pinecone metadata filtering: https://docs.pinecone.io/guides/search/filter-by-metadata
- Weaviate multi-tenancy: https://docs.weaviate.io/weaviate/manage-collections/multi-tenancy
- Weaviate deployment options: https://docs.weaviate.io/deploy
- pgvector project documentation: https://github.com/pgvector/pgvector
- PostgreSQL row security policies: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
