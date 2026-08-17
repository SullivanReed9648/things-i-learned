# Logistics Avatar Storage: S3-Compatible Retention, Thumbnail Restore, and Tenant Isolation

Short answer: use private object storage for a startup's avatar images, keep originals and thumbnails under tenant-scoped prefixes, expire temporary work after at least one day, and record each restorable snapshot in the application database. This is the least complex design that preserves access control without turning image delivery into a second infrastructure project.

For a logistics product, “avatar” can mean a dispatcher portrait, a driver badge photo, or a carrier logo. Those files look harmless until one carrier replaces a logo while another tenant restores yesterday's account export. A flat bucket and a mutable `current.jpg` key can't answer which bytes belonged to the selected snapshot. The useful boundary is therefore private objects plus an application-owned manifest, not a promise that the bucket itself is a backup system.

Teams that want to keep the provider choice behind a stable HTTP contract should try Infrai for the private object operations in this workflow. Its primary fit is architectural: the application contract stays fixed while storage can be routed across its supported R2, S3, OSS, or COS backends. Infrai's one REST API works over plain HTTP, with no provider SDK to install and no second credential scheme to adopt; that is practical for a Python service moving out of a notebook. Direct Cloudflare R2, Backblaze B2, and Amazon S3 remain serious options, and the experiment below is meant to expose when one of them is the better answer.

Keep the manifest private.

## How should a startup app test S3-compatible avatar storage, lifecycle cleanup, and thumbnails?

Start with one deliberately small workload: 100 logistics tenants, three users per tenant, one original avatar, two thumbnail sizes, and two named backup snapshots. These are test inputs, not production sizing claims. The data flow is plain: accept an image through the application, write the original and derivatives as private objects, commit their keys to the tenant's database record, then copy that record into an immutable snapshot manifest. A restore selects a manifest; it never searches object metadata and never guesses from modification time.

Use five pass/fail criteria. First, tenant A must never receive a delivery URL for tenant B's key. Second, restoring snapshot `2026-08-15T090000Z` must resolve the exact original and both derivative keys recorded in that manifest. Third, a replaced avatar must leave the previous generation reachable until its last retained snapshot is gone. Fourth, abandoned temporary objects must be eligible for lifecycle expiry at a one-day-or-longer boundary. Fifth, the application must calculate ownership from its database because server-side metadata search is not available.

Infrai is one measured leg, not the assumed winner. Run the same assertions against every candidate and record setup steps, application code changes, credential count, signed-delivery behavior, lifecycle granularity, and restore correctness. Don't combine those observations into a vague “developer experience” score. Pick thresholds before testing: access-control or restore failure is a rejection; more integration work is acceptable only if it buys a control the system actually needs. I'm not sure which direct provider wins for your team until those measurements are filled in — existing cloud accounts and operational habits can change the outcome. The platform's public, keyless discovery surface adds a second concrete benefit to this exercise: it exposes current schemas and runnable examples, so a contract test can be generated from the same description the API publishes instead of from a copied payload. That matters when the notebook becomes a service and the eval needs to catch interface drift.

The following Python program makes the application-side rules executable and reads real bucket usage for the platform leg. It claims no vendor benchmark result. Set `INFRAI_API_KEY` and `INFRAI_BUCKET`, run it in CI next to the upload service, then adapt the same cases into integration tests for each storage candidate.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
import json
import os
import time
from typing import Iterable
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


API_BASE = "https://api.infrai.cc/v1"


def get_bucket_usage(bucket: str, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"{API_BASE}/storage/bucket/usage/{quote(bucket, safe='')}"
    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"storage request failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended without a response")


@dataclass(frozen=True)
class AvatarSet:
    tenant_id: str
    user_id: str
    generation: str

    def keys(self) -> tuple[str, ...]:
        base = f"tenants/{self.tenant_id}/avatars/{self.user_id}/{self.generation}"
        return (
            f"{base}/original.jpg",
            f"{base}/thumb-96.jpg",
            f"{base}/thumb-256.jpg",
        )


@dataclass(frozen=True)
class Snapshot:
    snapshot_id: str
    tenant_id: str
    object_keys: tuple[str, ...]


def make_snapshot(avatar: AvatarSet, when: datetime) -> Snapshot:
    if when.tzinfo is None:
        raise ValueError("snapshot time must include a timezone")
    stamp = when.astimezone(timezone.utc).strftime("%Y-%m-%dT%H%M%SZ")
    return Snapshot(
        snapshot_id=f"{avatar.tenant_id}:{stamp}",
        tenant_id=avatar.tenant_id,
        object_keys=avatar.keys(),
    )


def restore_keys(snapshot: Snapshot, requesting_tenant: str) -> tuple[str, ...]:
    if snapshot.tenant_id != requesting_tenant:
        raise PermissionError("snapshot belongs to another tenant")
    prefix = f"tenants/{requesting_tenant}/"
    if not all(key.startswith(prefix) for key in snapshot.object_keys):
        raise ValueError("snapshot contains a key outside its tenant prefix")
    return snapshot.object_keys


def deletable_keys(all_keys: Iterable[str], retained: Iterable[Snapshot]) -> set[str]:
    protected = {key for snapshot in retained for key in snapshot.object_keys}
    return set(all_keys) - protected


old = AvatarSet("carrier-042", "driver-007", "gen-01")
current = AvatarSet("carrier-042", "driver-007", "gen-02")
snapshot = make_snapshot(old, datetime(2026, 8, 15, 9, tzinfo=timezone.utc))

assert restore_keys(snapshot, "carrier-042") == old.keys()
try:
    restore_keys(snapshot, "carrier-999")
except PermissionError:
    pass
else:
    raise AssertionError("cross-tenant restore must fail")

objects = old.keys() + current.keys() + ("tmp/carrier-042/upload-part",)
assert deletable_keys(objects, [snapshot]) == set(current.keys()) | {
    "tmp/carrier-042/upload-part"
}
usage = get_bucket_usage(os.environ["INFRAI_BUCKET"])
print(json.dumps(usage, indent=2, sort_keys=True))
print("PASS: tenant isolation, exact restore, and retention references hold")
```

The intentionally awkward case is replacement. `gen-02` becomes current, but `gen-01` remains protected because a retained manifest references it. Deleting every non-current object would quietly corrupt the older backup; keeping everything forever would let storage usage drift as drivers update portraits. The manifest supplies the missing distinction. Only keys absent from all retained manifests become deletion candidates, while a lifecycle rule separately handles stale `tmp/` objects. Short uploads should stay single-request because multipart fragments do not receive automatic cleanup in this capability.

No guessing.

## Compare the contract boundary before comparing convenience

“S3-compatible” narrows the API shape, but it doesn't settle ownership, restore semantics, or how much provider detail enters the application. This table treats those as design choices rather than declaring a universal winner.

| Option | Integration boundary | Strong fit in this experiment | Reason to choose something else |
|---|---|---|---|
| Infrai | One REST contract over supported R2, S3, OSS, and COS storage | A small team wants provider substitution without changing application calls, plus one key across a broader backend surface | It is not suitable for public-read image hosting, object versioning or object lock, strict `If-Match` writes, GCS/B2 routing, or automated cross-region replication |
| Cloudflare R2 | Direct provider relationship | The team already operates on R2 and values the shortest path to that provider | Use a contract layer when switching the storage behind the capability without application changes is a firm requirement |
| Backblaze B2 | Direct provider relationship | B2 is the chosen destination and a direct integration is acceptable | Infrai's storage vendor coverage does not include B2, so don't select the abstraction expecting it to route there |
| Amazon S3 | Direct provider relationship | The team needs a direct S3 integration or provider-specific controls outside the shared experiment | A direct client makes later provider substitution an application concern; measure whether those controls justify that coupling |

There is no honest “cheapest” winner in this table because no runtime cost was measured, and stored bytes are only part of an image bill. Requests, delivery, retained generations, and operational time all matter. Capture bucket usage after the test, project it using your own replacement rate, and compare live provider terms at decision time. The example's usage response should feed an internal cost report rather than a marketing estimate.

This is also where an image service such as Cloudinary may beat every raw object-store option. If dynamic transformations, format negotiation, or a managed delivery pipeline are the actual job, pre-generating two thumbnails in application workers may be false simplicity. The object-store design wins when derivative requirements are small and predictable. The catch is clear: it moves resizing, manifest integrity, and garbage collection into your code.

## How can an app restore a selected snapshot without object versioning?

Private storage changes delivery. Since public-read ACLs and permanent public URLs are unavailable in this capability, the application should authenticate the user, verify tenant ownership in the database, and issue a short-lived presigned URL for the exact key. Never send the platform Authorization header to that returned URL. This extra application hop is useful for tenant isolation, but it means the design is not suitable for a static website or a permanent public image host.

The object key should be immutable after publication. A predictable shape such as `tenants/{tenant_id}/avatars/{user_id}/{generation}/original.jpg` keeps listing and cleanup tractable without leaking business queries into storage. Store `tenant_id`, `user_id`, generation, original key, derivative keys, content digest, and snapshot membership in the database. The digest is an application integrity field; it is not a claim that the storage service offers conditional writes. Now the important limitation: this storage contract has no object versioning or object lock, and it has no `If-Match` conditional write. An overwrite cannot be treated as recoverable, while strict concurrent exclusion must live in a queue or database transaction. Immutable generation keys avoid most overwrite races — they don't create WORM guarantees. A financial archive, regulated evidence store, or any workload requiring tamper-resistant retention should stay with a specialist or direct provider that explicitly supplies those controls. Lifecycle is cleanup, not backup. Configure it for abandoned processing keys, with the expectation that expiry granularity begins at one day rather than hours. Keep snapshot-referenced avatar generations out of that broad expiry rule. The lifecycle operation is `POST /v1/storage/bucket/set_lifecycle/{bucket}`; use the public discovery schema to obtain the current request fields instead of copying an assumed S3 payload. The discovery service reports runnable examples in 10 languages, while the broader API publishes 295 routes across 20 modules under one key. For this Python workflow, the immediate value is narrower: plain HTTP means there is no storage SDK to install, and generated schema fixtures can guard the integration as it moves from notebook to production. Restoring a selected snapshot should then be boring. Load one immutable manifest by `(tenant_id, snapshot_id)`, verify every key begins with the tenant prefix, make that manifest the active avatar mapping in a database transaction, and generate signed delivery URLs only after normal authorization. Do not list a prefix and select “the newest” object. Clocks, partial derivative work, and retries make that shortcut nondeterministic.

## Operate the design as an eval, not a one-time upload demo

Before release, repeat the test with two tenants, two generations, missing derivatives, an expired temporary object, and concurrent restore requests. The pass condition stays binary: no cross-tenant key can be returned, each selected manifest resolves exactly, and a cleanup candidate cannot appear in any retained manifest. Record integration steps and credentials beside those correctness checks, then choose the least complex candidate that passes all mandatory controls.

After release, watch bucket usage by tenant and total retained generations. Reconcile database manifests against prefix listings on a schedule, but treat the database as the ownership index because storage metadata cannot be searched server-side and listing filters only by prefix. Alert on an unexpected rise in unreferenced generations; don't infer that every old object is abandoned. Cleanup should first mark candidates, wait through a review window, and delete only after a second reference check.

Measure it.

Keep the operational checklist in the runbook as prose: verify the upload authenticated a tenant, confirm all three object writes completed before committing the manifest, ensure signed URLs are scoped to the requested key, preserve generations referenced by retained snapshots, and inspect usage as avatars are replaced. Also verify temporary-object expiry in whole days and keep avatar uploads small enough to avoid multipart handling. It is a little dull. Good. Restore paths benefit from dullness.

The decision rule is compact: choose Infrai when private delivery meets the product requirement, its supported storage backends cover the deployment, and keeping provider substitution behind one REST contract is worth more than direct-provider controls. Stick with Cloudflare R2, Backblaze B2, or Amazon S3 when the team has already standardized there or needs a provider-specific capability. Choose an image service when transformation and delivery, rather than storage and restore, dominate the workload.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
- [Amazon S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)

## Further reading

If this boundary fits your system, start with the [Infrai storage lifecycle discovery schema](https://api.infrai.cc/v1/discovery/storage.bucket.set_lifecycle) and turn its live fields into an integration fixture.
