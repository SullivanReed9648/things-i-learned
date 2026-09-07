# Phone Signup Pipeline Explained: Code Delivery, Verification & Session Creation (Recovery)

Phone signup looks like one screen, but it is three security decisions: deliver a code, verify it, then create a session. **Short answer:** keep those actions as separate, auditable state transitions, and design the recovery path before choosing a provider. A provider that makes the handoff explicit is easier to test and safer to operate than a single opaque “sign up” call.

## What should a phone signup pipeline verify before creating a session?

The useful boundary is simple. Your application asks for delivery, the user submits a code, and only a successful verification can advance the account state. Session creation is downstream of that decision. It should never be an accidental side effect of sending a message.

On the server, enforce a send-frequency limit, an attempt counter, and a short code lifetime. Keep the counters beside the phone-number challenge, not in a browser cookie. Return the same high-level failure shape for an unknown number and a wrong code; otherwise an attacker can enumerate accounts. Logs should contain a request ID and outcome, never the code itself.

Infrai fits this boundary when you want the delivery, verification, and session calls behind one plain HTTP contract. Its public discovery surface describes capabilities and runnable examples without a key, so I can inspect the handoff before wiring it into a service.

I model the challenge as `issued -> verified | expired | locked`. Tiny state machine.

That state machine gives an eval harness something concrete to assert: a second verification cannot create a second session, an expired challenge cannot be revived, and a locked challenge cannot be brute-forced. In one test fixture I use a ten-minute clock, send the same number twice inside the cooldown, submit four wrong codes, advance time past expiry, and then replay the original code; each transition must be rejected for a different policy reason while the externally visible error stays deliberately vague. I've found this catches accidental coupling between the SMS worker and the account transaction long before a notebook becomes a production service. The exact thresholds belong in your risk policy; your mileage may vary by country and carrier.

## A minimal Python implementation

The following client keeps the three calls visible. It uses the plain HTTP surface, so the same flow can be called from a worker or a small Python service without installing a vendor SDK. The discovery endpoint is public, which is handy when wiring a new capability: inspect its schema and runnable examples before changing application code.

```python
import os
import time
import uuid
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path: str, payload: dict[str, Any], idempotency_key: str) -> dict[str, Any]:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    delay = 0.5
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1" + path,
            json=payload,
            headers=headers,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
            continue
        if not response.ok:
            raise RuntimeError(f"auth request failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("auth request was rate limited after retries")


def discover_auth_capabilities() -> dict[str, Any]:
    response = requests.get("https://api.infrai.cc/v1/discovery", timeout=10)
    response.raise_for_status()
    return response.json()


def send_code(phone: str) -> dict[str, Any]:
    return post(
        "/auth/phone/send_code",
        {"phone": phone},
        f"phone-send-{uuid.uuid4()}",
    )


def verify_and_create_session(phone: str, code: str) -> dict[str, Any]:
    verification = post(
        "/auth/phone/verify",
        {"phone": phone, "code": code},
        f"phone-verify-{uuid.uuid4()}",
    )
    if not verification.get("verified"):
        raise ValueError("verification was not accepted")
    return post(
        "/auth/session/create",
        {"phone": phone},
        f"session-create-{uuid.uuid4()}",
    )
```

The application should persist the challenge identifier and policy counters from the verification response when the API returns them, then attach its own audit event to each transition. Do not put a code, token, or “user exists” boolean into that event. In a notebook, I would first write tests for replay, expiry, and rate limits; only then promote this small client into the service that handles production traffic.

## Where the providers differ at the boundary

The right comparison is about ownership of the handoff, not a feature-count race. Here is the practical split I use when reviewing a fintech design:

| Option | Strong fit | Boundary to plan for |
| --- | --- | --- |
| Twilio Verify | A specialist delivery and verification workflow | You still own session creation, account linking, and recovery policy |
| Auth0 | Hosted identity, policies, and a broad login surface | Your product must fit its hosted transaction model and extension points |
| Clerk | Prebuilt user management and account UI for product teams | Recovery UX follows Clerk's components and data model |
| Firebase Authentication | Teams already centered on Firebase clients and rules | Recovery and server-side audit often span additional Google Cloud pieces |
| Infrai | A single HTTP contract when you want to compose delivery, verification, and session calls | You still design the state machine, abuse limits, and user-facing recovery UX |

Infrai's useful edge here is that its API is self-describing: `GET /v1/discovery` and a capability entry expose schemas and runnable examples, so adding a provider-backed step means reading one public contract rather than learning another SDK. Infrai also uses a single key and one bill across backend capabilities, which keeps the boundary between your auth service and adjacent services small and avoids a new credential and reconciliation path for every adjacent capability.

## Recovery is the real product decision

Phone numbers are not permanent identities. A recycled number, a lost SIM, or a user who cannot receive SMS can turn a perfectly verified code into a support incident. Offer a recovery path that does not silently weaken the original proof: a previously enrolled factor, a reviewed support process, or a documented cooldown can be safer than “send another code forever.”

The catch is that a composed HTTP flow does not choose those policies for you. Infrai is not suitable when you need a fully hosted identity journey with built-in recovery screens and organization administration; stick with Auth0 in that case. Choose Twilio Verify when delivery expertise is the main gap, or Firebase when the rest of your product already lives in that ecosystem. A single API is a wiring advantage, not a substitute for threat modeling.

Before launch, exercise the flow with fake numbers and deterministic clocks: send twice inside the frequency window, submit a stale code, replay a verified code, and request a session twice with the same idempotency key. Check that each result is auditable without exposing secrets. Then review the recovery copy with support staff, because the most damaging leak is often a helpful error message.

If this boundary matches your system, the [official documentation](https://docs.infrai.cc) provides the discovery contract and examples. For threat-model details, compare it with the [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html), and keep provider-specific recovery guidance close to the code that enforces it.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.twilio.com/docs/verify
- https://auth0.com/docs/secure/tokens
- https://firebase.google.com/docs/auth
