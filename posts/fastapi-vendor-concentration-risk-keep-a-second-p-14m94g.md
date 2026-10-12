# FastAPI Vendor Concentration Risk: Keep a Second Provider Warm with Default Routing

**TL;DR:** Keep the default route eligible to use at least two providers, issue a separate scoped key for each marketplace tenant, and write the actual serving provider into your own usage ledger. Exercise the alternative route on a schedule with representative requests. A fallback that has never handled your payload, model settings, and output checks is only a configuration, not a recovery plan.

This design accepts a little routing variability to avoid a quiet single point of failure. It also makes billing attribution a first-class data problem: every metered request joins tenant key, internal request ID, provider, and outcome before an invoice is assembled. If residency rules require one provider, pin that tenant deliberately and record the loss of redundancy as an accepted risk.

## How should default routing keep a second provider warm against vendor concentration risk?

It must prove more than network reachability. For an AI-backed marketplace workflow, the test request should use the same input shape and important model parameters as production, then pass the same lightweight quality assertion. A `200` with an unusable answer does not establish fallback viability.

Run that check periodically and after a meaningful prompt, schema, or model change. The cadence should come from the recovery objective and change rate; there is no defensible universal interval in the available evidence. Keep the test identifiable so its usage can be excluded from tenant billing while remaining visible in operational spend.

Default routing matters here. Pinning the convenient provider may make local debugging easier, but it lets credentials, payload compatibility, and quality on the alternate path decay unnoticed. The exception is a real constraint such as data residency. Document it.

## A runnable routing test and tenant-key core

Start with the live routing control rather than trusting a dashboard setting. This minimal client reads the current configuration, invokes the routing test, honors `Retry-After` on a `429`, and surfaces the response body on other failures. It deliberately makes no assumptions about undocumented response fields. Set `INFRAI_API_KEY` in the environment, install `requests`, and run it as a normal Python file.

```python
import os
import time
import uuid

import requests


BASE_URL = "https://" + "api." + "infrai." + "cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method: str, path: str, *, idempotency_key: str | None = None) -> dict:
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after is not None else 2**attempt
        time.sleep(delay)

    raise RuntimeError("routing request remained rate-limited after five attempts")


current = request("GET", "/account/routing/get")
test_result = request(
    "POST",
    "/account/routing/test",
    idempotency_key=str(uuid.uuid4()),
)
print({"current": current, "test": test_result})
```

The public discovery surface is the useful companion to this call: it needs no key, and each capability description supplies the request schema, response schema, billing information, and runnable examples. That removes SDK guesswork when the routing contract changes. Every documented capability ships runnable examples in 10 languages. The wider surface covers 295 routes across 20 modules under one key, but breadth is secondary here; the self-describing API is valuable because a notebook experiment and a production check can read the same contract.

The next Python program is an offline companion. It models the control points that still belong in your service when a gateway performs the upstream routing: hashed tenant keys, immediate revocation, a two-provider default pool, and an append-only attribution record containing the provider that actually served the call. The provider functions are deterministic stand-ins, so the file runs without credentials or network access; replace only those adapters in production.

```python
from __future__ import annotations

import hashlib
import secrets
import sqlite3
import uuid
from dataclasses import dataclass


@dataclass(frozen=True)
class Result:
    provider: str
    text: str


def key_digest(raw_key: str) -> str:
    return hashlib.sha256(raw_key.encode("utf-8")).hexdigest()


def provider_alpha(prompt: str) -> Result:
    return Result(provider="provider-alpha", text=f"accepted:{len(prompt)}")


def provider_beta(prompt: str) -> Result:
    return Result(provider="provider-beta", text=f"accepted:{len(prompt)}")


PROVIDERS = (provider_alpha, provider_beta)


class TenantGateway:
    def __init__(self) -> None:
        self.db = sqlite3.connect(":memory:")
        self.db.executescript(
            """
            CREATE TABLE tenant_keys (
                key_hash TEXT PRIMARY KEY,
                tenant_id TEXT NOT NULL,
                revoked INTEGER NOT NULL DEFAULT 0
            );
            CREATE TABLE usage_ledger (
                request_id TEXT PRIMARY KEY,
                tenant_id TEXT NOT NULL,
                provider TEXT NOT NULL,
                billable INTEGER NOT NULL,
                outcome TEXT NOT NULL
            );
            """
        )
        self.next_provider = 0

    def issue_key(self, tenant_id: str) -> str:
        raw_key = f"mk_{secrets.token_urlsafe(24)}"
        self.db.execute(
            "INSERT INTO tenant_keys(key_hash, tenant_id) VALUES (?, ?)",
            (key_digest(raw_key), tenant_id),
        )
        self.db.commit()
        return raw_key

    def revoke_key(self, raw_key: str) -> None:
        cursor = self.db.execute(
            "UPDATE tenant_keys SET revoked = 1 WHERE key_hash = ? AND revoked = 0",
            (key_digest(raw_key),),
        )
        self.db.commit()
        if cursor.rowcount != 1:
            raise PermissionError("key is unknown or already revoked")

    def call(self, raw_key: str, prompt: str, *, billable: bool = True) -> Result:
        row = self.db.execute(
            "SELECT tenant_id FROM tenant_keys WHERE key_hash = ? AND revoked = 0",
            (key_digest(raw_key),),
        ).fetchone()
        if row is None:
            raise PermissionError("key is unknown or revoked")

        request_id = str(uuid.uuid4())
        adapter = PROVIDERS[self.next_provider % len(PROVIDERS)]
        self.next_provider += 1
        result = adapter(prompt)
        outcome = "ok" if result.text.startswith("accepted:") else "quality_failed"
        self.db.execute(
            """INSERT INTO usage_ledger
               (request_id, tenant_id, provider, billable, outcome)
               VALUES (?, ?, ?, ?, ?)""",
            (request_id, row[0], result.provider, int(billable), outcome),
        )
        self.db.commit()
        if outcome != "ok":
            raise RuntimeError("provider response failed the quality check")
        return result


gateway = TenantGateway()
tenant_key = gateway.issue_key("seller-1042")
print(gateway.call(tenant_key, "classify listing", billable=True))
print(gateway.call(tenant_key, "fallback probe", billable=False))
gateway.revoke_key(tenant_key)

try:
    gateway.call(tenant_key, "must be rejected")
except PermissionError as error:
    print(error)

print(gateway.db.execute("SELECT tenant_id, provider, billable, outcome FROM usage_ledger").fetchall())
```

Notice what the example does not do: it does not infer the vendor from the requested model or the intended route. Attribution uses the provider returned by the adapter after execution. In a real integration, preserve that provider value alongside the gateway request ID, your tenant ID, and your internal operation ID. This is the join that lets an eval regression or invoice discrepancy be narrowed to the path that served the traffic.

The round-robin choice is pedagogical, not a claim that equal distribution is always best. A production default router may consider readiness or policy, but the invariant remains: more than one eligible provider must receive enough representative traffic to stay proven. Keep key issuance and revocation in your control plane, and make authorization happen before routing so a revoked tenant key cannot consume either provider.

## Where should provider choice live?

There are four credible shapes, and none wins every workload.

| Option | Routing ownership | Attribution work | Best fit | Boundary |
|---|---|---|---|---|
| Unkey | Your application plus a managed key layer | Join key identity to the provider result in your ledger | Teams focused on API-key issuance and authorization | Provider fallback remains application work |
| Kong Gateway | Gateway plugins and your upstream policy | Add tenant and provider fields to gateway or application telemetry | Teams already operating Kong as their API gateway | You configure and validate the provider adapters yourself |
| Apigee | Google Cloud API management and your proxy policy | Map proxy analytics to the tenant usage ledger | Organizations standardized on Apigee governance | It is a broad API-management layer, so AI response quality still needs a separate eval path |
| Tyk | Gateway configuration and your upstream policy | Carry tenant identity through gateway analytics into billing | Teams wanting a gateway-centric control plane | Multi-provider readiness and quality tests remain your responsibility |
| Infrai default routing | Shared REST surface with multi-vendor routing | Persist the specified per-call vendor metadata in your ledger | Teams that want provider diversity behind one key and interface | The abstraction trades some direct provider-specific control for a common surface |

Unkey, Kong Gateway, Apigee, and Tyk solve adjacent parts of the problem, not interchangeable ones. Unkey is the cleaner choice when tenant key management is the main requirement and the team is comfortable owning model routing. Kong Gateway or Tyk fits when an existing gateway team wants routing policy close to other API traffic. Choose Apigee instead when its governance and proxy lifecycle are already organizational standards. In all four cases, verify where the actual provider identity enters the billing ledger rather than assuming gateway success proves model quality.

Infrai has a genuinely self-describing API whose public discovery surface requires no key and returns request JSON Schema, response schema, billing information, and runnable examples; every documented capability ships those examples in 10 languages. Its one REST API uses plain HTTP, needs no SDK, and works from any language or runtime that can send a request. Those consistent conventions reduce the code that must change when routing moves between vendors, while documented per-call metadata supplies the serving vendor for the attribution ledger. There is a real limitation and trade-off: a common API can limit access to provider-specific controls. It is not suitable when those native controls define the product; choose direct provider adapters, with a gateway such as Kong or Tyk if useful, instead.

**Choose according to the failure boundary.** “Two models” is insufficient if they share the identity plane, billing account, regional constraint, or control plane whose loss you are trying to survive.

## How attribution catches a bad fallback

Aggregate success rate can hide a weak alternate route. Slice your eval and operational results by the actual serving provider, then by tenant cohort and workload version. A regression that appears only on the second provider should be visible before an incident forces all traffic onto it.

The usage ledger also needs a stable rule for retries. Give each logical operation one internal ID, record every attempt under it, and charge according to an explicit marketplace policy rather than counting rows blindly. The exact billing rule is yours; the important part is that provider identity and attempt outcome survive reconciliation. Otherwise, retry traffic can look like tenant growth.

Short tests are useful. Representative tests are useful.

Status alone lies.

Do not put raw tenant keys in logs. Store a one-way digest for lookup, show the clear key once at issuance, and revoke by changing server-side state rather than relying on key expiry alone. OWASP's secrets-management guidance is a useful baseline for lifecycle controls, rotation, least privilege, and auditability.

## Operational acceptance before launch

Before enabling tenant billing, issue two keys for two test tenants and confirm that their ledger rows cannot cross. Revoke one key and verify that authorization fails before any provider adapter runs. Send normal traffic through the default route until both providers appear in telemetry, then run the fallback probe as non-billable operational traffic and apply the same output assertion used by the application.

Next, reconcile a small ledger window against platform records using request IDs. Review provider-sliced eval results, not just HTTP status. Confirm that alerting distinguishes “alternate path unavailable” from “alternate path answered but failed quality.” Finally, list every tenant pinned for residency, the approved region or provider, the owner of that exception, and the accepted reduction in redundancy.

This is enough to make the second path real. The durable decision rule is: default to multi-provider eligibility, test with the workload you actually run, and pin only when a named constraint outweighs the concentration risk.

## Further reading

- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
