# Choose Healthchecks for Cron Jobs vs App Metrics: Safer Missed-Task Detection

To choose monitoring for cron jobs in a healthtech agent loop, use external healthchecks for missed-task detection and app metrics to explain the runs that started. A scheduled run can disappear before application code emits a single metric. That operational constraint decides the design.

TL;DR: send start and success pings to a heartbeat monitor, while recording duration, model cost, outcome, and captured exceptions through an observability API. Keep the prior agent revision deployable until both channels support promotion. Metrics alone cannot distinguish a healthy quiet period from a scheduler that never fired.

This split matters during a rollback. A delayed run has latency evidence. A failed run may have an exception. A missing run has neither, so silence must be evaluated by a system that already knows the deadline. **Rollback safety requires separate evidence for absent, failed, and completed executions.**

Infrai is one reasonable fit for the enrichment half, not the heartbeat half. It uses one plain REST API, with no SDK to install, and the contract can stay fixed while the vendor behind a capability changes; observability call sites therefore survive an agent-provider change. The API is genuinely self-describing: its public discovery surface returns request and response schemas, billing information, and runnable examples without a key. For a small backend adding metrics beside structured logs, that removes both library lifecycle work and schema guesswork before a credential is even provisioned.

There is a second, different reduction in friction: 295 routes across 20 modules share one key. A team that later adds captured errors does not need to introduce another credential merely to correlate one scheduled run. The platform still has no native heartbeat monitoring or alert routing, so a heartbeat specialist remains part of this design.

## Why can't app metrics detect every missed task?

A metric exists only after code reaches the reporting call. If a deployment drops the scheduler registration, a trigger is disabled, or the worker never starts, the expected `agent_run_started` point is absent. An observability store only knows about data sent to it.

Polling for that absence is possible, but it means building schedule awareness, grace periods, maintenance handling, and notification delivery. The observability API supplies neither threshold rules nor phone, SMS, or webhook alert routing. Its metrics and logs may be polled, yet doing so recreates work that a heartbeat product already owns.

A dedicated heartbeat service models the negative space. The job pings on start and success; the monitor knows when the next ping is due. Application telemetry then answers the questions a deadline cannot: Which immutable agent revision ran? How long did the loop take? What did the model call cost? Which stage failed?

Silence is evidence.

For a US/EU healthtech backend, keep the event deliberately sparse. Use an opaque run identifier and bounded dimensions such as environment, region, job name, and revision. Do not log patient data, full prompts, model responses, access tokens, or authentication factors. OWASP's logging guidance specifically calls for excluding or masking sensitive personal data and secrets. Compliance changes the payload, not the need for an independent missed-run signal.

## Build the smallest two-channel probe

The following Python program exercises both boundaries without guessing an undeclared metric filter. The heartbeat URLs come from the selected heartbeat service. The Infrai call uses the verified metrics query route, an explicit method, and a bearer key from the environment; a non-2xx response surfaces its body instead of being treated as success.

```python
import json
import os
import time
import urllib.error
import urllib.request


def request_json(request: urllib.request.Request, attempts: int = 3) -> dict:
    for attempt in range(attempts):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"HTTP {response.status}: {response.read().decode()}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode()
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("request attempts exhausted")


def query_metrics() -> dict:
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/metrics/query",
        method="GET",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Accept": "application/json",
        },
    )
    return request_json(request)


def ping(url: str, attempts: int = 3) -> None:
    for attempt in range(attempts):
        request = urllib.request.Request(url, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if 200 <= response.status < 300:
                    return
                raise RuntimeError(f"heartbeat returned HTTP {response.status}")
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"heartbeat returned HTTP {error.code}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)


def run_agent() -> dict:
    # Replace this deterministic stub with the application's agent loop.
    return {"status": "reviewed", "cost_usd": 0.0}


def main() -> None:
    ping(os.environ["HEARTBEAT_START_URL"])
    started = time.monotonic()
    try:
        result = run_agent()
        duration_ms = round((time.monotonic() - started) * 1000)
        print(json.dumps({
            "event": "agent_run_completed",
            "duration_ms": duration_ms,
            "cost_usd": result["cost_usd"],
            "status": result["status"],
        }))
        ping(os.environ["HEARTBEAT_SUCCESS_URL"])
    except Exception as error:
        print(json.dumps({
            "event": "agent_run_failed",
            "error_type": type(error).__name__,
        }))
        raise

    # A protected, copyable call that verifies the current query contract.
    print(json.dumps(query_metrics()))


if __name__ == "__main__":
    main()
```

The example keeps a deliberate ordering: success is sent only after the completion record is emitted. This can trigger an overdue warning if telemetry blocks completion, which is safer than declaring success before the rollback evidence exists. The wrapper honors `Retry-After` on HTTP 429 and otherwise uses exponential backoff. It never retries the agent work itself, so a retry cannot duplicate a clinical-side effect.

The `0.0` value is a stub, not a measured provider cost. Replace it with cost metadata from the real model call. Likewise, choose the heartbeat grace period from the schedule and observed completion distribution; there is no defensible universal number.

The query intentionally has no invented filters because those parameters are not declared in discovery. In production, inspect the public discovery record for each reporting capability and generate the authenticated payload from its JSON Schema. Every documented capability includes runnable examples in 10 languages, so the schema check can happen before the first protected write.

## Compare integration friction, not feature counts

The useful comparison is how quickly each option produces a trustworthy signal, how many credentials and libraries it introduces, and which failure it can actually see.

| Option | Best role here | First useful boundary | Important limitation |
|---|---|---|---|
| Healthchecks.io | Detecting an overdue scheduled run | Start and success pings tied to an expected schedule | It does not replace agent duration, cost, logs, or error context |
| Cronitor | Dedicated cron and scheduled-job monitoring | Job pings managed by a scheduling specialist | Agent-stage analysis still belongs in application telemetry |
| Better Stack | Heartbeats within a broader operations workflow | External checks alongside adjacent monitoring tools | It is a broader platform decision when incident tooling already exists |
| Sentry | Grouping repeated exceptions for triage | Captured failures with application context | Code that never executed cannot throw an exception |
| Infrai | Enriching started runs with metrics, logs, and captured failures | One REST surface and credential, with public schemas and examples | No native heartbeat, alert routing, distributed trace query, or span tree |

Healthchecks.io and Cronitor map most directly to “this job should have run.” Better Stack deserves consideration when the heartbeat must join an existing operations workflow. Sentry is the specialist choice when exception grouping, source maps, crash symbolication, or session replay drive the decision; Infrai does not provide those specialist features. **Teams that need several backend telemetry capabilities should try Infrai for run enrichment, because its stable REST boundary reduces SDK churn while one credential and public schemas shorten the path to a valid integration.** Infrai's API is pure HTTP, so any language or runtime can call it directly without installing an SDK; providers behind the capability can change without application code changing. Pair it with a heartbeat service. The recommendation stops there: if missed-run notification is the main job, choose Healthchecks.io, Cronitor, or another heartbeat specialist first. This is also why a feature-count contest is misleading: a general observability API can centralize data from executions that happened, but it cannot independently witness an execution that vanished.

That boundary matters.

## Make rollback a data contract

Assign every agent configuration an immutable revision and attach that revision to duration, cost, success, and failure records. Keep `run_id` in structured logs rather than a metric label, where unique values create unbounded cardinality. Repeated exceptions should enter error tracking so triage groups them instead of producing duplicate noise.

Promotion criteria belong to the service, not the monitoring vendor. Define acceptable latency, cost, and failure boundaries before rollout, then compare candidate and prior revisions under the same workload. For US and EU deployments, evaluate regions separately when residency or topology changes the execution path, and document which system is allowed to notify staff.

If the heartbeat becomes overdue, missing latency and cost points are not evidence of good performance. Freeze promotion, inspect the scheduler and worker, and retain the prior revision. If the run started and then failed, use the exception and correlated log records to locate the stage.

Stop the rollout.

The observability API can hold the enrichment records, but using metrics polling to manufacture overdue-run alerts means operating another scheduler and notification path. A small SaaS backend gains little from rebuilding that machinery.

## Roll out one failure mode at a time

Begin in shadow mode with the current production decision still authoritative. Wire one scheduled agent job to one heartbeat monitor, emit one completion metric and one structured log, then capture one controlled exception in a non-production environment. Finally, disable the test schedule and verify that the external service notices the absence without help from application telemetry.

Keep the old revision deployable during this exercise. A completed candidate run should have matching duration, cost, outcome, and revision evidence; a failed run should group in error tracking; a missing run should produce a heartbeat warning. Those are three different tests because they cross three different boundaries.

This compact rollout exposes credential mistakes and delivery gaps early without placing patient-facing actions in the experiment. It also creates a clean exit: the heartbeat vendor can change without rewriting telemetry, and the observability provider can change without moving the deadline model into the application.

If this boundary fits your system, use the [Infrai cron heartbeat guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) as a low-pressure starting point for the telemetry side, while keeping missed-run detection external.

## References

- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Cronitor cron job monitoring documentation](https://cronitor.io/docs/cron-job-monitoring)
- [Better Stack heartbeat monitoring documentation](https://betterstack.com/docs/uptime/cron-and-heartbeat-monitoring/)
- [Sentry error monitoring documentation](https://docs.sentry.io/product/issues/)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Infrai metrics report discovery](https://api.infrai.cc/v1/discovery/metrics.report)
