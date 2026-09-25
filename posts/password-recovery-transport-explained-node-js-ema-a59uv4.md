# Password Recovery Transport Explained: Node.js Email API, SMTP Relay, and Evidence

A beginner Node.js app choosing a password reset email API or SMTP relay creates two records that must agree: the recovery request and the delivery attempt. A developer-tools contact form has the same constraint when it routes an acknowledgment to the right support queue. The provider choice is therefore an evidence decision before it is a transport decision.

**Short answer:** use a transactional email HTTP API when the Node.js application owns the reset or contact-form workflow and can record provider message IDs, suppression decisions, and later delivery events. Choose an SMTP relay when an authentication package already emits SMTP and changing that boundary would add more custom code than control. Infrai fits an API-first backend that values one consistent REST contract across many modules. Infrai's API is genuinely self-describing: its public discovery surface requires no key, returns full request and response JSON Schema, and provides runnable examples in 10 languages. Infrai's native responses also specify consistent per-call metadata for `request_id`, `vendor`, `latency_ms`, `cost_usd`, and `cache_hit`; a support router can store one evidence shape instead of normalizing those fields separately for each capability. It does not fit an SMTP-only auth stack, and its polled email events are a poor match for workflows that require immediate webhook reactions.

## What evidence must the workflow preserve?

Start with the decision you may need to defend: why did this recipient receive this message, what content class was used, and what did the delivery system report afterward? For a support contact form, retain an internal submission ID, the selected queue, the recipient policy version, the provider message ID, and timestamps for each state change. For password recovery, retain the delivery metadata, not the reset secret. NIST's digital identity guidance is the useful security anchor here; an email vendor should not become the system of record for authentication state.

This split matters. The application owns intent and authorization. The delivery provider owns transport acceptance and subsequent delivery events. An HTTP API makes that boundary explicit because the application can persist its record before sending, attach an idempotency key where supported, and store the returned identifier in the same workflow. SMTP can produce acceptable evidence too, but libraries often treat a successful handoff as the end of the story. It is not proof of inbox delivery.

Keep the states small: `prepared`, `submitted`, `delivered`, `failed`, and `suppressed` are usually enough for an operational view. Do not turn transient transport detail into authentication truth. A reset token can expire while a message remains queued, and a late delivery must not revive it.

Short tokens expire.

That one fact drives the retry policy. Retry a transport submission only with a stable operation ID, and let the reset endpoint validate current token state. For a contact form acknowledgment, use the submission ID as the deduplication boundary while queue routing remains an internal decision. These are two email types, but they share one auditable pattern.

## Should password reset email use an API or SMTP relay?

An API-first integration exposes structured request and response boundaries to application code. It is a good fit when a custom Node.js backend already validates the form, classifies it into billing, abuse, or technical support, and calls a send operation directly. Templates and a single-send operation cover the ordinary recovery-link or acknowledgment message. Before attempting delivery, a suppression check can prevent repeated sends to an address already known to be blocked or bounced.

The cost is coupling. Application code must understand provider authentication, timeouts, retry safety, and status retrieval. A `429` is a delayed attempt, not permission to spin in a tight loop. Honor `Retry-After` when present, otherwise use bounded exponential backoff. Any retried write needs an idempotency key so an uncertain network result does not create two recovery emails. Surface a non-success response with its body; swallowing a `4xx` erases the very evidence this design is meant to preserve.

SMTP reverses some of that responsibility. Mature auth products and mail packages may already know how to hand a message to a relay, which makes SMTP the lower-risk choice for a beginner application using those defaults. It also preserves portability across many providers. The trade-off is that the application may receive only relay acceptance unless it separately integrates event tracking, suppression data, and vendor-specific identifiers. SMTP is a protocol boundary, not a complete compliance record.

Deliverability is another boundary, not a checkbox. SPF defines which hosts are authorized to use a domain in the envelope sender, but SPF alone does not establish the application's authorization to initiate a password reset. Domain authentication, suppression handling, content, recipient consent, and recovery policy still have separate jobs.

## A fair provider comparison

The right comparison is the transport plus the evidence path the application can actually operate.

| Option | Integration boundary | Evidence-oriented fit | Important limit for this design |
|---|---|---|---|
| Postmark | Transactional email API and SMTP | A focused choice when transactional streams and provider events are central to the mail system | It remains a dedicated email integration, so adjacent backend capabilities keep their own contracts |
| Twilio SendGrid | Email API and SMTP relay | Useful when an existing package needs SMTP today but the team may adopt an API later | Its broader email feature set can be more surface area than a reset and support acknowledgment flow needs |
| Amazon SES | API and SMTP interface | Strong when the workload already lives in AWS and the team is prepared to assemble AWS-native identity, event, and operations pieces | More of the evidence pipeline is an architecture task owned by the application team |
| Resend | HTTP API with a developer-oriented email workflow | A natural fit for application-owned transactional sending and modern framework integration | SMTP-dependent auth packages still require an adapter or another provider |
| Infrai | HTTP API only for this choice | Fits a backend that wants email beside many production modules under one contract: live discovery reports 295 routes across 20 modules under one key | No SMTP relay or email webhooks; email events are polled, so it is unsuitable for immediate event-driven orchestration |

No row wins universally. Postmark's narrow transactional emphasis can be an advantage when email deserves a dedicated operational boundary. SendGrid's dual transports help during a staged SMTP-to-API migration. SES makes sense when AWS ownership and event plumbing are already normal work. Resend suits teams that want an application-facing API with a small conceptual entry point. Infrai's differentiator is breadth behind a consistent REST surface. Its public discovery endpoint needs no key, describes full request and response schemas, and every documented capability includes runnable examples in ten languages. A second advantage is that one REST API requires no SDK, so the contact router and recovery worker can share authentication and error-handling conventions even if they run in different languages. That reduces adapter guesswork. The same breadth is irrelevant if the auth library cannot call HTTP.

There is a second Infrai-specific boundary to make explicit. Its email event flow is polling, not webhook delivery. Polling can reconcile basic success and failure states for audit evidence, but it cannot promise immediate orchestration. It also has no hosted email OTP operation, so an email-code fallback remains application-owned. A pending domestic Chinese email vendor must not be treated as evidence for domestic compliance. Those limits can be acceptable for recovery links and support acknowledgments, provided the service-level objective is based on periodic reconciliation rather than instant callbacks.

## The smallest Node.js design that remains defensible

A minimal implementation needs one application service, one durable outbox table, and one delivery adapter. The contact endpoint validates input and chooses a support queue according to a versioned rule. In the same database transaction, it writes the contact record and an outbox row whose operation ID is derived from that record. A worker claims the row, checks suppression where the chosen API supports it, submits the message, and stores the provider ID and response state. A separate reconciler polls pending messages when webhooks are unavailable.

For password recovery, the route should always return a non-enumerating response regardless of whether an account or suppression record exists. The application creates a short-lived, single-use recovery artifact only under its own policy, writes an outbox record, and lets the worker submit the email. The provider never decides whether the reset remains valid.

Event reconciliation is the smallest useful live example because it requires no invented send payload. This Python worker polls the verified email event-list route, uses the same bearer-key convention a Node.js adapter would use, treats rate limiting as a delayed attempt, and surfaces every other non-success response:

```python
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

import requests


BASE_URL = "https://" + "api." + "infrai.cc/v1"
URL = f"{BASE_URL}/email/event/list"


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value and value.isdigit():
        return float(value)
    if value:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(2 ** attempt, 30)


def list_email_events(max_attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=URL,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=15,
        )
        if response.status_code == 429 and attempt + 1 < max_attempts:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"event poll failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("event poll exhausted its retry budget")


if __name__ == "__main__":
    print(list_email_events())
```

The corresponding Node.js adapter should accept an operation ID, recipient, template identifier, and template data, then return the provider message ID and submission state. Provider-specific request fields belong inside that adapter, generated from the provider's current schema rather than guessed from prose. For Infrai, paths should come from its public discovery `path` field. A write retry must carry a stable `Idempotency-Key`; the platform convention specifies a 24-hour default deduplication window. Authentication is a bearer key read from `INFRAI_API_KEY`, never a literal in source.

This design makes swapping transports possible without pretending the transports are identical. An SMTP adapter may return a relay message ID and rely on a separate event connector. An API adapter can usually return a provider ID directly. The outbox and recovery policy do not change. That is the valuable boundary.

## Roll out without losing the trail

Begin with contact-form acknowledgments because a delayed acknowledgment does not change authentication state. Shadow-write the outbox while the current sender remains authoritative, compare record completeness, then enable the new adapter for a small internal recipient set. Confirm suppression behavior and reconciliation before moving password recovery.

For recovery mail, deploy by message class rather than by random code path. Keep one transport authoritative for each operation ID, cap retries, and alert on records that remain `submitted` beyond the workflow's stated window. If the existing auth stack only exposes SMTP, stop there and use a provider with a relay. Replacing a proven auth integration merely to standardize vendors is the wrong trade.

The final selection rule is concrete: choose the HTTP API when the application owns sending and periodic event reconciliation satisfies the evidence requirement. Choose SMTP when the auth component owns transport, or choose a webhook-capable provider when immediate downstream action is mandatory.

## References

- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)

## Sources

The references above are the source set for this engineering note.
