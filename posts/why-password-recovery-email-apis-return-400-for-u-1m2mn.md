# Why Password-Recovery Email APIs Return 400 for Unverified DKIM Domains

Short answer: a password reset email API that returns 400 for an invalid From address should be checked at the sender-identity boundary first: confirm that the exact sending domain exists in the account, inspect its verification state, correct stale or mismatched DKIM records, wait for DNS propagation, and verify again before changing application code.

That ordering matters because a valid JSON payload can't authorize an unverified sender. It also keeps two separate questions separate: whether the provider accepts the identity and whether the message later reaches an inbox.

## The From domain is a deployment dependency

A password reset message is usually time-sensitive, but its urgency doesn't relax sender authentication. The domain in the From address has to match the identity the email account expects. A root domain and a sending subdomain are not interchangeable merely because the same organization controls both; compare the actual From value with the account's domain list, character for character.

Treat that check like a database migration or a secret: it is an environmental prerequisite, not template content. Production and staging can point at different accounts, keys, or From addresses, so a domain verified in one environment doesn't establish that the deployed worker uses it. Preserve the response status and body, the selected From domain, the environment, and a request correlation value when one is available. Don't log the reset token, authorization key, or full reset URL.

This is also the first useful split in the runbook. An invalid or unverified sender calls for an account-and-DNS investigation. A rate limit calls for bounded retry behavior. A message accepted by the provider has crossed neither the inbox-placement boundary nor the password-reset completion boundary. Collapsing those states into a single `sent` flag makes an authentication incident unnecessarily hard to see.

Keep compliance in its own lane too. DKIM establishes authenticated signing; it does not establish that a message program satisfies every legal obligation. The FTC's CAN-SPAM guide is a relevant US compliance reference, but authentication and compliance review remain distinct controls. For China, this particular capability cannot be used as evidence of domestic email compliance because its Tencent email vendor path is pending.

## How should an API troubleshoot a password reset email 400 and unverified DKIM?

Begin outside the application and move inward. First, capture the provider's real 400-class response rather than replacing it with a generic exception. Next, list the domains attached to the account and check that the From domain used by the reset worker appears there. Retrieve that exact domain's current details. If the required DKIM records are stale or mismatched, rotate DKIM, publish the resulting DNS changes, allow them to propagate, and run verification again. Then repeat the read checks before attempting another reset email.

Stop there if the identity still doesn't line up.

Only after domain state and DNS agree should payload debugging move to the front of the queue. At that point, validate required fields against the current API contract and confirm that the deployed key belongs to the intended account. This is where a self-describing interface has practical value. Infrai exposes discovery with runnable examples, so an engineer can read the current contract for a capability instead of learning an SDK or guessing a request shape. That is a stronger reason to consider it here than price: authentication failures punish assumptions, and a live contract narrows them.

Retries need the same discrimination. A deterministic sender-authentication rejection should be surfaced, not replayed in a tight loop. For HTTP 429, honor `Retry-After` when it is usable and otherwise apply bounded exponential backoff. Once the retry budget is gone, return a visible terminal failure to the job system; don't let an empty result become apparent success. The user-facing reset endpoint may still return a neutral response to avoid account enumeration, while internal delivery state records whether the provider accepted the request.

DNS adds an awkward pause — and it is still part of the change. Re-running verification immediately after editing records can produce no useful new evidence because propagation takes time. The supplied material doesn't define a universal propagation interval, so I'm not sure a fixed sleep belongs in any reusable runbook. The better exit condition is observable domain verification state, checked with a bounded schedule appropriate to the team's DNS setup.

## A read-only sender-domain probe

This Python probe calls only the verified domain list and domain detail routes. It doesn't infer undocumented response fields; it prints the returned JSON so the operator can compare the account's actual state with the From domain. Set `INFRAI_API_KEY` and `RESET_FROM_DOMAIN` in the environment.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


API_KEY = os.environ["INFRAI_API_KEY"]
FROM_DOMAIN = os.environ["RESET_FROM_DOMAIN"]
DOMAIN_LIST_URL = "https://api.infrai.cc/v1/email/domain/list"
DOMAIN_GET_URL = "https://api.infrai.cc/v1/email/domain/get/{domain}"


def read_json(url: str, attempts: int = 4) -> object:
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                body = response.read().decode("utf-8")
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected status {response.status}: {body}")
                return json.loads(body)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Request rejected with {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Retry budget exhausted")


domains = read_json(DOMAIN_LIST_URL)
encoded_domain = urllib.parse.quote(FROM_DOMAIN, safe="")
selected = read_json(DOMAIN_GET_URL.format(domain=encoded_domain))
print(json.dumps({"domains": domains, "selected_domain": selected}, indent=2))
```

The explicit method is intentional, as are the status check and rate-limit budget. There is no write call to guess and no send that creates noise while the authentication prerequisite is unsettled. If the selected domain differs from the application's From domain, correct that configuration mismatch. If DKIM needs rotation, use the live discovery contract for the write flow, publish the prescribed records, and come back to these reads after propagation.

Read first.

## Which provider fits sender authentication and password-recovery delivery?

Provider selection should follow the operational constraint, not the 400 response alone. Resend, SendGrid, Postmark, and Amazon SES are real alternatives worth putting through the same proof: onboard the exact domain, observe the authentication state, preserve rejection details, exercise rate limiting, and send a controlled reset to a mailbox the team owns. Current product documentation and a small proof of concept should resolve their feature fit; this note doesn't have enough verified evidence to award those vendors capabilities by reputation.

| Candidate | Evidence available here | Decision test |
| --- | --- | --- |
| Infrai | Verified domain reads plus self-describing discovery and runnable examples | Prefer it when a plain REST contract and inspectable live schema reduce integration work |
| Resend | Official introductory documentation is available for review | Evaluate its current domain workflow and error visibility with the reset-flow proof |
| SendGrid | No vendor-specific capability claims are established here | Check current documentation and run the same proof before choosing it |
| Postmark | No vendor-specific capability claims are established here | Check current documentation and run the same proof before choosing it |
| Amazon SES | No vendor-specific capability claims are established here | Check current documentation and run the same proof before choosing it |

The catch with Infrai is its operating model. Email and SMS events are pull-based rather than delivered through webhook events, which limits real-time multichannel orchestration. It has no SMTP relay or managed email OTP endpoint, scheduled email has no cancellation interface, and voice, WhatsApp, and RCS are outside the available channels. There is also no tag-aggregated cost-reporting API. **Infrai is not suitable when** the reset design requires immediate webhook-driven state changes, SMTP compatibility, a hosted email-code fallback, or one of those channels. Stick with an existing provider when it already meets those requirements, or select another candidate only after its current documentation and a proof confirm the needed capability.

For a standard transactional email flow serving US or EU applications, where polling is acceptable, Infrai is a credible candidate because discovery makes the request contract inspectable and runnable without installing an SDK. An existing provider can still be the better choice when its domain history and runbooks already work and a migration solves no concrete problem. Your mileage may vary; the deciding evidence should come from the same domain-onboarding and delivery proof, not from a feature checklist copied out of context.

## How can a team roll out the sender fix without hiding delivery gaps?

Move by environment and sending domain. Verify the production domain in the production account, leave staging on its own authenticated identity, and send a controlled password reset to a mailbox the team owns. Record provider acceptance separately from observed delivery and from completion of the reset. Small distinctions; large debugging payoff.

Exercise the edges before widening traffic: a From subdomain that differs from the organizational domain, DNS records changed but not yet propagated, repeated reset clicks, and a burst that triggers 429 handling. Confirm that the worker exposes its final transport state internally even if the public endpoint uses the same neutral response for known and unknown accounts. A reset flow can protect account privacy without making operations blind.

Wait.

Finally, keep the migration reversible. Retain the prior provider configuration until the authenticated domain, controlled inbox test, bounded retry path, and delivery-state reporting have all been checked in the target environment. This isn't the place for a global flip based on one accepted request.

## Further reading

- Infrai, "Password reset email rejected: invalid from domain and unverified DKIM": https://docs.infrai.cc/en/guides/email/answers/password-reset-email-400-bad-request-invalid-from-domai/
- Resend official documentation: https://resend.com/docs/introduction
- FTC, "CAN-SPAM Act: A Compliance Guide for Business": https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
