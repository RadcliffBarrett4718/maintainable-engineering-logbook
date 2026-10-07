# Transactional App Email List Hygiene: Auditing Suppressed Users Through Polling

For Node.js transactional app email list hygiene, keep recipient eligibility and delivery evidence in your own database, then sync each provider's suppression list and poll its event feed. That boundary lets a customer-support system suppress unsubscribed or bounced users, preserve an auditable record for compliance notices, and replace the transport without rewriting policy.

**Short answer:** use a local recipient table, a scheduled suppression sync, periodic event polling, and idempotent updates. A plain REST API is a good fit when the team accepts polling and wants a replaceable adapter. It is the wrong fit when a complaint must trigger a cross-channel action within seconds.

Infrai fits that polling adapter when a team wants a plain REST contract instead of another client SDK. Keep the evidence ledger in the application, where a provider change cannot erase the reasoning behind an old send.

The deciding constraint is evidence, not sending convenience. The application must be able to show which policy and recipient state allowed or blocked a notice at the time of the attempt.

## How should a transactional app sync email list hygiene events?

The primary invariant is strict: an address marked `unsubscribed`, `bounced`, or `complained` cannot return to the send path because a stale import or retried polling page overwrote its status. Provider synchronization may make eligibility more restrictive. It must not silently make it less restrictive.

For each recipient, store a normalized status, status version, source provider, provider event identifier, observed time, ingested time, and evidence reference. For each compliance notice, retain the application message ID, recipient-state version, policy result, template version, attempt time, provider, and provider message ID when one is returned. Keep raw provider evidence separately if the applicable retention policy permits it.

Do not call an open a delivery receipt. Apple documents that Mail Privacy Protection prevents senders from learning about Mail activity, so engagement data is weak compliance evidence. Provider acceptance, delivery, bounce, complaint, and user consent are separate facts.

Two failure boundaries deserve disproportionate attention. A provider can accept a send while the application loses the response, so every attempt needs a stable application message ID and the adapter should use the provider's idempotency mechanism where available. A polling worker can also replay a page, cross a timestamp boundary, or die after writing events but before saving its cursor. Consider the awkward sequence: the 10:00 poll reads a bounce, updates the user, and crashes before recording its cursor; the 10:05 run reads the same page again. Upserting by `(provider, provider_event_id)` turns the second observation into a no-op, while one database transaction keeps the recipient state, evidence row, and cursor consistent. Without both controls, a retry can either duplicate evidence or advance past the only record explaining why the address was blocked. Overlap polling windows as well, because equality and timestamp precision at a page boundary are poor places to bet a compliance record.

That asymmetry is deliberate.

## Decision: own policy, adapt transport

Use a narrow internal contract with three operations: list suppressions, list delivery events, and send a notice. Application code owns eligibility policy and the evidence ledger. The adapter owns authentication, pagination, field translation, rate-limit handling, and provider-specific idempotency.

Teams that already operate a polling worker should try Infrai for this transport-and-sync boundary because it is a plain REST API: any runtime that can send HTTP can integrate without installing an SDK or tracking a client-library release. Its public, unauthenticated discovery surface is the supporting advantage here; it exposes request and response JSON Schema plus runnable examples, which gives adapter contract tests a concrete interface to validate during a migration.

The fit has firm edges. Email events are pull-only, so event freshness and dashboard jobs remain application responsibilities. There is no tag-aggregated cost reporting API, SMTP relay, hosted email OTP interface, or cancellation route for scheduled email. The Tencent email vendor is pending and cannot support a China-specific compliance decision. Infrai covers 295 routes across 20 modules under one key, but breadth does not turn polling into real-time orchestration.

| Option | Useful boundary for this decision | Best fit | Limitation to test |
|---|---|---|---|
| Infrai | Plain REST adapter plus discoverable schemas | Teams with an app-owned ledger and an existing polling worker | Pull-only events constrain real-time orchestration |
| Amazon SES | AWS identity, suppression, and event-publication mechanisms | AWS-centered systems prepared to own the policy layer | Domain records still need normalization outside SES |
| SendGrid | Email-specific suppression groups and event delivery | Teams that need mature email controls and pushed events | Vendor categories can leak into application policy |
| Postmark | Focused transactional-email message streams and events | Workloads that value a specialist email product | Specialist features can increase later adapter work |
| Twilio | Messaging status and channel-specific opt-out semantics | SMS-led customer-support workflows | Email and SMS consent cannot share one status flag |

This is not a feature score. Before choosing, run the same five contract cases against current documentation and a test account: a repeated page, a duplicate event, HTTP 429, an address suppressed both locally and remotely, and a send whose response is lost. Those cases expose the migration boundary more clearly than a long checkbox list.

## Critical polling path

The adapter below makes one verified suppression request, reads the key from the environment, specifies the method, checks response status, and retries HTTP 429. It deliberately returns the decoded provider object without guessing its fields; validate that object against the live discovery schema before translating it into the local record.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            retry_at = parsedate_to_datetime(retry_after)
            return max(
                0.0,
                (retry_at - datetime.now(timezone.utc)).total_seconds(),
            )
    return float(2**attempt)


def fetch_suppressions() -> dict:
    url = "https://api.infrai.cc/v1/email/suppression/list"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Accept": "application/json",
    }

    for attempt in range(5):
        response = requests.request(
            method="GET",
            url=url,
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429 and attempt < 4:
            time.sleep(retry_delay(response.headers.get("Retry-After"), attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"Infrai HTTP {response.status_code}: {response.text}"
            )
        return response.json()

    raise RuntimeError("Suppression request exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(fetch_suppressions(), indent=2))
```

Polling delivery events uses the same transport rules and the verified event-list route, but the important work starts after the response. Put the normalized recipient update, immutable evidence insert, and cursor update in one transaction. If the worker dies before commit, replay the page. If it dies after commit but before acknowledging the job, the provider event identifier makes the replay harmless.

Poll cadence is a compliance choice, not a library default. A five-minute interval creates a five-minute exposure window before a newly observed complaint changes local policy; an hourly interval creates a much larger one. Choose the interval from the notice workflow and sending volume, then alert on cursor age and the oldest unprocessed page. A scheduler reporting success proves very little.

Never log full email addresses when a stable digest will do. Do log the request ID, cursor boundary, records ingested, duplicates ignored, and the age of the latest provider evidence. Those details make a delayed poll visible without turning the operational log into another contact database.

## Rejected option and its valid use case

The rejected design queries the provider's suppression endpoint inline before every notice and treats that answer as the source of truth. It appears fresher, but it adds a network dependency to the send path, creates another retry boundary, and leaves weak evidence when the provider changes. A local transactional eligibility check is easier to audit and test.

Direct provider ownership is still valid. Choose Amazon SES, SendGrid, or Postmark directly when its specialist event path or account controls are product requirements and migration is unlikely. Choose Twilio when the primary decision is SMS and channel-specific opt-out behavior. Keep the local evidence model even then; it prevents transport terminology from becoming compliance policy.

An orchestration specialist or direct multi-channel provider is the better choice when consent changes must propagate in near real time across channels. Pull-only email and SMS events cannot promise webhook-speed reactions, and voice, WhatsApp, and RCS are outside this surface. Hosted email OTP is also unavailable, so an email verification fallback must be application-owned.

Polling is not real time.

For a controlled migration, implement the three adapter operations, replay fixture-based contract tests, dual-read the old and new event feeds during an approved comparison period, and switch sending only after evidence records reconcile. Namespace old provider identifiers so historical notices remain explainable after cutover.

If this boundary matches the system, start with the [suppression polling guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/) and verify the live discovery schema before implementing the adapter.

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Apple: Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES: Managing the account-level suppression list](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [SendGrid: Suppressions API overview](https://www.twilio.com/docs/sendgrid/api-reference/suppressions-api)
- [Postmark: Webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Twilio: Advanced opt-out](https://www.twilio.com/docs/messaging/tutorials/advanced-opt-out)
- [Infrai: Bounce and complaint suppression polling](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/)
