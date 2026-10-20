# Auditable Healthtech Access — 3 Boundaries for SMS OTP, Authenticator Apps, Email Codes

Use an authenticator-generated one-time password for routine login, keep a signed email link for signup address verification, and treat SMS as a constrained recovery or enrollment channel rather than interchangeable proof. The decision rule is evidence first: choose the factor whose verifier, enrollment, retry, and recovery events can be explained later without retaining the secret or sensitive message content.

**TL;DR:** for a US/EU healthtech SaaS, the simplest control to audit is usually a server-verified TOTP factor after enrollment. Email remains useful for the original signup link, but an email code often shares the same compromised endpoint as password recovery. SMS can cover users who cannot enroll an app, although number reassignment, carrier delivery, roaming, and SIM-swap risk require tighter recovery controls. None of the three is phishing-resistant; WebAuthn is the relevant next comparison when that property is required.

## Should healthtech login use an authenticator app, SMS OTP, or email code?

An audit trail should prove what the service decided, not reveal what the user received. For each challenge, record a random correlation ID, account ID, factor class, policy version, issuance time, expiry time, attempt count, outcome, and a coarse failure reason. Record enrollment and factor replacement as separate security events. Never log a TOTP seed, email link token, SMS code, full phone number, or full email address.

The invariants are small enough to state plainly:

1. A challenge is single-use and expires on a server-enforced clock.
2. Verification has a strict attempt limit and a rate limit scoped beyond one IP address.
3. Enrollment and replacement require evidence stronger than possession of the new factor alone.
4. Authentication logs are append-oriented, access-controlled, and covered by a documented retention policy.
5. Recovery cannot silently reduce assurance below the login policy it bypasses.

That last rule catches a common design error. A carefully configured authenticator factor adds little if support can replace it after checking only inbox access. Recovery is part of authentication, not an administrative footnote.

For healthtech, minimize the event payload. The HIPAA Security Rule's technical safeguards include access control, audit controls, integrity, and person or entity authentication requirements. GDPR data minimization and storage limitation point in the same operational direction: retain the evidence needed for the stated purpose, with a documented lifetime, rather than keeping every delivery artifact indefinitely. Whether a particular service is regulated depends on its role and data processing; counsel and the security owner must map the system to the applicable obligations.

## Invariants and failure boundaries

The boundary for an authenticator app is enrollment. TOTP, standardized in RFC 6238, derives short-lived values from a shared secret and time. It continues to work when the handset has no mobile service, but clock skew and loss of the enrolled device must be handled. The server stores a protected seed or equivalent verification material, so seed access is security-critical. A successful six-digit entry proves knowledge of the current value; it does not prove that the user saw the genuine site.

Email's boundary is the mailbox and its surrounding recovery process. A signed, one-use signup link can verify control of an address while preserving useful context in the browser. After dealing with spam filters, rate limits, and OTP delivery gaps, I treat queue acceptance as a transport event, never as proof of possession. Delivery is less deterministic than local TOTP verification: filtering, forwarding, delayed queues, and mailbox compromise sit outside the application's transaction boundary. DKIM provides a domain-level message-signing mechanism; it does not turn an email link into a phishing-resistant authenticator. Open tracking is also poor compliance evidence. Apple Mail Privacy Protection can download remote content without the recipient actively opening the message, so a tracking pixel must not be recorded as verification or consent.

SMS crosses still more operators. NIST SP 800-63B treats use of the public switched telephone network for out-of-band authentication as restricted and tells verifiers to consider risk indicators such as device swap, SIM change, and number porting. Delivery receipts can describe carrier handling, but they do not establish that the intended person read a code. Keep message content generic; a lock-screen preview should not disclose a diagnosis, clinic, medication, or even that an account exists.

The practical comparison follows from those boundaries:

| Path | Evidence the service can own | External failure boundary | Operational burden | Appropriate role |
| --- | --- | --- | --- | --- |
| Authenticator TOTP | Enrollment, protected seed version, verification result, clock window, attempts | User device clock, copied seed, device loss | Seed protection, recovery codes, replacement ceremony | Routine second factor where phishing resistance is not mandated |
| Email link or code | Token issue, message submission, token redemption, attempts | Mail routing, filtering, forwarding, mailbox recovery | Domain authentication, bounce handling, anti-enumeration, link expiry | Signup address verification and carefully governed fallback |
| SMS OTP | Challenge issue, gateway submission, verification result, attempts | Carrier routing, roaming, number reassignment, SIM or port events | Consent records, regional rules, abuse controls, number-change recovery | Accessibility or recovery path after a documented risk decision |

There is no honest universal "cheapest" answer. Per-message charges are only one input; support contacts, fraud review, seed custody, recovery, localization, and compliance evidence all move the total. Measure those costs against the same account-risk tier and recovery policy. Otherwise the comparison is accounting theater.

## Critical path: separate delivery from proof

The verifier should make one policy decision and emit one normalized audit event regardless of channel. Delivery adapters can fail or retry without redefining success. For an email signup link, success means token redemption after verification, not queue acceptance, a delivery status, or a remote image request.

The following Python sketch shows the boundary. The token store must persist only a keyed digest, enforce expiry and attempt limits atomically, and mark the challenge consumed in the same transaction as the verified event. `audit.append` receives metadata, never the presented token.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


class ChallengeStore(Protocol):
    def consume_if_valid(
        self, challenge_id: str, presented_token: str, now: datetime
    ) -> bool: ...


class AuditLog(Protocol):
    def append(self, event: dict[str, object]) -> None: ...


@dataclass(frozen=True)
class VerificationRequest:
    challenge_id: str
    account_id: str
    factor: str
    policy_version: str


def verify_signup_link(
    request: VerificationRequest,
    presented_token: str,
    store: ChallengeStore,
    audit: AuditLog,
) -> bool:
    now = datetime.now(timezone.utc)
    accepted = store.consume_if_valid(request.challenge_id, presented_token, now)

    audit.append(
        {
            "event": "signup_address_verification",
            "challenge_id": request.challenge_id,
            "account_id": request.account_id,
            "factor": request.factor,
            "policy_version": request.policy_version,
            "occurred_at": now.isoformat(),
            "outcome": "accepted" if accepted else "rejected",
        }
    )
    return accepted
```

Short-lived secrets invite race conditions. Two concurrent requests can both observe an unused token unless consumption is atomic; a read followed by a later update is insufficient. The same store operation should increment failed attempts without exposing whether an account exists. At the edge, return a uniform response for unknown and known addresses, while internal reason codes remain coarse and access-controlled.

Deployment deserves equal care. Rotate signing and encryption keys with version identifiers, test old-key verification during the planned overlap, and alert on sudden changes in issuance-to-redemption time, rejection rate, retries, and factor replacement. Split those metrics by region and factor, but use aggregation thresholds that avoid turning dashboards into user activity records. Synthetic tests can exercise email authentication and delivery, while TOTP test vectors from RFC 6238 check verifier correctness without production seeds.

## What was rejected, and when is it still valid?

The rejected design used email codes as the default second factor for every login because it appeared to require no enrollment. It was easy to explain and easy to ship. It also coupled password recovery and the second factor to one mailbox for many users, made login availability depend on mail delivery, and produced ambiguous signals when privacy features fetched message content. Those are poor boundaries for the primary login path when compliance evidence is the deciding axis.

Email codes still have a valid use case: low-risk accounts whose password and mailbox controls are independently assessed, especially when broad accessibility matters and the documented assurance target permits it. A signup verification link is stronger UX than asking a person to transcribe a code, provided the application binds it to a short-lived, single-use server challenge and handles cross-device redemption intentionally.

SMS also remains valid for a defined population that cannot use an authenticator app, or as one input to a recovery ceremony with additional checks. Do not call a successful SMS delivery report identity proof. It isn't.

The resulting architecture is deliberately uneven: local TOTP verification for routine second-factor checks, an email link for address possession during signup, and controlled fallbacks for users who cannot complete enrollment. The compliance evidence stays consistent across them because issuance, verification, replacement, recovery, and policy version are modeled as first-class events. Channel choice can then change without rewriting what "verified" means.

## References

- RFC 6238, TOTP: Time-Based One-Time Password Algorithm: https://datatracker.ietf.org/doc/html/rfc6238
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- NIST SP 800-63B, Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- W3C, Web Authentication Level 3: https://www.w3.org/TR/webauthn-3/
- Apple, Use Mail Privacy Protection: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- U.S. Department of Health and Human Services, HIPAA Security Rule technical safeguards: https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- Regulation (EU) 2016/679, Article 5 principles: https://eur-lex.europa.eu/eli/reg/2016/679/oj
