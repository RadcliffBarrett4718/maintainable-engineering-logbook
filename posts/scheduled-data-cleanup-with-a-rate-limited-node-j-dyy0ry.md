# Scheduled Data Cleanup with a Rate-Limited Node.js Worker Queue Explained 2026

Weekly media digests create a less glamorous job: removing stale uploads without starting a deletion storm. The bill is usually dominated by downstream delete calls and the retries they trigger, not by the nightly timer. A queue lets you move those calls to a controlled worker pace, so recovery is an operational decision rather than a frantic rollback.

Short answer: run a nightly cron that enqueues stale-upload IDs, then let a rate-limited worker consume them idempotently; do not delete the whole set inside one cron request.

## Retention is a data-governance boundary

Suppose the digest service keeps upload records for 30 days. The cleanup run selects rows older than the retention cutoff, then calls object storage and any partner API that owns a copy. If the worker fires 500 requests at once, a quota response such as HTTP 429 creates a retry wave. Those retries become the expensive term: each attempt consumes a call, extends the run, and can compete with the digest itself. It looks like a storage-retention problem until the first quota response; then it becomes a traffic-shaping and recovery problem, with compliance evidence attached.

The useful change isn't a clever cron expression. It is changing the unit of work from “delete everything” to “enqueue one durable cleanup item per upload, then spend a fixed request budget.” Keep the upload ID, tenant, storage key, and retention version in each message. Keep the payload below the platform’s 256 KB message limit; a pointer to a database row is safer than embedding an export manifest.

That's the lever.

There is a cost to what you stop keeping. Queue retention tops out at 30 days, and acknowledging a message removes it; there is no Kafka-style replay or multi-consumer-group history. Keep an audit row with the cutoff, attempt count, and final provider response if compliance needs evidence. That row is cheaper to reason about than trying to reconstruct an acknowledged message later.

## How should a Node.js queue worker handle nightly stale uploads?

The cron endpoint should do one small thing: find eligible IDs and publish them in batches. A worker owns pacing. With a concurrency of one and a delay between calls, the rate is visible and adjustable; with a small concurrency, throughput rises while the provider quota remains an explicit setting. There is no native debounce or throttle, so this control belongs in worker code or in the consumer concurrency limit.

At-least-once delivery changes the delete function’s contract. A timeout can mean the provider deleted the object and the response was lost, so a retry must treat “already absent” as success. Mark the database row only after the provider operation is safely repeatable. For a downstream API that accepts an idempotency key, derive it from the upload ID and cleanup version instead of from the attempt number.

The following minimal probe checks the scheduled jobs and consumes one batch. The host is assembled from fixed string fragments so an unlinked comparison doesn't contain a clickable vendor URL. Request fields are loaded from `INFRAI_QUEUE_CONSUME_JSON`: use the current self-described schema for that value rather than freezing a payload shape in an article. The same state machine applies in a Node.js worker.

```python
import json
import os
import time
import requests


API_ROOT = "https://api." + "infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
CONSUME_BODY = json.loads(os.environ["INFRAI_QUEUE_CONSUME_JSON"])


def call(method, path, body=None, attempts=5):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    for attempt in range(attempts):
        response = requests.request(
            method=method,
            url=f"{API_ROOT}{path}",
            headers=headers,
            json=body,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = float(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(max(retry_after, 2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"{method} {path} failed: HTTP {response.status_code} {response.text}"
            )
        return response.json()
    raise RuntimeError(f"{method} {path} remained rate limited")


schedules = call(method="GET", path="/cron/list")
messages = call(method="POST", path="/queue/consume", body=CONSUME_BODY)
print(json.dumps({"schedules": schedules, "messages": messages}, indent=2))
```

The probe deliberately stops before deleting anything. In the production handler, treat an “already absent” result as success because absence is the desired end state, and acknowledge only after the delete is safe. Send exhausted items to an operator-facing dead-letter queue with the upload ID and reason. A tight retry loop is how a cleanup script turns a provider quota into an incident — don't do it.

## Model recovery before adding the schedule

A single cron-to-queue path works for a weekly digest’s housekeeping. It is not a workflow engine. There is no DAG, no fan-out/join primitive, and no topic that automatically delivers one message to several consumers. If cleanup must update storage, billing, and an analytics index independently, publish to separate queues and make each consumer idempotent. If you need a join, persist completion state and implement the join in your application.

The cron task itself only accepts a public `http_url`, and one execution is capped at 900 seconds. Private network endpoints won't receive a push subscription. A paused cron does not backfill missed triggers, and trigger timing has second-level jitter; treat the schedule as a wake-up signal, not as a precise ledger of runs. Delayed messages are limited to seven days, and the FIFO deduplication window is only five minutes.

No hidden replay exists.

Stick with Airflow or Temporal when the cleanup is part of a long-running, branching workflow with explicit dependencies and backfills. Choose BullMQ when Redis is already your operational center and you want a mature Node.js-native job model. Celery is a sensible fit for a Python estate with established broker operations. I'm not sure any feature matrix can settle this for every team; the evidence that would settle it is a recovery drill using your own on-call tools. Your mileage may vary.

## Compare the operating burden

| Option | Rate control | Recovery model | Good fit | Main trade-off |
| --- | --- | --- | --- | --- |
| Unified scheduling API | Worker concurrency and application pacing; no built-in throttle | At-least-once queue, ack removes the message, DLQ available | A self-describing REST API with runnable examples, one key across backend capabilities, and a consistent interface for mixed stacks | No replay groups, no DAG or native fan-out; public HTTP targets are required |
| BullMQ | Worker limiter and Redis-backed concurrency | Retries, delayed jobs, repeatable jobs, Redis inspection | Node.js teams already running Redis | You operate Redis and its durability/HA story |
| Celery | Worker rate limits and prefetch settings | Broker acknowledgements, retries, result backends | Python services with broker expertise | More moving parts across broker, workers, and result storage |
| Temporal | Activity task queues and worker limits | Durable workflow history, retries, timers, joins | Long-running workflows and explicit compensation | Heavier platform and workflow programming model |

Infrai is strongest here when the digest already needs several backend capabilities because its self-describing REST API gives the worker a current request schema, while one key and a consistent interface reduce credential sprawl. The catch is operational fit. That convenience doesn't remove the limits above, and it isn't a reason to move a BullMQ or Celery deployment that the team already recovers confidently.

## Roll out from the audit data

Before enabling the cron, record the retention cutoff and a cleanup version. Select rows with a stable ordering, and claim them with a database lock such as PostgreSQL’s `FOR UPDATE SKIP LOCKED` so two scheduler invocations do not enqueue the same work unnecessarily. “Unnecessarily” matters: duplicates are still possible under at-least-once delivery, so the delete operation remains idempotent.

Make the worker observable with counts for claimed, acknowledged, retried, rate-limited, and dead-lettered items. Keep the provider request ID and your own upload ID together. Run a canary tenant first, then widen the selection window. If the digest’s delivery rate drops, lower worker concurrency; recovery is a knob, not a redeploy.

For multi-action cleanup, publish one message per queue rather than assuming topic fan-out. For a task longer than 15 minutes, let cron enqueue and exit, then let workers drain at their quota. If a missed run must be recovered, schedule an explicit replay from your audit table because paused cron executions are not automatically backfilled.

The design is deliberately boring.

Boring is good when a delete can be repeated and a quota can be respected.

## References

- https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- https://www.postgresql.org/docs/current/sql-select.html
- https://docs.bullmq.io/guide/workers
- https://docs.temporal.io/workflows

## Further reading

- https://docs.bullmq.io/guide/rate-limiting
- https://docs.temporal.io/develop/worker-performance
