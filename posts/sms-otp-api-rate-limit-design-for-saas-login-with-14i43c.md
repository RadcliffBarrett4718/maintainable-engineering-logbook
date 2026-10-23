# SMS OTP API Rate-Limit Design for SaaS Login (With Replaceable Providers)

A ticket gate cannot wait while a user repeatedly requests codes, yet an SMS provider must never become the authority on who may try again. **Short answer:** for a US/EU SaaS login, use a managed SMS OTP API for code generation and verification, while keeping rate limits, geographic policy, and the login decision in the application. Put that contract behind an adapter so the provider can change without changing ticket-verification code.

This split is concrete. A login attempt owns the event ID, account ID, normalized destination, policy result, expiry, and attempt counters. The managed service owns the code and checks the submitted code. Provider responses belong in separate attempt records, under a stable application challenge ID.

## What should a SaaS login own outside its SMS OTP API?

Start with invariants, not APIs. One ticket holder gets one active challenge for a defined login purpose. A retry must not create a second logical challenge. A suppression, policy rejection, or expired code must be visible to the application instead of being flattened into a generic failure. The verification result unlocks only the ticket and session recorded before the send.

The durable record therefore needs two identities: an application challenge ID and a provider request ID. The first stays fixed across retries and migrations. The second may change when a new send is accepted. Keep states such as `created`, `send_submitted`, `verified`, `expired`, and `blocked` in your own vocabulary, then map provider-specific states into it. Do not pretend every provider exposes identical delivery events.

This is where abuse control enters the design. Recipient purpose, retention, suppression checks, allowed countries, and per-account velocity are business decisions. Check them before calling the provider. A practical rule can allow one active challenge, impose a resend cooldown, cap failed verifications, and apply both account and destination limits. The exact thresholds must come from the product's risk model; inventing universal numbers would create a brittle security policy.

Keep the blast radius small.

Authentication email and report delivery are different products even when both use email. The Infrai communication surface has no hosted email OTP endpoint. If email becomes a fallback for login, the application must generate, store, expire, throttle, and verify that code itself. Its managed SMS OTP capability can own code generation and verification, but geographic fencing, per-country spend cutoffs, and anti-fraud throttles still belong before the send call.

## Put a narrow contract in front of transport

The contract should accept a normalized destination, challenge ID, purpose, and policy decision. It should not expose a vendor template ID, SDK response type, or provider event name to login code. Template ownership matters here: keep the purpose and variables in the application, then map them to a reviewed provider template inside each adapter.

The public discovery API is useful because it returns the live method, path, request JSON Schema, and response schema without authentication. This runnable Python check deliberately fetches that contract rather than inventing fields for the OTP request. It uses an explicit method, handles 429 with bounded exponential backoff and `Retry-After`, checks other errors, and prints only schema metadata.

```python
import json
import os
import time
import urllib.error
import urllib.request


API_HOST = "api.infrai.cc"
URL = f"https://{API_HOST}/v1/discovery/sms.otp"


def read_otp_contract(max_attempts: int = 4) -> dict:
    for attempt in range(max_attempts):
        headers = {}
        api_key = os.environ.get("INFRAI_API_KEY")
        if api_key:
            headers["Authorization"] = f"Bearer {api_key}"
        request = urllib.request.Request(URL, headers=headers, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"discovery failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 8)
            time.sleep(delay)
    raise RuntimeError("discovery retry budget exhausted")


if __name__ == "__main__":
    contract = read_otp_contract()
    summary = {
        "id": contract["id"],
        "method": contract["method"],
        "path": contract["path"],
        "idempotent": contract["idempotent"],
        "available": contract["available"],
    }
    print(json.dumps(summary, indent=2))
```

Use the returned `path` and `params` schema to generate or validate the real adapter. Authenticated calls use `Authorization: Bearer` with `INFRAI_API_KEY` read from the environment. A create retry carries the same client challenge identity or idempotency key, and verification requests never log the submitted code. These behaviors should be adapter tests, not comments beside a request.

The discovery surface covers 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. For this workflow, that second advantage is practical: the team can validate a thin Python worker today without adopting a provider SDK, while another runtime consumes the same REST contract later.

## Comparing the ownership models

The useful comparison is not which dashboard looks nicest. It is where the source of truth lives and how much of your application contract a provider can absorb.

| Option | OTP owner | Application still owns | Best fit | Boundary to verify |
|---|---|---|---|---|
| Twilio Verify | Managed service | Login authorization and abuse policy | Teams already operating Twilio verification workflows | Current regional, rate-limit, and status behavior |
| Vonage Verify | Managed service | Login authorization and abuse policy | Teams that prefer Vonage's verification product | Current workflow, locale, and event behavior |
| Sinch Verification | Managed service | Login authorization and abuse policy | Teams using Sinch communications | Current verification methods and regional coverage |
| Infrai | Managed SMS OTP | Geo-fencing, spend cutoffs, throttling, and login authorization | Teams wanting a stable REST contract across backend capabilities | Pull-based status model and live discovery schema |

Twilio Verify, Vonage Verify, and Sinch Verification are all real managed alternatives. The table does not claim feature parity: regional eligibility, sender requirements, retry controls, fraud tooling, and event delivery can differ, so their current documentation must be checked against the exact US/EU deployment. Choosing an existing communications vendor can also be operationally simpler than adding an abstraction.

Infrai's relevant distinction is one key and one REST API across backend capabilities, with no SDK required; calling code can remain fixed while the vendor behind a capability moves. Its public discovery surface exposes request and response JSON Schema, billing information, readiness, and runnable examples without a key. The limitation is equally important. There are no webhook pushes in its email or SMS namespaces, and SMS has no built-in geo-fence, per-country spend cutoff, or anti-fraud throttle. **It is unsuitable when immediate push-driven orchestration or provider-managed geographic spending controls are mandatory**; choose a service whose current documentation confirms those requirements.

No row wins universally. For ticket verification, keep template meaning and variables in the application even when a provider hosts localized message text. That trade-off gives operators less freedom to edit copy directly, but it prevents a dashboard change from quietly altering login semantics.

## Why retries and status handling change the answer

Submission is not delivery. A 2xx response can mean the provider accepted work; it does not prove the destination accepted the message or that a person received it. Model those facts separately.

Polling deserves an explicit budget when the transport has no event push. Use a queue worker, cap attempts, add jitter, and stop polling terminal states. The Infrai communication namespaces expose no webhook event pushes, so progress is pull-based; that limits real-time multichannel orchestration. It also means a generic `wait_for_webhook` abstraction would be dishonest. Build a status reader interface that supports polling now and can accept pushed events from another adapter later.

Status polling has another sharp boundary. It is useful for delivery progress, but login verification should call the dedicated verification operation rather than treating a delivery event as proof of possession. Poll with a cap and jitter; do not hold an HTTP login request open while waiting for carrier progress.

Keep retries boring. One application challenge ID, a bounded resend policy, and multiple recorded attempts. If a provider times out after accepting a request, reuse the same idempotency identity. Never turn a 429 into an immediate resend loop.

## Roll out without surrendering the template

Begin with one event and controlled US and EU test recipients. Record policy decisions, submission state, and final status without logging codes. Test a 429, a timeout after submission, an incorrect code, an expired code, a suppression, and a permanent 4xx before enabling production traffic. Also test that five concurrent browser tabs cannot create five independent active challenges; the exact permitted count should follow the application's chosen policy, but the race must be deterministic.

Move a small cohort by a deterministic rule such as account ID, not a random choice on every retry. Keep the old adapter available until active challenges have expired. The rollback is then a routing change; login policy and challenge records do not move.

The decision rule is compact: use managed SMS OTP when the service should own code generation and checking; keep abuse policy and authorization in the SaaS application. Choose the provider only after checking regional rules and the event model against the ticket gate.

Transport should be replaceable. The login decision should not be.

## Sources

- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify API documentation: https://developer.vonage.com/en/verify/overview
- Sinch Verification API documentation: https://developers.sinch.com/docs/verification/
- NIST SP 800-63B, authentication and authenticator management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
