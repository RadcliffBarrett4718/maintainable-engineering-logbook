# Property portal sign-in: SMS OTP rate limits, lockout windows, and replay defense

Use the part of the system you actually control — your own counters, your own challenge records — to carry the security of an SMS OTP login, and treat the carrier hop as best-effort transport that may or may not arrive. That one decision settles most of the rate limiting, retry, lockout and replay protection questions before they turn into a vendor comparison, because none of those controls can live on the far side of a network you don't own.

The system worth designing against here is a property management portal. Once a month it generates owner statements — a PDF per building, occasionally 3 MB of maintenance line items — and emails them as attachments. The same portal signs owners and on-site staff in with a six-digit code over SMS. Both jobs are delivery problems, and a US/EU SaaS tends to buy them from the same place, which is exactly why teams reason about them the same way.

They are not the same problem at all.

## Two deliveries with opposite failure behavior

The statement email has slack. If it lands forty minutes late nobody notices, and because the job is keyed by report ID, a retry after a transport error is safe. It also fails loudly: a rejecting mailbox provider returns a bounce you can parse, and a DKIM signature (RFC 6376) gives that provider a stable identity to attribute the reputation to, so the feedback loop closes over days and weeks. Deliverability work on this path is slow, measurable, and forgiving.

The login code has none of that. Someone is staring at a form. The credential is worth minutes, a re-request costs real money, and if that re-request mints a second live code it widens the guessing surface while confusing the person who already typed the first one. Worse, this path fails quietly. A carrier delivery receipt tells you a handset acknowledged the message, not that a human read it, and a code that never arrives usually looks like nothing at all in your metrics — just a login that didn't complete.

So the reliability budget for the OTP flow has to be spent before the send, in code you own, and not recovered afterward by retrying harder.

## How should an SMS OTP login flow handle send limits, retry, lockout, and replay protection?

Four decisions, in this order: who is allowed to trigger a send, how long a code stays valid, how many guesses it survives, and what consuming it means. Each one belongs in a store shared by every application instance. A countdown rendered in the browser is feedback, not a limit, and if two workers can independently approve the same request, that countdown is hiding a race rather than preventing one.

| Control | What it actually stops | Where it belongs |
|---|---|---|
| Send quota per account, per destination number, per client network | Flooding one tenant, or spraying across many accounts from one place | Shared transactional store, checked before the send |
| One pending challenge per account and purpose | Two live codes racing each other in the same inbox | Reserved in the same transaction as the send |
| Short expiry, measured from issue and never extended | A captured code staying useful long after the attempt | The challenge record |
| Attempt cap with a lockout on the challenge | Online guessing against a six-digit space | The challenge row, not the account row |
| Single-use consumption | Replay of a code read off a shared or forwarded handset | One conditional update |

NIST SP 800-63B gives you defensible numbers for two of those rows: an out-of-band secret sent to the subscriber should be valid for no more than ten minutes, and verifiers must throttle failed attempts, with 100 consecutive failures on one account as the outer bound. Six random digits is the floor for the secret itself. Those are ceilings, not targets — five minutes and five attempts is a far more common shape for a portal login, and it costs you almost nothing in completion rate.

Destination policy is the control teams skip. A property SaaS operating in the US and the EU knows which country codes its buildings are in, so the send path should refuse E.164 numbers outside that set and trip a circuit breaker when spend per destination jumps. Without it, your OTP endpoint is a payout mechanism for someone else's revenue share.

## The failure modes that show up in the first month

The one I'd bet on appearing first is the double send. A tenant double-taps the button, or a mobile browser quietly retries the POST after a dropped connection, and two challenges get created a few hundred milliseconds apart. The person types the code from the first message; your verifier compares against the newest record; they get told the code is wrong. They request another. Now you've paid three times, trained a user to distrust the flow, and burned quota that was meant to stop an attacker. The fix is boring and structural: make the send idempotent on a key of account, purpose and a client-supplied request identifier, so a repeat inside the window returns the same challenge identifier, reuses the same pending record, and does not extend its expiry. One user action, one live credential — and if the transport returns a `429`, honor `Retry-After` on that same logical operation rather than starting a new one.

Lockout is the second trap, because a naive implementation hands an attacker a denial tool. Hammering wrong codes at a building manager's account on a Friday afternoon should not lock that account until Monday. Lock the challenge, keep any account-level lock short, and make sure an alternate authenticator exists so a locked-out user has a path that doesn't route through your support queue.

Then there's the failure mode specific to this industry: recycled numbers. Tenants move out, carriers reassign the number, and a login code arrives at a handset belonging to a stranger. Re-verification on number change has to require the current authenticator and send notice to the previous contact — and the purpose binding matters here, because a code issued for login must never be accepted as authorization to change the number it was sent to.

Expiry drift is the quiet one. Delivery in some corridors is slower than your product manager expects, completion rates dip, and someone proposes raising the expiry to fifteen minutes. That trades a wider replay window for everyone against a carrier problem affecting a few. I'm not sure any single expiry value is right across every US and EU destination; the thing that would settle it is your own distribution of time-from-issue-to-verify per country code, which you won't have until you measure it.

One more, on the compliance side. Phone numbers are personal data, so they don't belong in metric labels, ordinary application logs, or error strings — aggregate by country code and policy outcome, keep the raw number in one table with a documented retention rule, and log the challenge identifier instead.

## Consuming a code has to be one atomic write

Read the challenge, compare the code, then mark it used in a second statement, and you've built a race: two submissions arriving a millisecond apart both read a pending record and both win. Doing the state transition in a single conditional update closes it. That update is the replay defense — everything else is bookkeeping around it.

```python
import hashlib
import hmac
import os

PEPPER = os.environ["OTP_PEPPER"].encode()
ATTEMPT_CAP = 5
LOCK_SECONDS = 900

CONSUME = """
UPDATE otp_challenge
   SET state = 'consumed', consumed_at = now()
 WHERE id = %(id)s
   AND state = 'pending'
   AND expires_at > now()
   AND locked_until <= now()
   AND code_digest = %(digest)s
RETURNING account_id
"""

MISS = """
UPDATE otp_challenge
   SET attempts = attempts + 1,
       locked_until = CASE WHEN attempts + 1 >= %(cap)s
                           THEN now() + make_interval(secs => %(lock)s)
                           ELSE locked_until END
 WHERE id = %(id)s AND state = 'pending'
"""


def code_digest(challenge_id: str, purpose: str, code: str) -> bytes:
    # Purpose is inside the digest, so a login code cannot be replayed
    # against the number-change endpoint even if the row is still pending.
    payload = f"{purpose}:{challenge_id}:{code}".encode()
    return hmac.new(PEPPER, payload, hashlib.sha256).digest()


def verify(cur, challenge_id: str, purpose: str, code: str) -> str | None:
    """Return the account id on success, None on every other outcome."""
    cur.execute(CONSUME, {
        "id": challenge_id,
        "digest": code_digest(challenge_id, purpose, code),
    })
    row = cur.fetchone()
    if row:
        return row[0]

    cur.execute(MISS, {"id": challenge_id, "cap": ATTEMPT_CAP, "lock": LOCK_SECONDS})
    return None
```

Two details in there are worth defending. The plaintext code is never stored — only a keyed digest, so a leaked database backup doesn't hand over live credentials, and the equality test in the `WHERE` clause reveals nothing about the code itself. And `verify` collapses wrong, expired, consumed and locked into one return value on purpose: if the endpoint reports which of those happened, it becomes an oracle that tells an attacker whether a given account has a code in flight right now.

The counters, meanwhile, sit outside this function. Attempts belong to the challenge; send quota belongs to the account, the number and the caller's network. Mixing them is how the attempt cap ends up throttling legitimate users on a shared office connection.

## Rollout: shadow mode first, enforcement second

Compute every decision before you enforce any of them. Run the quota checks, the destination policy and the attempt logic in shadow for a couple of weeks, emit the outcome as a metric, and change nothing about the response — shared networks in a building management office, staff who log in from four devices, and normal impatient re-requests all look like abuse until you've seen the actual distribution.

Then enforce for one cohort, and watch the right signal. Send-side error rates tell you about your transport; verification completion rate per destination country tells you whether the product still works. Alert on the second one.

Testing deserves more than a happy-path case. Wire a fake transport into CI and assert the awkward orderings explicitly: two verifications submitted at once against one challenge, a re-request while the first delivery is still pending, a worker restart after quota was reserved but before the send returned, a consumed code replayed thirty seconds later, and a suppressed number attached to a stale account. Each of those should map to one documented state transition and one audit record. If your team can't name the store that owns idempotency, "we retry safely" is a wish.

The trade-off worth stating plainly: SMS is a restricted authenticator in SP 800-63B terms, which obliges you to assess the risk of the channel and offer an alternative. For a portal whose owners already receive signed statement emails on a mail path you operate and monitor, an emailed code, or TOTP (RFC 6238) for staff who log in daily, is the stronger primary factor with SMS as the fallback. The catch is that email OTP moves the same delivery-reliability problem into your own pipeline, where a DMARC policy change at one large mailbox provider can quietly take out authentication as well as billing notices. Stick with SMS as the primary when your users are field staff on shared devices who won't install anything, and when a number is the only contact detail the leasing process reliably captures.

Transport choices change. The state machine shouldn't.

## Sources

- [NIST SP 800-63B Digital Identity Guidelines: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [RFC 6238: TOTP: Time-Based One-Time Password Algorithm](https://datatracker.ietf.org/doc/html/rfc6238)
- [RFC 6585: Additional HTTP Status Codes (429 Too Many Requests)](https://datatracker.ietf.org/doc/html/rfc6585)
- [ITU-T Recommendation E.164: The international public telecommunication numbering plan](https://www.itu.int/rec/T-REC-E.164)
- [Regulation (EU) 2016/679 (General Data Protection Regulation)](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
