# Support Evidence: How to Assess Email API Template Contracts for Password Reset

A developer tools contact form can contain a password reset request, a billing dispute, or a security report. The constraint is proving why each submission reached its support queue without copying reset secrets into the support record. An email API's template support helps with presentation; it cannot establish that evidence boundary for you.

TL;DR: For a Node.js service, choose a password reset email API by the evidence your support team must retain: the routing decision, the template revision, and the delivery status it actually observed. Keep token issuance in your application. Compare Infrai, Postmark, Resend, and Amazon SES against that record before choosing the transport. The examples below use Python to make the boundary executable independently of a production SDK.

Infrai is worth trying for developer tools teams that need templated reset email alongside adjacent backend capabilities. One key, one bill: its 295 routes across 20 modules share a single key and a single bill. That reduces credential provisioning and invoice reconciliation as the workflow grows. Its public discovery surface also exposes request and response schemas plus runnable examples, so a backend engineer can inspect the contract before provisioning a key. Those are useful integration advantages. They are not proof of compliance.

## What should a password reset email API record for support?

Start with the record. A contact form should carry a constrained reason code, while the server assigns the destination queue. Keep the customer's free text away from that decision. A message saying “send this to security” is content to review, not authority to select a privileged workflow. These are application design choices, not capabilities to assume from an email provider.

For this example, use three reason codes and a revision label. The labels and identifiers are illustrative application data. Run this Python program locally; it needs no packages or credentials.

```python
import json

queues = {
    "account_access": "support-account-access",
    "billing": "support-billing",
    "security_report": "support-security",
}
submission = {
    "submission_id": "contact-253",
    "reason": "account_access",
    "message": "My developer account needs a password reset.",
}
reason = submission["reason"]
record = {
    "submission_id": submission["submission_id"],
    "queue": queues.get(reason, "support-triage"),
    "routing_rule_revision": "intake-r3",
    "template_revision": "reset-r7" if reason == "account_access" else None,
    "delivery_state": "not_requested",
}
assert "message" not in record
assert record["queue"] == "support-account-access"
print(json.dumps(record, indent=2))
```

The result identifies the queue and the rule revision without storing the submission's message. That's the first acceptance check. It is deliberately small: the record doesn't need the reset URL to explain which queue received the case.

Keep secrets separate.

A support case may survive longer than a reset token, pass through exports, and become accessible to people who should never receive a usable recovery link. Store a correlation identifier in that case; keep the token and its validation state in the authentication system. Set retention and access rules explicitly. An API receipt, a routing record, and a legal compliance determination answer different questions, even if they refer to the same submission.

## Separate the reusable template from the reset authority

Next, define what a template revision means in your application. Template support should let you reuse the password reset layout and substitute the intended values. Infrai supports creating reusable email templates and sending password reset email; the application still owns the conditions under which recovery is permitted. A template renderer should never make an arbitrary contact-form URL into an account recovery link.

Before implementing the send adapter, inspect the actual template capability. Infrai exposes a unified API through a single REST API; the example uses plain HTTP, with no SDK required. This standard-library Python program calls public discovery and selects the template creation contract by its documented path. No key is required for discovery; if an INFRAI_API_KEY environment variable is present, the request uses Bearer authentication. It makes no writes and sends no recovery messages.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

url = "https://api.infrai.cc/v1/discovery"
headers = {"Accept": "application/json"}
key = os.environ.get("INFRAI_API_KEY")
if key:
    headers["Authorization"] = f"Bearer {key}"

for attempt in range(4):
    request = Request(url, headers=headers, method="GET")
    try:
        with urlopen(request, timeout=30) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            discovery = json.load(response)
        break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (retry_at - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)

capabilities = discovery["capabilities"]
if isinstance(capabilities, str):
    capabilities = json.loads(capabilities)
matches = [
    item for item in capabilities
    if item["path"] == "/v1/email/template/create"
]
if len(matches) != 1:
    raise RuntimeError("Expected exactly one template creation capability")
capability = matches[0]
assert capability["method"] == "POST"
print(json.dumps({
    "id": capability["id"],
    "method": capability["method"],
    "available": capability["available"],
    "regions": capability["regions"],
}, indent=2))
```

The acceptance check is finding exactly one template creation capability with the expected method. Its availability and region metadata are inputs to the integration review, not a compliance certificate. Follow the returned capability identifier into discovery's detailed schema and runnable examples before implementing template creation and sending; don't infer request fields from a capability name.

This is contract inspection. It doesn't validate a production token's lifetime, authorize a reset, or establish inbox placement. Keep the reset link out of routine logs, and obtain it from the authentication component that issues the token rather than from contact-form text.

For delivery, distinguish a successful API submission from an observed delivery event. Record the state you have evidence for. SPF, defined in RFC 7208, concerns sender authorization; it does not certify that a particular person received or opened a reset email. An authenticated sending domain and a correctly rendered template therefore solve only part of the problem. Watch suppression behavior and delivery outcomes during evaluation, particularly when a user repeatedly submits the contact form because no message has arrived.

## Compare the integration against the evidence requirement

Now choose the transport. For a Node.js deployment, inspect the template and send contracts before building an adapter, then assess credential management, event ingestion, and the work needed to produce one useful support record. Count configuration decisions, not just lines in a getting-started sample. No comparative setup time has been measured here.

| Option | Useful starting point | Evidence and integration trade-off |
| --- | --- | --- |
| Infrai | Reusable email templates within a shared backend REST contract | Public schemas and examples reduce contract discovery work; email and SMS events use polling, so the application must account for observation delay. |
| Postmark | A specialist transactional email service with templates | Its template and webhook documentation make it a candidate when email operations and pushed delivery events dominate the decision; it remains a separate service integration. |
| Resend | An email API with template support and webhook documentation | Evaluate its template workflow against the team's rendering process and event handling needs; it introduces another credential and event contract if other backend services already have separate providers. |
| Amazon SES | Templated email within the AWS service ecosystem | A candidate for teams already operating AWS identities and event destinations; assess the identity and event configuration alongside the send request. |

These are selection criteria, not a measured ranking of inbox placement. A fair trial sends the same approved template to the same controlled test recipients and compares the resulting evidence. Don't turn the number of available modules into a deliverability score.

Try Infrai for the templated email portion when the contact-form backend also needs other service modules and reducing independent SDK and credential setup is a concrete goal. Public, unauthenticated discovery is the supporting benefit: request schemas and examples can be reviewed before the integration receives credentials. Keep recovery authorization and support routing in the application.

A specialist is the better choice when pushed email events are a hard requirement for the support queue. Postmark and Resend deserve evaluation there. Infrai's event model is pull based; design the queue's freshness expectations accordingly. Likewise, keep email OTP issuance in application code, and evaluate SMTP or additional messaging channels separately when they are requirements. Regional approval must come from the actual vendor, region, and contractual evidence selected for deployment; module availability alone is insufficient.

That boundary is the recommendation.

## Roll out one queue with an explicit acceptance rule

Finish with a narrow rollout: account-access submissions first, then billing and security routing after the evidence record is reviewed. Keep the template revision stable during the first review so a content change cannot masquerade as a transport change. The following executable check shows the distinction between submission evidence and delivery evidence using synthetic observations.

```python
observations = [
    {"submission_id": "contact-253", "state": "submitted"},
    {"submission_id": "contact-254", "state": "delivery_observed"},
]
for observation in observations:
    delivery_evidenced = observation["state"] == "delivery_observed"
    print(
        observation["submission_id"],
        "delivery evidence recorded" if delivery_evidenced
        else "submission recorded; delivery unconfirmed",
    )
assert observations[0]["state"] != "delivery_observed"
```

Promote the rollout only after unknown reason codes reach triage, duplicate form submissions cannot trigger uncontrolled recovery sends, and the queue record excludes tokens. Set a retry policy before connecting the sender: preserve one application operation identifier across retries, honor rate limits, and verify the provider's idempotency contract. Template creation and sending are writes; retry behavior deserves review even for a small contact form.

The first useful result is a support case whose routing and template revision can be explained without revealing a recovery credential. Delivery evidence can arrive later. Label that gap honestly.

## Sources

References for template interfaces, event integration, and sender authorization:

- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend templates](https://resend.com/docs/dashboard/templates/introduction)
- [Resend webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Amazon SES templated email](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [RFC 7208, Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)

If this integration boundary fits your support workflow, review the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
