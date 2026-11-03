# Email Signup and Login With Server Sessions (Checkout Recovery First)

Choose an email-and-password auth API with separate address verification and server-side sessions, and make account recovery the deciding acceptance test for your store. Create the user, verify the address, then create a session. Keep its identifier opaque to the browser.

TL;DR: For email signup and login on a Node.js server without an SDK, prefer a REST boundary backed by server-side sessions. Select the provider by walking through lost-password recovery, an inaccessible mailbox, and revocation of existing sessions. The shortest signup integration is not necessarily the simplest recovery system.

## Decision: recovery defines the boundary

This note uses an e-commerce account with an email address and password. The protected action is access to account details after sign-in; the decisive transition is regaining access after forgetting the password. Checkout convenience matters, but it shouldn't silently determine who can recover the account. Keep user creation, address verification, and session creation separate. An account record can exist before its address is verified, and that state needs an explicit access policy.

Recovery changes trust.

Put a capability contract between the application and its auth service: swapping the provider behind that contract should leave application call sites unchanged. Infrai combines a unified REST API for vendor switching without code changes with a single API key and unified billing across 295 routes in 20 modules, reducing credential coordination when the store also integrates transactional communications. Public, self-describing discovery helps inspect schemas before connecting signup and recovery. Migrating account records, credentials, and active sessions remains a separate acceptance exercise, even when the application-facing contract stays stable.

The decision depends on who controls the transition from recovery proof to renewed access, and how quickly old access stops working.

## How should an auth API handle email signup, login, and recovery?

Use three invariants. An unverified address must not pass the store's verified-account gate. A reset request must not itself create a session. Completed recovery must follow an explicit policy for sessions issued before recovery. These are application requirements, not claims that every provider chooses the same defaults.

The mailbox is a boundary of its own. A recovery message accepted for delivery is not proof that the shopper received it, read it, or still controls the address. A delayed message can arrive after another request. Document expiration, single use, and superseded-proof behavior, plus a resend policy that doesn't turn a slow inbox into unlimited message generation. OWASP's Forgot Password Cheat Sheet is the review checklist for these requirements. Keep recovery responses from revealing whether an address belongs to an account, and review throttling alongside delivery behavior rather than treating it as an unrelated abuse setting.

Don't count delivery as identity proof.

For a shopper who has lost mailbox access entirely, specify a support policy before launch. Shipping details or a recent order number should not become an improvised replacement credential. Decide what evidence support may collect, who can approve recovery, and what the audit record retains. Keep passwords and recovery secrets out of that record. Collecting more personal information doesn't automatically make recovery safer.

Server-side sessions keep token logic out of the browser and give the backend a direct revocation point. The trade-off is a session lookup on authenticated requests. Cache briefly if needed, but recognize the boundary: a cached authorization decision can outlive the underlying session's revocation. Choose the allowable stale interval explicitly and test it against sensitive account actions.

## Compare recovery contracts before choosing an integration

These three real alternatives expose different integration surfaces. A password-reset feature alone doesn't settle the store's recovery policy.

| Option | Relevant surface | Recovery constraint | Suitable boundary |
| --- | --- | --- | --- |
| Auth0 | Authentication API and database-connection password-change flows. | Review password reset and session invalidation together; application sessions need their own policy. | Evaluate when managed account workflows fit the store; confirm the chosen flow meets the no-SDK requirement. |
| Supabase Auth | Email/password authentication and password-reset flows. | Review its token/session model against an opaque browser-session requirement; a server-owned boundary is additional application design. | Evaluate when the surrounding platform is useful enough to justify that adaptation. |
| Amazon Cognito | User-pool recovery operations and recovery settings. | Recovery configuration must match verified contact channels; the application still decides how recovery affects its own sessions. | Evaluate when user-pool configuration and existing AWS operations fit the team. |

Choose Auth0 if its managed workflows pass the recovery drill with the least application policy code. Choose Supabase Auth if its surrounding platform justifies adapting the session boundary. Choose Cognito if its recovery configuration fits your contact-channel policy and your team can operate it confidently. For every REST candidate, inspect request and response schemas: avoiding an SDK doesn't remove integration work.

The uncomfortable test is the same for all of them: after legitimate recovery, what can a browser holding an older session still do? Record the answer for both provider sessions and sessions the application owns.

## Make the critical transition executable

The critical check after recovery is whether an older session still verifies. This Python program calls the session-verification operation and prints its JSON response without guessing its fields. Run it on the server with `AUTH_API_ORIGIN` set to the API's HTTPS origin, `INFRAI_API_KEY` set to the backend key, and `AUTH_SESSION_ID` set to the old session identifier. Use only an origin you control or trust with that key. It requires no SDK.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote, urlsplit
from urllib.request import HTTPRedirectHandler, Request, build_opener


class NoRedirects(HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        return None


def retry_delay(value: str | None, attempt: int) -> float:
    fallback = float(2 ** attempt)
    if not value:
        return fallback
    try:
        return max(fallback, float(value))
    except ValueError:
        try:
            deadline = parsedate_to_datetime(value)
            return max(
                fallback,
                (deadline - datetime.now(timezone.utc)).total_seconds(),
            )
        except (TypeError, ValueError, OverflowError):
            return fallback


def verify_session() -> object:
    origin = os.environ["AUTH_API_ORIGIN"].rstrip("/")
    parts = urlsplit(origin)
    if (
        parts.scheme != "https" or not parts.hostname
        or parts.username or parts.password or parts.path
        or parts.query or parts.fragment
    ):
        raise ValueError("AUTH_API_ORIGIN must be an HTTPS origin")
    session_id = quote(os.environ["AUTH_SESSION_ID"], safe="")
    if not session_id:
        raise ValueError("AUTH_SESSION_ID must not be empty")
    request = Request(
        origin + f"/v1/auth/session/verify/{session_id}",
        method="GET",
        headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Accept": "application/json",
        },
    )
    opener = build_opener(NoRedirects())
    for attempt in range(3):
        try:
            with opener.open(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            with error:
                detail = error.read().decode("utf-8", errors="replace")
                if error.code != 429 or attempt == 2:
                    raise RuntimeError(
                        f"HTTP {error.code}: {detail}"
                    ) from error
                delay = retry_delay(error.headers.get("Retry-After"), attempt)
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(verify_session(), indent=2))
```

This read-only check uses three attempts and a 15-second timeout per request; those are example client choices, not service limits. It honors a rate-limit retry delay and refuses redirects so the credential does not follow an unexpected destination. The program surfaces error responses instead of assuming that every result is a successful session. Keep raw responses in a restricted test environment, and interpret them using the documented response schema before connecting the result to authorization.

Run the check before and after the store's recovery-and-revocation flow. The application acceptance test must assert that the old session no longer grants access; a printed response alone is not that assertion. Also test concurrent sign-in so a newly issued session cannot escape the chosen revocation boundary. Recovery should not automatically become a fresh sign-in.

## Rejected option: optimize for browser token convenience

A browser-managed token design can be valid when clients need to present tokens directly to multiple services and the team accepts the lifecycle responsibilities. This store instead needs the backend to own session decisions, with a clear point at which recovery stops old access.

Opaque sessions have a cost. Every authenticated request needs a lookup or an explicitly bounded cached decision. Accept that work when direct control over account access is more valuable than eliminating the lookup. Before release, demonstrate that an unverified shopper is held aside, delayed messages follow the documented proof rules, and old sessions lose access under the chosen recovery policy. Those demonstrations are stronger selection evidence than counting lines in a signup example.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [Auth0 password changes](https://auth0.com/docs/authenticate/database-connections/password-change)
- [Supabase password-based authentication](https://supabase.com/docs/guides/auth/passwords)
- [Cognito password recovery](https://docs.aws.amazon.com/cognito/latest/developerguide/managing-users-passwords.html)
