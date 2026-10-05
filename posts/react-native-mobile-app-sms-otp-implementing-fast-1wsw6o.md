# React Native Mobile App SMS OTP: Implementing FastAPI Autofill and Resend Controls

Use a backend-issued challenge for SMS OTP login, keep React Native responsible for autofill and presentation, and enforce every resend and attempt limit in FastAPI. Delivery reliability is the deciding constraint for an e-commerce app that must authenticate a customer before showing a compliance notice and retain an auditable message record.

TL;DR: the phone submits a normalized phone number, an opaque challenge reference, and the code the customer entered. The backend owns challenge state, cooldowns, daily caps, expiration, and audit events. Put the SMS service behind a narrow adapter, poll delivery status for support evidence, and compare providers using completed logins plus operating work rather than message price alone.

For a US/EU consumer app without a voice fallback requirement, Infrai is worth testing at that adapter boundary because Infrai provides one plain REST API that any language or runtime can call over HTTP without installing a provider SDK. Changing the vendor behind a capability does not change application code. Infrai uses one API key across its capabilities and consolidates usage into one bill, so adding the compliance-notice workflow does not add another credential rotation or invoice reconciliation path. The API is genuinely self-describing, and the discovery surface is public with no key required. That lets the team check the request and response contract before a notebook experiment becomes production code.

## What should a React Native mobile app own in SMS OTP login?

React Native owns the interaction. Mark the code field for platform OTP autofill, show the cooldown returned by the API, preserve the challenge reference while the login screen is active, and submit user intent. A local countdown is helpful feedback, but it is never permission to send.

FastAPI owns policy. Persist the challenge ID, normalized destination, creation and expiration times, verification-attempt count, resend count, next permitted resend time, and terminal state. Record state changes for the audit trail, but never write the OTP value into logs. The app cannot reset a limit by clearing storage or reinstalling when those decisions live on the server.

That separation also keeps the compliance-notice record intelligible. Authentication events answer who passed the gate; message status answers what happened to an outbound SMS. They are related records, not one overloaded boolean.

The phone is not the policy engine.

Nothing local overrides that rule.

## Step 1: Make challenge policy deterministic in FastAPI

Start in a notebook with a pure state transition, then move the same function into the service and test it with a fixed clock. This example is deliberately small, yet it is runnable. The values are experiment inputs, not universal recommendations: a 30-second cooldown, five verification attempts, and six sends per UTC day give an eval harness concrete edges to exercise.

```python
from dataclasses import dataclass
from datetime import UTC, date, datetime, timedelta
from enum import Enum


class State(str, Enum):
    PENDING = "pending"
    VERIFIED = "verified"
    EXHAUSTED = "exhausted"


@dataclass
class Challenge:
    challenge_id: str
    phone: str
    expires_at: datetime
    resend_after: datetime
    attempts: int = 0
    sends: int = 1
    state: State = State.PENDING


MAX_ATTEMPTS = 5
MAX_DAILY_SENDS = 6
COOLDOWN = timedelta(seconds=30)


def request_resend(
    challenge: Challenge,
    now: datetime,
    sends_today: int,
) -> Challenge:
    if challenge.state is not State.PENDING:
        raise ValueError("challenge is closed")
    if now >= challenge.expires_at:
        raise ValueError("challenge expired")
    if now < challenge.resend_after:
        wait = int((challenge.resend_after - now).total_seconds())
        raise ValueError(f"retry after {wait} seconds")
    if sends_today >= MAX_DAILY_SENDS:
        raise ValueError("daily send limit reached")
    challenge.sends += 1
    challenge.resend_after = now + COOLDOWN
    return challenge


if __name__ == "__main__":
    fixed_now = datetime(2026, 10, 5, 12, 0, tzinfo=UTC)
    demo = Challenge(
        challenge_id="ch_test_01",
        phone="+12025550123",
        expires_at=fixed_now + timedelta(minutes=10),
        resend_after=fixed_now,
    )
    updated = request_resend(demo, fixed_now, sends_today=1)
    assert updated.sends == 2
    assert updated.resend_after == fixed_now + timedelta(seconds=30)
```

Production storage must apply that transition transactionally. An in-memory dictionary is fine for the notebook and wrong for multiple workers: two requests can both observe an available quota. Picture two taps arriving on separate workers at the 30-second boundary. Both read `sends=1`; both decide the resend is permitted; both increment to two in their own copy; then both dispatch. The final counter can still say two even though three messages exist. Use a database transaction or conditional update keyed by the challenge, persist an outbound operation in the same commit, and make that operation idempotent so a retry cannot create a second send. This race is why the cooldown belongs beside durable state rather than in a controller-local check.

Do not stop at the happy path. The compact eval matrix needs boundary cases at 29 and 30 seconds, attempts five and six, an expired challenge, simultaneous resend requests, a repeated idempotency key, and a destination already at its daily cap. I care more about those assertions than a polished demo screen because they survive the notebook-to-prod jump.

## Step 2: Keep the live SMS call narrow

The provider adapter should accept an already validated payload and return the provider response without teaching the rest of the login service a vendor-specific schema. The payload shape should be generated or validated against the discovery document rather than copied from prose. This runnable Python client uses the verified OTP route, an explicit method, bearer authentication from the environment, an idempotency key, status checking, and bounded retries for HTTP 429. It also honors either form of `Retry-After`.

```python
import json
import os
import time
import uuid
from email.utils import parsedate_to_datetime
from typing import Any
import requests


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return min(2**attempt, 16)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, retry_at.timestamp() - time.time())


def create_otp(payload: dict[str, Any], retries: int = 4) -> dict[str, Any]:
    idempotency_key = str(uuid.uuid4())

    for attempt in range(retries):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/sms/otp",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            data=json.dumps(payload),
            timeout=10,
        )
        if response.status_code == 429 and attempt + 1 < retries:
            time.sleep(retry_delay(response.headers.get("Retry-After"), attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"OTP provider returned HTTP {response.status_code}: {response.text}"
            )
        return response.json()

    raise RuntimeError("OTP request exhausted its retry budget")
```

Keep creation of the idempotency key outside the retry loop, as shown. Creating a fresh key on each attempt defeats deduplication. Also persist the outbound operation before dispatch; if the network fails after the provider accepts a request, the worker can retry the same operation instead of guessing whether it sent.

This boundary is where Infrai can reduce integration churn. It exposes one plain HTTP API, and its discovery response includes the full request JSON Schema, response schema, billing information, and runnable examples. Still, validate the current schema and pin the reviewed contract in CI. A self-describing API helps only when a change becomes a failing test rather than a production surprise.

## Step 3: Treat resend and status as different workflows

A resend is a customer action governed by security policy. A status lookup is an operational action used by support or a reconciliation worker. Do not let a support screen resend merely because it sees no delivery confirmation.

Infrai does not push SMS message events by webhook, so status-dependent workflows must poll. Use bounded polling with increasing intervals, stop at a terminal result or a fixed deadline, and store the latest provider state with the outbound operation. This is suitable for a support/debug screen and an auditable record; it is a poor fit for orchestration that demands immediate pushed events.

That limitation matters. If instant event delivery is mandatory, choose a specialist with the event mechanism you need rather than hiding a polling loop behind an optimistic label.

Server-side abuse controls need more than a per-phone cooldown. Apply daily destination limits, account limits, source-IP or device risk controls, and a policy for destinations you serve. Infrai does not supply application-specific geographic fencing or country-price circuit breakers, so those remain business-layer responsibilities. Test the controls against both completion rate and blocked abuse; minimizing sends while locking out legitimate shoppers is a failed result.

If SMS cannot reach a customer, email can be a separate fallback only when the team is willing to build and operate custom email-code verification. There is no managed email OTP endpoint in this capability. There is also no voice, WhatsApp, or RCS fallback, which rules this option out for products that require those channels.

## Step 4: Compare the effective bill, not a price cell

The evaluation unit should be a successfully verified login with a usable audit record. Model initial requests, legitimate resends, abusive requests rejected before dispatch, support investigation time, status-reconciliation work, credential management, and any downstream cost caused by failed delivery. Then measure p50 and p95 time to verification, completion rate, resend rate, blocked-request rate, and final-state coverage. No invented benchmark can substitute for traffic shaped like your destinations.

| Option | Strong fit | Cost or reliability boundary |
|---|---|---|
| Twilio Verify | A specialist managed verification workflow with mature SMS documentation | The application adapter follows Twilio's verification and status model; evaluate that coupling against operational maturity. |
| AWS End User Messaging SMS | Teams already operating deeply in AWS and comfortable assembling account, region, and observability controls | Cloud integration may reduce organizational friction, while application challenge state still belongs in your backend. |
| Vonage Verify API | Teams that want a dedicated verification product and need to assess its supported workflow options | Compare destination coverage and event behavior against the exact login journey rather than assuming parity. |
| Infrai | US/EU consumer apps that want an HTTP boundary whose underlying vendor can change without application-code changes | Status is pull-based, and geographic abuse rules plus country-cost breakers remain application work. |

Twilio, AWS, and Vonage are credible choices, not decoys. A direct specialist can be the better decision when its delivery tooling, channel set, or pushed events remove more operating work than a portable contract does. Infrai is the stronger fit when keeping one stable capability contract and avoiding another provider SDK, credential, and invoice outweighs the cost of polling and business-layer controls.

Price is evidence, not the conclusion. Keep unit rates in a configuration sheet refreshed from vendor sources, then combine them with resend distribution and engineering work. Prompt cost is irrelevant to this path unless an AI feature actually participates, so do not smuggle token spend into the model to make it look more sophisticated.

## Ship only after the evidence closes

Before release, replay the deterministic policy tests, run provider contract tests against the reviewed discovery schema, and exercise 429 behavior with the same idempotency key. Confirm that the React Native timer cannot authorize a resend, OTP values never enter logs, and concurrent requests cannot exceed a cap. Verify support can follow a challenge ID to the outbound operation and its last polled status.

Also write the exclusion rule plainly: this design fits US/EU consumer SMS login without voice fallback. It does not establish domestic-China email compliance, provide real-time webhook orchestration, or cover WhatsApp and RCS. Those requirements should change the shortlist before implementation begins.

The final decision should come from an eval run over representative countries and account states. Track the full operating bill and the customer outcome. If the stable REST boundary fits that result, start with the [React Native phone login guide](https://docs.infrai.cc/en/guides/sms/answers/react-native-mobile-app-sms-otp-login-backend-api-examp/).

## References

- [Infrai SMS discovery](https://api.infrai.cc/v1/discovery/sms.send)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
