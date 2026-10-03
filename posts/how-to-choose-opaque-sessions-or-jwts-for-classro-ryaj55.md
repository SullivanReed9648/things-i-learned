# How to Choose Opaque Sessions or JWTs for Classroom SaaS (Under Abuse)

TL;DR: Use an opaque, server-side session for a browser-based classroom SaaS unless a separately operated service must verify a bearer token without calling the application. Social sign-in does not change that rule. Keep the provider tokens out of the browser, rotate the application session after the callback, and make revocation a server-side operation. Choose a JWT only for a bounded, documented trust relationship where local verification is genuinely required.

This choice matters for an edtech product wiring Google and GitHub sign-in because bot resistance depends on control after login, not on the shape of the credential. A signed JWT can prove that an issuer made a claim; it cannot tell a quiz API that the account was disabled thirty seconds ago unless the API checks fresh state. An opaque identifier makes that check natural. Neither format stops scripted sign-ups, credential stuffing, or a stolen browser cookie by itself.

The plain-language flow is short. The browser returns from the social provider with an authorization response. The callback validates the OAuth transaction, maps the verified external identity to one internal student or instructor account, creates a fresh random application session, and sends only that session identifier in a cookie. Each request resolves the identifier to current server state. Provider access and refresh tokens, when the application actually needs them, stay encrypted on the server and follow their own lifecycle.

## What session strategy should a SaaS app use: opaque server state or JWT?

For this system, the browser should hold a meaningless random value in a `Secure`, `HttpOnly` cookie with an explicit `SameSite` policy. The session record behind it can contain an account identifier, creation and expiry times, an authentication level, and a revocation flag. Do not put a Google or GitHub access token in that cookie just because the social callback produced one; that token represents a different audience and lifecycle.

Revocation wins.

Here is a runnable session core using only the Python standard library. It deliberately begins after the provider callback has validated `state`, the authorization response, issuer metadata where applicable, and the identity claims. Those OAuth checks belong in a maintained protocol library, not in handwritten string parsing. The example focuses on the application-session boundary.

```python
from __future__ import annotations

import hashlib
import secrets
import time
from dataclasses import dataclass


@dataclass(frozen=True)
class Session:
    account_id: str
    created_at: int
    expires_at: int
    auth_level: str


class SessionStore:
    def __init__(self, pepper: bytes) -> None:
        self._pepper = pepper
        self._records: dict[str, Session] = {}

    def _key(self, raw_id: str) -> str:
        return hashlib.sha256(self._pepper + raw_id.encode()).hexdigest()

    def create(self, account_id: str, ttl_seconds: int = 3600) -> tuple[str, Session]:
        now = int(time.time())
        raw_id = secrets.token_urlsafe(32)
        session = Session(
            account_id=account_id,
            created_at=now,
            expires_at=now + ttl_seconds,
            auth_level="social_sign_in",
        )
        self._records[self._key(raw_id)] = session
        return raw_id, session

    def resolve(self, raw_id: str) -> Session | None:
        key = self._key(raw_id)
        session = self._records.get(key)
        if session is None:
            return None
        if session.expires_at <= int(time.time()):
            self._records.pop(key, None)
            return None
        return session

    def revoke(self, raw_id: str) -> None:
        self._records.pop(self._key(raw_id), None)


if __name__ == "__main__":
    store = SessionStore(pepper=secrets.token_bytes(32))
    cookie_value, created = store.create("student_2048")
    assert store.resolve(cookie_value) == created
    store.revoke(cookie_value)
    assert store.resolve(cookie_value) is None
    print("create, resolve, and revoke passed")
```

Run it as a script and it exercises creation, lookup, expiry handling, and logout. A production store must add atomic rotation, an indexed account-to-session relationship for revoke-all, durable shared storage, key management, and a defined response when that storage is unavailable. The hash means a database read does not reveal a usable cookie value; the separate pepper still needs secret storage and rotation planning.

The cookie emitted by the web adapter would look like `__Host-session=<random value>; Path=/; Secure; HttpOnly; SameSite=Lax`. The `__Host-` prefix forbids a `Domain` attribute and requires `Path=/` and `Secure`, which narrows accidental cookie scope. `SameSite` is useful defense in depth, but OWASP still recommends CSRF protection for state-changing requests. A synchronizer token or a properly designed signed double-submit token is a separate control; session format does not remove that obligation.

One easy mistake is rotating the provider credential but retaining the pre-login application session. Rotate at the authentication boundary. Otherwise, an identifier fixed before login can become an authenticated session afterward, which defeats the point of validating the social callback carefully.

OWASP calls for at least 64 bits of entropy in a session identifier. The example asks `secrets` for 32 random bytes, then URL-safe encodes them; that is deliberately generous and still keeps the cookie compact. I choose extra randomness here because it costs little, while a predictable identifier destroys every later control. Entropy is not a substitute for expiry or revocation, though. Those remain independent tests.

## Why doesn't a signed token solve bot abuse?

Because signature verification answers a narrow question: was this token signed by a trusted key and are its validated claims acceptable? Bot and abuse resistance asks different questions. How quickly is one device creating student accounts? Did twenty accounts enroll in the same class within a minute? Has a teacher revoked this account? Is the request replaying a captured session from a new context? Those decisions require fresh, cross-request state. Fast revocation is the decisive property here. With an opaque session, disabling an account or revoking all of its sessions changes the next lookup. With a self-contained JWT, an API that performs only local signature and claim checks will usually accept the token until its expiry. A denylist or introspection call can close that gap, but either reintroduces server-side state and network dependency. That can be the right design; it just removes the strongest simplicity argument for self-contained tokens. Keep rate limits keyed on more than an account. Before authentication there is no trustworthy account identifier, and after authentication a farm can spread activity across many accounts. Use privacy-reviewed signals such as transaction identifier, IP prefix, device-bound evidence where justified, and the target classroom or invitation. Apply tighter limits to expensive actions such as generating AI feedback than to reading an assignment. That protects both abuse budget and prompt cost. Challenges should be risk-triggered, not the foundation. A CAPTCHA may raise automation cost, but the server still needs idempotency, enrollment limits, replay detection, and an audit trail for decisions. Avoid treating the social provider's successful login as proof that the person is an authorized student. Authentication establishes control of an external identity; roster membership and instructor privileges remain application authorization decisions.

Format cannot carry that policy.

## Draw the boundary before choosing the format

The useful comparison is not "modern token versus old cookie." A cookie is a browser transport mechanism; either an opaque identifier or a JWT can be placed in one. The architecture question is where current authority lives and which components are allowed to evaluate it.

Name every verifier.

| Decision pressure | Opaque server session | Self-contained JWT |
|---|---|---|
| Immediate logout or account suspension | Natural on the next lookup | Needs short expiry, a denylist, introspection, or another fresh-state check |
| Browser exposure | Random identifier only | Claims are readable by the holder unless separately encrypted |
| Independent API verification | Requires shared storage or introspection | Works with trusted issuer keys and strict claim validation |
| Claim or role changes | Reflected by current server state | Can remain stale until refresh or expiry |
| Operational dependency | Session store sits on the request path | Key distribution and clock/claim rules sit on every verifier |
| Credential size | Small and stable | Grows with headers, claims, and signatures |

Pick the opaque design when the browser talks mainly to one application boundary, revocation must be prompt, and authorization changes with classroom state. Pick a JWT when independently deployed verifiers must authorize requests during a temporary loss of contact with the issuer, then specify issuer, audience, allowed algorithms, key rotation, maximum lifetime, clock-skew handling, and revocation behavior before implementation. RFC 8725 requires algorithm verification and validation of issuer and audience in the relevant contexts; merely decoding a token is not authentication.

Do not use a JWT payload as a convenient profile cache. Student names, email addresses, roster roles, and accommodation-related attributes become copied into logs, browser tools, and downstream services more easily when embedded in every request. Send the minimum stable subject needed by the verifier, and load changeable authorization data at the service that owns it.

There is a real cost trade-off. A server lookup adds a dependency and consumes storage operations. A locally verified token spends CPU and avoids that lookup, but it shifts work into key distribution, validation consistency, incident response, and constrained token lifetimes. Measure the whole path with representative concurrency. A notebook benchmark of signature verification versus a dictionary lookup does not model a shared store, cache misses, key refresh, or revoke-all during an incident.

That is the trade-off I would put in the design record: opaque sessions buy immediate central control at the price of an online state dependency; JWTs buy independent verification at the price of stale authority and distributed validation rules. The application scenario makes the former more valuable.

## Test the policy, not just the happy path

An eval harness for authentication should express invariants. Start with five: a callback cannot be replayed; login rotates the session; logout makes the old value fail; disabling an account invalidates every active session; and a role change cannot retain instructor access. Add malformed cookies, expired records, parallel refreshes, store timeouts, and duplicate social identities.

For abuse controls, replay fixed event streams rather than tuning thresholds from anecdotes. Label the expected decision for ordinary class enrollment, a burst of fake-account creation, repeated callback failures, and an instructor inviting a large cohort. Track false challenges alongside blocked activity. The cheapest rule computationally can still be expensive if it locks out an entire lecture.

The test boundary around JWTs needs adversarial cases: wrong audience, wrong issuer, expired token, future validity time, disallowed algorithm, unknown key identifier, and a valid token for the wrong token type. For opaque sessions, test identifier entropy indirectly by enforcing the generator, then focus on fixation, lookup timing, expiry, rotation races, revocation propagation, and storage failure. Never log the raw credential in either suite.

Fail it on purpose.

Keep observability low-cardinality and pseudonymous. Record a hashed internal account key, session creation and revocation outcomes, callback failure category, risk decision, and request correlation identifier. Do not record authorization codes, provider tokens, cookies, JWTs, or CSRF tokens. Alert on changes in rates and outcomes, not on a single magic threshold copied from another product.

## Ship with a revocation story

Before deployment, walk one account through sign-in, privilege change, logout, password or provider-link change where applicable, administrator suspension, and deletion. For each transition, write down which sessions survive and why. Then rehearse a signing-key compromise or session-store exposure: identify what can be revoked, how quickly verifiers learn, what evidence remains, and how users are reauthenticated.

The operational checklist is prose because the dependencies are connected. Run the session store across failure domains, set explicit timeouts, and decide whether authenticated requests fail closed when lookup is unavailable. Rotate the session identifier on login and privilege elevation. Bound idle and absolute lifetime according to the risk of the action, reauthenticate before sensitive instructor operations, and expose a revoke-all control. Keep OAuth transaction state short-lived and single-use. Review cookie scope at every reverse proxy, preserve `Secure`, and verify that caches never store authenticated responses under a shared key.

Finally, budget the authentication path like any other production feature. Count callback failures, storage latency, revocation lag, challenge rate, and expensive AI actions per trusted cohort. This is where notebook-to-production discipline pays off: the format choice becomes one testable policy among several, while the abuse controls remain visible and adjustable.

For a classroom browser application, the default remains an opaque server session. Move to self-contained JWTs only when a named verifier and a concrete availability requirement justify accepting their revocation and validation complexity. The credential format is one boundary. Authorization, abuse decisions, and incident response are the system.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc7519
- https://datatracker.ietf.org/doc/html/rfc8725
- https://datatracker.ietf.org/doc/html/rfc9700
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie
