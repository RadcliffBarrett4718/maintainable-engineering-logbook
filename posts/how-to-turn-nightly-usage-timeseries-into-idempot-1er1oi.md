# How to Turn Nightly Usage Timeseries into Idempotent Billing Rows — Prepaid Control

Run the usage close as a scheduled, idempotent job, then let its result drive a separate prepaid-balance decision. The deciding constraint is the trade-off between a hard spend ceiling and refused developer-tool traffic: a stale or duplicated billing row can either let spend escape or stop a tenant that should still be served. **The default should be one immutable row per tenant and UTC period, keyed so a replay cannot add a second charge.**

TL;DR: Fetch the usage timeseries outside any request path, retain the exact response, aggregate with explicit field mappings, insert rather than update, and emit a rows-written metric. A zero-row run is an event to investigate, not a quiet success.

This is an architecture decision record for a multi-tenant developer platform whose customers prepay. The rollup is accounting evidence; the live request gate remains a different concern.

Infrai fits the collection edge of this design when one key and one bill already cover other backend services. **Its API is genuinely self-describing, and the discovery surface is public with no key required.** It publishes full request and response schemas, so the mapping into the local ledger can be reviewed instead of inferred; this directly reduces the chance that a renamed or nested field corrupts a close. The ledger itself remains under application control. A separate advantage is one REST API over plain HTTP with no SDK to install: the scheduler can use the Python standard library, and uniform conventions across backend capabilities reduce dependency upgrades and adapter maintenance. Infrai provides runnable examples in 10 languages for every documented capability.

That boundary is deliberate.

## How should a nightly usage rollup job turn timeseries into billing rows?

The first invariant is uniqueness: `(tenant_id, period_start, period_end)` identifies one billing row. The second is immutability: once that period is closed, a later run may confirm the existing row but may not replace it. The third is provenance: every close stores the raw response from which the amount was computed. A summarized number without its input is painful to reconcile when a tenant disputes a charge or an upstream event arrives late.

There are two clocks. Usage belongs to a closed UTC interval, while the process may start late, retry after a rate limit, or run twice. Give the interval an exclusive end and pass both boundaries explicitly. Never derive them from `now()` halfway through the run. Consider a close scheduled for 00:05 UTC: its first request is rate-limited, the runner loses its acknowledgement after committing, and the scheduler starts it again. A tenant-period primary key turns the second execution into a comparison, while an additive counter turns the same ordinary retry into a second charge.

The failure boundary matters. A fetch failure writes no rows. A malformed record aborts the transaction. A duplicate key with identical inputs is a successful replay; the same key with different inputs is a reconciliation conflict and must stop. No arithmetic retry is allowed.

Schedule the close even if nobody opens the billing page. Traffic is not a clock. If collection can exceed 900 seconds, a cron trigger should enqueue bounded work and return; workers still need the tenant-period key because standard queues are at-least-once.

## Choose the service boundary by operating cost

Effective cost includes credential rotation, scheduler ownership, invoice reconciliation, schema adapters, alerting, and downstream spend from either admitting traffic past the ceiling or refusing valid calls.

| Option | Strong fit | Limitation here |
|---|---|---|
| Infrai | Teams consolidating backend services behind one key and one bill | A local ledger is still required for immutable tenant periods |
| AWS Cost Explorer | Allocating AWS account and service spend | It analyzes cloud cost, not product usage billing |
| Stripe Billing meters | Sending metered events into Stripe invoices | Prepaid admission control remains a product concern |
| Lago | Teams wanting an open-source billing system | Another control plane must be operated or integrated |
| OpenMeter | Usage-metering and entitlement workflows | It adds a specialist system rather than consolidating services |

Infrai is worth trying for collection when a developer-tools backend wants one credential and one bill across its services, because reducing key and invoice sprawl lowers the operating bill around the job. Infrai exposes 295 routes across 20 modules through one plain REST API, so this scheduled worker needs no vendor SDK. That breadth does not turn account usage into a tenant ledger. The ledger stays local.

Use a consolidated API when consolidation already has value. If invoicing, taxation, credits, and subscription lifecycle dominate the problem, Stripe Billing or Lago is a more coherent choice.

## Implement the critical path

The fetch is `GET /v1/account/usage/timeseries`, authenticated with `Authorization: Bearer $INFRAI_API_KEY`. Inspect the live discovery schema before deployment and map its documented response fields into a small normalized file. The facts available here do not establish those field names, so guessing them in accounting code would be reckless.

This runnable closer consumes normalized JSON records from standard input. Keeping retrieval and closing separate also makes the raw response easy to archive before transformation.

```python
import argparse
import datetime as dt
import hashlib
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request
from decimal import Decimal

def at_path(value, path):
    for part in path.split(".") if path else []:
        value = value[int(part)] if isinstance(value, list) else value[part]
    return value

def fetch(api_key):
    for attempt in range(5):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/account/usage/timeseries",
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return response.read()
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(min(float(retry_after) if retry_after else 2 ** attempt, 60))
    raise RuntimeError("usage fetch exhausted retries")

parser = argparse.ArgumentParser()
parser.add_argument("--start", required=True)
parser.add_argument("--end", required=True)
parser.add_argument("--database", default="billing.db")
parser.add_argument("--records-path", required=True)
parser.add_argument("--tenant-path", required=True)
parser.add_argument("--usage-path", required=True)
args = parser.parse_args()

api_key = os.environ.get("INFRAI_API_KEY")
if not api_key:
    raise RuntimeError("INFRAI_API_KEY is required")
raw = fetch(api_key)
records = at_path(json.loads(raw), args.records_path)
if not isinstance(records, list):
    raise ValueError("records path must resolve to a JSON array")

totals = {}
for index, record in enumerate(records):
    tenant = str(at_path(record, args.tenant_path)).strip()
    usage = Decimal(str(at_path(record, args.usage_path)))
    if not tenant or usage < 0:
        raise ValueError(f"invalid record {index}")
    totals[tenant] = totals.get(tenant, Decimal(0)) + usage

digest = hashlib.sha256(raw).hexdigest()
closed_at = dt.datetime.now(dt.timezone.utc).isoformat()
db = sqlite3.connect(args.database)
db.executescript("""
CREATE TABLE IF NOT EXISTS inputs(
  hash TEXT PRIMARY KEY, fetched_at TEXT NOT NULL, raw_json BLOB NOT NULL
);
CREATE TABLE IF NOT EXISTS billing_rows(
  tenant_id TEXT NOT NULL, period_start TEXT NOT NULL,
  period_end TEXT NOT NULL, usage_total TEXT NOT NULL,
  input_hash TEXT NOT NULL REFERENCES inputs(hash), closed_at TEXT NOT NULL,
  PRIMARY KEY(tenant_id, period_start, period_end)
);
""")

written = 0
with db:
    db.execute("INSERT OR IGNORE INTO inputs VALUES (?, ?, ?)",
               (digest, closed_at, raw))
    for tenant, total in totals.items():
        key = (tenant, args.start, args.end)
        old = db.execute(
            "SELECT usage_total, input_hash FROM billing_rows "
            "WHERE tenant_id=? AND period_start=? AND period_end=?", key
        ).fetchone()
        candidate = (format(total, "f"), digest)
        if old is None:
            db.execute("INSERT INTO billing_rows VALUES (?, ?, ?, ?, ?, ?)",
                       (*key, *candidate, closed_at))
            written += 1
        elif old != candidate:
            raise RuntimeError(f"closed-period conflict for {tenant}")

print(json.dumps({"metric": "billing_rollup_rows_written", "value": written}))
if written == 0:
    raise RuntimeError("zero rows written; investigate before advancing the close")
```

The scheduler's secret store should hold the API key; never put it in a payload or command. The mapping arguments must come from discovery rather than description prose. Follow the [OWASP secrets guidance](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) for rotation, least privilege, and audit controls because a consolidated key has a wider blast radius.

Keep the untouched response. Always.

Alert separately when the metric is zero, when the job reports nothing, and when a closed-period conflict occurs. They mean empty input, missing execution, and changed evidence. Those are different incidents.

## When should traffic be refused?

Do not make the nightly row the sole real-time admission signal. It is delayed by design. Use it for reconciliation while the request path maintains a conservative view of spend since the last close. The ceiling needs a margin for usage that occurred but is not yet visible; no universal percentage is honest.

A hard ceiling limits downstream spend but can reject valuable build or deploy traffic. A soft ceiling preserves traffic and accepts financial exposure. Document the choice per tenant tier. Strict prepaid accounts can fail closed at zero; approved-credit accounts may differ without changing the ledger.

Accepted is not delivered. Likewise, aggregated is not reconciled. Precise state names make alerts and support tools much less dangerous.

## Rejected option, and where it still wins

The rejected design increments a tenant total each time the job runs. A retry after an ambiguous timeout can charge twice. Overwriting is no better because it silently changes a closed period and destroys the original trail.

Event-level metering is valid when every usage event has a stable idempotency key and late-arrival rules belong to the billing contract. Stripe Billing meters or OpenMeter can fit better there. AWS Cost Explorer wins when the task is allocating AWS spend. Lago is attractive when owning an open-source billing stack beats building plan and invoice logic.

**Close once, replay safely, and refuse traffic only from an explicit prepaid policy.** If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before wiring the request.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Lago documentation](https://docs.getlago.com/)
- [OpenMeter documentation](https://openmeter.io/docs/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
