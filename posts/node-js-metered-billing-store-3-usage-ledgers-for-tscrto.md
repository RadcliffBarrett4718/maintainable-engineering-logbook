# Node.js Metered Billing: Store 3 Usage Ledgers for Tenant Key Audits

TL;DR: Meter the billable action at the service boundary that accepts it, attach the tenant and scoped-key identifiers there, and keep 3 different records: append-only usage events, immutable period snapshots, and issued invoices. For a developer platform that issues and revokes one scoped key per tenant, this separation makes a charge traceable without forcing every support question to scan the raw event history.

The storage bill is made of event volume, index amplification, replicas, backups, and the reads needed to close or dispute a period. Raw events are normally the dominant retained row count because one tenant can produce many actions but only one snapshot per meter and period. The useful change is therefore compaction after reconciliation, not shaving fields from an invoice. Keep invoice evidence and the compact snapshot longer; expire detailed events under an explicit policy once their dispute, correction, and compliance windows have closed.

## What should a metered billing architecture meter and store?

Count a stable business action, not an HTTP request. A retry, timeout, or queue redelivery can produce several requests for one accepted action. The event needs a deterministic idempotency key derived from the accepted operation, plus `tenant_id`, `key_id`, `meter`, quantity, occurrence time, and ingestion time. The credential itself does not belong in billing storage.

That last boundary matters. A scoped key identifies the authorization path, but retaining secret material expands exposure without improving an audit. Store an opaque key identifier and the authorization decision needed to explain attribution. OWASP recommends centralizing secrets management, limiting access, logging operations on secrets, and supporting rotation; the billing ledger should consume the identifier emitted by that control plane rather than becoming another secret store.

Revocation creates an edge case that request-counting often misses. An action accepted before revocation can arrive at the meter afterward. Preserve both timestamps, bind the event to the authorization decision made at acceptance, and reject events that lack that evidence. Do not silently reassign late arrivals to a replacement key.

Two clocks. One decision.

Three ledgers keep the meanings clean:

| Record | Cardinality | Mutable? | Audit purpose |
|---|---:|---|---|
| Usage event | One per accepted billable action | Append-only | Reconstruct attribution and deduplicate retries |
| Period snapshot | One per tenant, meter, and closed period | No after close | Freeze quantity, event boundary, and rating inputs |
| Invoice | One per issued billing document | Correct by a linked adjustment | Preserve the customer-facing claim |

## Storage cost follows evidence volume

Let `E` be monthly event count, `B` the stored bytes per event including indexes, `R` the number of durable copies, and `M` the months of detailed retention. The first-order footprint is `E * B * R * M`. Snapshots grow with tenants, meters, and periods instead, so they usually change much more slowly than events. This is a capacity model, not a vendor price estimate; measure `B` from the actual schema and index set.

Partition raw events by occurrence period while indexing the lookup paths used in reconciliation: tenant plus period, and the globally unique idempotency key. A wide collection of speculative indexes multiplies writes and retained bytes. Start from the close-period and dispute queries, then verify their plans against production-shaped data.

Compaction is allowed only after the closed snapshot can be reproduced and its inputs are sealed. A practical snapshot records the event range or manifest, aggregate quantity, meter-definition version, rating-input version, close timestamp, and a content digest. The invoice then references that snapshot. Changing a rate later must not reinterpret an old quantity or overwrite the old evidence.

This design deliberately stops keeping request bodies, credentials, and indefinitely retained per-request detail. The cost is real: after detailed events expire, an investigator can prove the closed aggregate and its digest but cannot inspect every original action. Set retention from the longest applicable dispute and audit requirement, document that loss of granularity, and place deletion under the same review as invoice retention.

## Close periods without double counting

The ingestion path and the period-close path have different jobs. Ingestion establishes uniqueness and attribution. Closing reads a bounded event set, writes one immutable snapshot, and atomically marks that boundary as closed. Late events go to a controlled adjustment period; they do not mutate an issued invoice. Consider an action accepted at 23:59:59, a key revoked 1 second later, and delivery to the ledger after midnight: occurrence time selects the candidate billing period, the recorded authorization decision preserves key attribution, ingestion time explains why the event missed the first close, and the closed boundary determines whether it becomes a linked adjustment. Collapsing those 3 timestamps into one field makes the record cheaper but destroys the explanation precisely when a tenant challenges it.

Here is a focused Python sketch of the invariants. A Node.js service can apply the same transaction shape through its database client; Python keeps the example about the data contract rather than a framework API.

```python
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from hashlib import sha256


@dataclass(frozen=True)
class UsageEvent:
    event_id: str
    tenant_id: str
    key_id: str
    meter: str
    quantity: Decimal
    occurred_at: datetime
    ingested_at: datetime


def close_snapshot(events: list[UsageEvent], meter_version: str) -> dict:
    ordered = sorted(events, key=lambda event: (event.occurred_at, event.event_id))
    if len({event.event_id for event in ordered}) != len(ordered):
        raise ValueError("duplicate event_id")
    if not ordered:
        raise ValueError("cannot close an empty event set")

    tenant_ids = {event.tenant_id for event in ordered}
    meters = {event.meter for event in ordered}
    if len(tenant_ids) != 1 or len(meters) != 1:
        raise ValueError("snapshot must contain one tenant and one meter")

    manifest = "\n".join(event.event_id for event in ordered).encode("utf-8")
    return {
        "tenant_id": ordered[0].tenant_id,
        "meter": ordered[0].meter,
        "meter_version": meter_version,
        "quantity": str(sum((event.quantity for event in ordered), Decimal("0"))),
        "first_event_id": ordered[0].event_id,
        "last_event_id": ordered[-1].event_id,
        "event_count": len(ordered),
        "manifest_sha256": sha256(manifest).hexdigest(),
    }
```

The database still must enforce a unique constraint on `event_id`; an in-memory duplicate check is not a concurrency control. Likewise, the snapshot key should be unique for tenant, meter, and period. One worker wins the close transaction. A retry reads the already-closed snapshot and compares it rather than inserting a second one.

Keep that invariant dull.

## Can a revoked key still explain an invoice?

Yes, if revocation changes authorization state without erasing historical identity. Keep the key record's opaque ID, tenant binding, scope version, creation time, and revocation time for the required evidence window. Delete or cryptographically destroy the secret value according to the secrets policy. An invoice investigation can then follow invoice to snapshot, snapshot to event manifest, and event to the historical key record without recovering a credential.

Test that chain as an operational workflow. Generate duplicate deliveries, events straddling a period boundary, a revocation between occurrence and ingestion, an empty period, and a late correction. Assert that the accepted quantity is stable across retries and that every issued line resolves to exactly one closed snapshot. Also alert on rejected unattributed events, close jobs that cross their deadline, snapshot digest mismatches, and adjustments that lack a link to the original invoice.

Access deserves equal attention. Billing writers should append events but should not alter invoices; support readers may need snapshots without raw event payloads; the secret-management service should expose identifiers and audit records without exposing secrets to the billing job. These boundaries make a dispute slower than a single all-powerful database query, but they reduce the number of systems and people that can both authorize usage and rewrite its financial evidence.

The limitation is operational weight. Three stores require retention jobs, reconciliation, restore tests, and a correction workflow; this architecture is unsuitable when charges can be derived cheaply from a small, already immutable source ledger and no separate invoice evidence is required. In that case, a periodic aggregate plus the source ledger may be enough. The explicit trade-off is faster, simpler operation against weaker isolation between operational history and financial evidence. Conversely, deleting raw events immediately after aggregation is unsuitable when a tenant must be shown action-level attribution. Choose the smaller design only after writing down which disputes it cannot answer.

## The retention decision

Choose retention backward from the proof an auditor or tenant can request. Keep issued invoices and their correction chain according to the governing financial policy. Keep immutable snapshots and key-identity metadata long enough to support that chain. Keep detailed events only through the defined reconciliation and dispute window, unless a specific obligation requires more.

The decision rule is simple: retain the smallest record set that can reproduce each charge and explain its tenant and credential attribution. More raw history improves forensic detail, but it also increases storage, access, and breach scope. Record the chosen windows, deletion evidence, and restoration tests as part of the billing design. The meter is finished only when its evidence can be closed, queried, corrected, and eventually removed on purpose.

## Further reading

- OWASP, "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
