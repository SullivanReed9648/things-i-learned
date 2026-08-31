# Marketplace Seller SMS OTP Login Rate Limiting Lockouts and Replay Evidence

Short answer: design a secure SMS OTP login flow by putting per-user, per-IP, and per-device abuse controls before delivery, then make retry, expiry, lockout, and one-time consumption explicit state that an audit log can reconstruct.

For a logistics marketplace, the concrete moment is a seller opening a new-order notification and signing in to arrange fulfillment. The SMS provider should deliver and verify the OTP; the marketplace should decide whether a request is allowed. That split matters because geography throttles and price-based kill switches aren't supplied by the OTP API, and a delivery receipt alone doesn't prove that the login control behaved correctly.

## How should SMS OTP login rate limiting handle retry and lockout?

Start with three independent send budgets: account, IP address, and device. Passing one budget must never compensate for failing another. A reasonable implementation also normalizes the phone number, checks suppression before delivery, and applies a country allowlist or deny rule before any provider call. The exact windows are risk-policy choices, not universal constants; tune them with an eval set containing ordinary seller retries, shared warehouse networks, device resets, and deliberate fan-out attacks.

Treat resend as another send attempt, not a free operation. A resend should invalidate the prior challenge, receive a fresh opaque challenge ID, and enter the same rate-limit counters. For verification, keep a short expiry, cap wrong guesses, and impose a temporary lockout after the cap. A correct code consumes the challenge atomically so two concurrent submissions can't both create sessions.

Order matters.

The evidence record should say which policy version ran, which dimensions allowed or denied the action, and what state transition occurred. It should not contain the plaintext OTP. Store a keyed or salted digest for comparison, minimize phone data in the event stream, and let the applicable retention policy decide how long the evidence remains available. I'm not sure a single retention period fits both US and EU deployments; counsel and the data owner need to resolve that against the actual processing purpose and jurisdiction.

## Put the abuse decision before delivery

The following Python example is intentionally provider-neutral. It is runnable with the standard library and models the control boundary that belongs in the marketplace backend. The `send_code` callback is where an adapter can first call `POST /v1/sms/suppression/check` and, only when the number isn't suppressed, call `POST /v1/sms/otp`. Those are the only provider routes this walkthrough needs to name.

In production, move challenge state, counters, and the consume operation into a transactional datastore. The in-memory stores make the transitions visible without pretending that a notebook dictionary is production coordination. The 429, 423, and 401 outcomes are useful test fixtures too — an eval harness can assert both the response and the audit event for every branch.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from hashlib import sha256
from json import dumps, loads
from os import environ
from secrets import token_hex
from time import sleep
from typing import Callable
from urllib.error import HTTPError
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class Decision:
    status: int
    reason: str
    challenge_id: str | None = None


@dataclass
class Challenge:
    user_id: str
    phone: str
    digest: str
    expires_at: datetime
    attempts_left: int
    consumed_at: datetime | None = None


class InfraiSms:
    def __init__(self) -> None:
        self.origin = environ["INFRAI_API_ORIGIN"].rstrip("/")
        self.api_key = environ["INFRAI_API_KEY"]

    def send_otp(self, payload: dict[str, object], decision_id: str) -> dict:
        body = dumps(payload).encode()
        for retry in range(4):
            request = Request(
                f"{self.origin}/v1/sms/otp",
                data=body,
                method="POST",
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json",
                    "Idempotency-Key": decision_id,
                },
            )
            try:
                with urlopen(request, timeout=10) as response:
                    if not 200 <= response.status < 300:
                        raise RuntimeError(f"unexpected status {response.status}")
                    return loads(response.read())
            except HTTPError as error:
                body_text = error.read().decode()
                if error.code != 429 or retry == 3:
                    raise RuntimeError(
                        f"OTP request rejected with {error.code}: {body_text}"
                    ) from error
                retry_after = error.headers.get("Retry-After")
                sleep(float(retry_after) if retry_after else 2**retry)
        raise RuntimeError("retry budget exhausted")


class OtpGuard:
    def __init__(
        self,
        send_code: Callable[[str, str], None],
        allowed_countries: set[str],
    ) -> None:
        self.send_code = send_code
        self.allowed_countries = allowed_countries
        self.challenges: dict[str, Challenge] = {}
        self.send_events: list[tuple[datetime, str, str, str]] = []
        self.locked_until: dict[str, datetime] = {}
        self.audit: list[dict[str, str | int]] = []

    @staticmethod
    def _digest(challenge_id: str, code: str) -> str:
        return sha256(f"{challenge_id}:{code}".encode()).hexdigest()

    def _count(self, now: datetime, dimension: str, value: str) -> int:
        cutoff = now - timedelta(minutes=10)
        return sum(
            1
            for timestamp, event_dimension, event_value, _ in self.send_events
            if timestamp >= cutoff
            and event_dimension == dimension
            and event_value == value
        )

    def _record(self, action: str, user_id: str, reason: str, status: int) -> None:
        self.audit.append(
            {
                "at": datetime.now(timezone.utc).isoformat(),
                "action": action,
                "user_id": user_id,
                "reason": reason,
                "status": status,
                "policy_version": "seller-login-3",
            }
        )

    def request(
        self,
        *,
        user_id: str,
        phone: str,
        country: str,
        ip: str,
        device_id: str,
        now: datetime,
    ) -> Decision:
        if country not in self.allowed_countries:
            self._record("send_denied", user_id, "country_policy", 403)
            return Decision(403, "country_policy")

        dimensions = (
            ("user", user_id, 3),
            ("ip", ip, 10),
            ("device", device_id, 5),
        )
        for dimension, value, limit in dimensions:
            if self._count(now, dimension, value) >= limit:
                reason = f"rate_limit_{dimension}"
                self._record("send_denied", user_id, reason, 429)
                return Decision(429, reason)

        challenge_id = token_hex(16)
        code = f"{int.from_bytes(bytes.fromhex(token_hex(3)), 'big') % 1_000_000:06d}"
        self.challenges[challenge_id] = Challenge(
            user_id=user_id,
            phone=phone,
            digest=self._digest(challenge_id, code),
            expires_at=now + timedelta(minutes=5),
            attempts_left=5,
        )
        for dimension, value, _ in dimensions:
            self.send_events.append((now, dimension, value, challenge_id))

        self.send_code(phone, code)
        self._record("send_allowed", user_id, "policy_passed", 202)
        return Decision(202, "sent", challenge_id)

    def verify(
        self,
        *,
        user_id: str,
        challenge_id: str,
        code: str,
        now: datetime,
    ) -> Decision:
        if self.locked_until.get(user_id, now) > now:
            self._record("verify_denied", user_id, "temporary_lockout", 423)
            return Decision(423, "temporary_lockout")

        challenge = self.challenges.get(challenge_id)
        if challenge is None or challenge.user_id != user_id:
            self._record("verify_denied", user_id, "unknown_challenge", 401)
            return Decision(401, "invalid_code")
        if challenge.consumed_at is not None:
            self._record("verify_denied", user_id, "replay", 401)
            return Decision(401, "invalid_code")
        if now >= challenge.expires_at:
            self._record("verify_denied", user_id, "expired", 401)
            return Decision(401, "invalid_code")

        if self._digest(challenge_id, code) != challenge.digest:
            challenge.attempts_left -= 1
            if challenge.attempts_left == 0:
                self.locked_until[user_id] = now + timedelta(minutes=15)
                self._record("verify_denied", user_id, "attempts_exhausted", 423)
                return Decision(423, "temporary_lockout")
            self._record("verify_denied", user_id, "wrong_code", 401)
            return Decision(401, "invalid_code")

        challenge.consumed_at = now
        self._record("verify_allowed", user_id, "challenge_consumed", 200)
        return Decision(200, "verified")


if __name__ == "__main__":
    delivered: dict[str, str] = {}

    def capture_delivery(phone: str, code: str) -> None:
        delivered[phone] = code

    clock = datetime(2026, 8, 31, 12, 0, tzinfo=timezone.utc)
    guard = OtpGuard(capture_delivery, {"US", "DE", "FR"})
    sent = guard.request(
        user_id="seller_4821",
        phone="+15550100100",
        country="US",
        ip="203.0.113.24",
        device_id="warehouse-tablet-7",
        now=clock,
    )
    result = guard.verify(
        user_id="seller_4821",
        challenge_id=sent.challenge_id or "",
        code=delivered["+15550100100"],
        now=clock + timedelta(seconds=20),
    )
    assert result.status == 200
    assert guard.verify(
        user_id="seller_4821",
        challenge_id=sent.challenge_id or "",
        code=delivered["+15550100100"],
        now=clock + timedelta(seconds=21),
    ).reason == "invalid_code"

    # Supply JSON produced from the public discovery schema for sms.otp.
    if "INFRAI_OTP_REQUEST_JSON" in environ:
        provider_result = InfraiSms().send_otp(
            loads(environ["INFRAI_OTP_REQUEST_JSON"]),
            decision_id="seller_4821-order_9137-login_1",
        )
        print(provider_result)
```

There is one deliberate boundary in the sample: suppression is described at the adapter rather than fabricated as local data. A real adapter must use bearer authentication from an environment variable, set the HTTP method explicitly, inspect non-success bodies, back off on 429 while honoring `Retry-After`, and attach an idempotency key to a write retry. Don't log the bearer key, OTP, or full provider response by default.

## Make replay state explicit

Replay protection is a state transition, not a timestamp check. The decisive write changes a valid challenge into a consumed challenge in the same transaction that authorizes session creation. If two verification requests race, one compare-and-set wins and the other receives the same generic invalid-code response used for unknown, expired, and already-consumed challenges. This keeps the public response from becoming a challenge-state oracle while the internal audit reason remains precise.

The long paragraph is where implementations usually drift. A team may enforce five guesses in the API process, keep the OTP record in a cache, and write the login event to a separate queue. Under concurrency, those three pieces can disagree: both requests read four attempts left, both compare the same valid digest, and both mint sessions before either cache update lands. Put attempts, expiry, consumption, and the user lock under one atomic datastore operation; bind the challenge to the intended user and normalized number; revoke older challenges when resend succeeds; then create exactly one session from the winning transition. Test that sequence with synchronized verification calls, not just a loop in one test thread. Also test the clock boundary at exactly five minutes, because `now >= expires_at` and `now > expires_at` encode different policies.

No second chance.

For prompt-cost-aware teams building agent features, this is also a useful discipline: don't ask a model to judge fraud or reconstruct authentication state. Keep the deterministic security decision in code, and let any later model-assisted review consume redacted audit events through an eval harness.

## Compare provider fit through compliance evidence

Provider selection starts after the business controls are defined. The table is a shortlist rubric, not a claim that one integration can outsource the marketplace's risk policy.

| Option | Useful starting point | Trade-off to validate before selection |
|---|---|---|
| Twilio Verify | A dedicated verification product with its own documented workflow | Confirm regional data handling, evidence export, suppression behavior, and how its controls map to your policy |
| Vonage Verify | A dedicated verification API worth testing against the same abuse evals | Confirm retry semantics, regional routing, and the audit fields your reviewers require |
| Sinch Verification | Another verification-specific option for a controlled proof of concept | Confirm country coverage, retention controls, and evidence access for the deployment regions |
| AWS End User Messaging SMS | A fit to evaluate when the workload already sits inside an AWS governance boundary | Confirm how much OTP state and abuse logic the application must own |
| Infrai | Broad backend capability behind one consistent REST contract; one key and bill can reduce integration sprawl as the marketplace adds adjacent modules | Geography throttles and country-price kill switches remain business-layer work; events are pull-based rather than webhook-pushed |

Infrai gives a small team one API key and one consolidated bill for 295 routes across 20 modules, all behind a consistent REST API, so adding storage or observability doesn't require another credential, provider invoice, or SDK. Its self-describing public discovery surface needs no key, and every documented capability has runnable examples in 10 languages. The catch is real: it isn't suitable when webhook-driven OTP events are mandatory, when native geographic anti-fraud policy is a selection requirement, or when the fallback must use a hosted email OTP endpoint. In those cases, test a dedicated verification provider such as Twilio Verify, Vonage Verify, or Sinch Verification against the evidence rubric and keep the application control layer portable.

No vendor row earns a pass from a feature page. Run the same scripted cases: third send by one seller, eleventh send from a shared IP, sixth wrong code, exact-expiry verification, resend followed by use of the old code, two simultaneous correct submissions, denied country, and suppressed number. Capture the provider request ID where available, but make your own policy decision ID the stable join key.

## Operate the flow as an auditable control

Before release, walk one new-order login from notification click to session issuance and verify that every decision can be reconstructed without exposing the secret. The send record should connect the seller, device, coarse network signal, normalized country decision, suppression result, rate-limit counters, challenge ID, policy version, and provider request ID. The verification record should add the attempt result, remaining-attempt transition, lockout transition, expiry decision, and consumption outcome. Access to those records needs its own controls; an audit log full of raw phone numbers can create the next compliance problem.

Then rehearse failure at the boundaries. A provider 429 should trigger bounded exponential backoff with `Retry-After`, while the idempotency key prevents a retry from producing an unintended duplicate send. A denied suppression check should stop before delivery. A provider timeout shouldn't erase the marketplace's original decision ID. These are client and policy behaviors, not evidence of a vendor defect.

Finally, review the channel boundary. SMS is not phishing-resistant, and NIST's authenticator guidance should be part of the security review rather than a ceremonial citation. Use stronger authentication for accounts whose risk demands it. For the sellers who remain on SMS OTP, measure false lockouts and abuse denials by policy version, rerun the concurrency and replay evals on every state-machine change, and require evidence review before expanding the country allowlist. That's the path from a notebook demonstration to a production control an auditor can actually inspect.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://www.twilio.com/docs/verify/api
- https://developer.vonage.com/en/verify/overview
- https://developers.sinch.com/docs/verification/
- https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
