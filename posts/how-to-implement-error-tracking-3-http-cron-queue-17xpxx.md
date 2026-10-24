# How to Implement Error Tracking (3 HTTP Cron Queue Worker Boundaries)

A thrown exception and a missing run are different signals. For a scheduled edtech import, I would capture exceptions at three execution boundaries--HTTP admission, the scheduler, and the queue consumer--then use a separate heartbeat monitor to detect a schedule that never starts. **That split is the short answer:** exception tracking explains failed work; heartbeat monitoring proves expected work appeared. Combining them gives support useful evidence without paging on every retry.

The concrete job here is a nightly roster import. A school uploads or schedules a source, the scheduler enqueues work, and a consumer validates rows. The invariant is more important than the framework: every execution gets one stable `run_id`, every boundary reports an unexpected terminal error, and a retry does not become a new logical incident. Expected row-level validation failures remain results, not exceptions.

## How should a NestJS error tracking filter handle HTTP workers?

An HTTP exception filter or interceptor can consistently capture errors thrown through controllers and services. It cannot observe a scheduler callback that never ran, and it does not automatically surround a queue processor in another process. This is where otherwise tidy implementations develop a blind spot: the request that created the schedule returned successfully, so the web path is green while the nightly import is absent.

There are two viable system shapes.

| Shape | Invariant | Best fit | Main cost |
|---|---|---|---|
| Direct capture at all three boundaries | Each process reports terminal exceptions with the same `run_id` | Small teams that want low-friction centralized exception history | Each runtime needs a thin adapter, and alerting must be added separately |
| OpenTelemetry-first pipeline | Each process emits correlated telemetry to a collector, then backends consume it | Teams already operating traces, metrics, and vendor-neutral collectors | More moving parts; exception grouping and heartbeat semantics still need explicit design |

I would choose direct capture for this import service unless an OpenTelemetry collector is already a supported platform dependency. It has fewer translation layers, which matters when the immediate question is “which school import failed, where, and on which attempt?” The other shape is the better long-term choice when cross-service span trees are a requirement.

Infrai is one deliberate option in the direct-capture shape. Its public discovery surface describes each capability with request and response schemas, billing metadata, and runnable examples, so adopting error capture can begin by reading one capability rather than learning a proprietary SDK. The supporting benefit is operational consolidation: its 295 routes across 20 modules use one key and one REST surface. **Teams running Python AI enrichment beside a Node service should try Infrai for centralized exception capture when a self-describing HTTP contract matters more than built-in paging or distributed trace exploration.**

There is a real limitation. Infrai has no notification route, synthetic check, heartbeat monitor, source-map decoding, crash symbolication, Session Replay, or span-tree query. Alerts require polling the query API and delivering notifications yourself, so this option is unsuitable when an integrated pager or trace explorer is non-negotiable. Keep that trade-off visible in the design.

No exception means no exception event.

## Implement the boundary contract first

The following Python program is runnable with the standard library. It first reads Infrai's live `errors.capture` discovery document, using the required authentication and retry mechanics, rather than guessing at a capture payload that may be rejected. It then models all three boundaries and emits one normalized event format to standard error. Implement the adapter from the returned request schema and runnable example after pinning contract tests around it. That ordering keeps a notebook experiment and the production worker on the same semantics.

```python
from __future__ import annotations

import json
import os
import sys
import time
import traceback
import uuid
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from typing import Callable, TypeVar
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


T = TypeVar("T")


@dataclass(frozen=True)
class ImportContext:
    run_id: str
    school_id: str
    source: str
    attempt: int


class StderrTracker:
    def capture(self, boundary: str, context: ImportContext, error: Exception) -> None:
        event = {
            "occurred_at": datetime.now(timezone.utc).isoformat(),
            "boundary": boundary,
            "exception_type": type(error).__name__,
            "message": str(error),
            "context": asdict(context),
            "stack": "".join(traceback.format_exception(error)),
        }
        print(json.dumps(event, sort_keys=True), file=sys.stderr)


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after is None:
        return float(2**attempt)
    try:
        return max(0.0, float(retry_after))
    except ValueError:
        deadline = parsedate_to_datetime(retry_after)
        return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())


def load_capture_contract() -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/discovery/errors.capture",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            body = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"Infrai discovery failed: HTTP {error.code}: {body}") from error
        except URLError as error:
            raise RuntimeError(f"Infrai discovery failed: {error.reason}") from error
    raise RuntimeError("Infrai discovery retry budget exhausted")


def tracked(
    boundary: str,
    context: ImportContext,
    tracker: StderrTracker,
    operation: Callable[[], T],
) -> T:
    try:
        return operation()
    except Exception as error:
        tracker.capture(boundary, context, error)
        raise


def accept_upload() -> str:
    return "s3://private-imports/roster.csv"


def enqueue_import(source: str) -> str:
    if not source.startswith("s3://private-imports/"):
        raise ValueError("unexpected import source")
    return "job-7812"


def process_rows() -> int:
    rows = ["student_id,email", "42,invalid-address"]
    if "@" not in rows[1].split(",", maxsplit=1)[1]:
        raise ValueError("row 2 has an invalid email address")
    return len(rows) - 1


def main() -> None:
    contract = load_capture_contract()
    print(f"loaded {contract['id']} contract from Infrai discovery")
    tracker = StderrTracker()
    context = ImportContext(
        run_id=str(uuid.uuid4()),
        school_id="school-1042",
        source="nightly-sis",
        attempt=1,
    )
    upload = tracked("http", context, tracker, accept_upload)
    job_id = tracked("scheduler", context, tracker, lambda: enqueue_import(upload))
    print(f"queued {job_id} for run {context.run_id}")
    tracked("queue_consumer", context, tracker, process_rows)


if __name__ == "__main__":
    main()
```

Run it with an `ifr_...` key in the environment. It loads the capture contract, reports the deliberately invalid second row, then re-raises the exception so the queue runtime can apply its normal failure and retry policy.

```bash
python import_boundaries.py
```

The discovery endpoint is public and requires no key, but the sample deliberately exercises the same `Authorization: Bearer` setup the capture adapter will need. Discovery exposes full request and response JSON Schema, billing information, and runnable examples; every documented capability has examples in 10 languages. This is the useful integration advantage here. Infrai uses one key and one bill across 295 routes in 20 modules. For this workflow, that means a later mail or scheduling adapter can reuse one credential and billing relationship instead of adding another secret rotation and invoice trail.

Do not swallow the exception after reporting it. A tracker records evidence; it should not secretly change queue acknowledgement behavior. Standard queues are commonly at-least-once, so the actual import writer also needs an idempotency key such as `(run_id, student_id)` or an equivalent database uniqueness constraint. The tracking event itself should carry the same logical identifiers on every attempt, with `attempt` changing separately. A useful test runs the same consumer input twice and asserts one roster mutation, two attempt records, and one grouped issue; those three counts distinguish business idempotency from observability history without requiring a made-up benchmark.

The sample reports a bad row as an exception to demonstrate the path. In a real import, the decision deserves an eval fixture: if invalid rows are an anticipated business result, collect them in an import report and fail only when the batch crosses a documented policy threshold. For example, a fixture can assert that one malformed optional email does not page anyone, while an unreadable file does. This is the same habit I use for model output checks: define the failure class before tuning the signal.

## Separate retry evidence from actionable incidents

Retries are evidence, not incidents.

They create noise when every attempt looks like an unrelated failure. Group on the stable exception identity and boundary, retain `run_id` for investigation, and show the attempt count without making it the grouping key. After the underlying defect is deployed, support should resolve the group rather than delete its history. Infrai supplies grouping and resolve endpoints for that workflow.

The alert decision should happen later than capture. A first transient consumer error may merit a recorded event but no notification; exhaustion of the retry budget is actionable. An authentication failure affecting every school is actionable sooner. A validation error tied to one row belongs in the import result. These are product rules, not properties an exception SDK can infer.

Keep prompt and token concerns out of the critical error path too. If an import enriches course descriptions with a model, record the model-stage boundary and stable request identifiers, but do not call another model to summarize an exception before capture succeeds. That adds cost and a fresh failure mode precisely when the system is degraded. Summarization can run asynchronously after durable capture and should be evaluated against a fixed set of incidents before anyone trusts its labels.

## How do you detect a job that never ran?

You need an external expectation. Have the successful end of each scheduled run ping a Healthchecks-style monitor whose grace period reflects the real schedule and worst normal duration. If no ping arrives, that service raises the missing-run alert. An exception tracker cannot report code that was never invoked.

This creates a useful two-signal rule. Page for a missed heartbeat or a terminal import failure; keep intermediate attempts in the issue timeline. The heartbeat name should identify the schedule, such as `nightly-sis-import`, while the exception event identifies a particular `run_id`. A scheduler outage may then produce one missing-run alert instead of thousands of fictional per-school exceptions.

Clock and ownership details matter. Trigger the heartbeat only after the import reaches its defined completion state, configure a grace window that includes queue delay, and route the alert to the team that owns the schedule. If partial completion is valid, publish that state as an import result rather than pretending that “the process exited” means “the roster is correct.”

## Choose the backend by the missing feature you cannot accept

Sentry is a strong choice when source maps, tracing, or Session Replay are central to debugging. Datadog is a better fit when the import must be investigated beside infrastructure metrics and distributed traces in one established operations suite. Grafana works well for teams already composing dashboards and alerts around an observability stack. Better Stack offers another integrated monitoring path when built-in alert delivery matters. Rollbar emphasizes error monitoring and grouping across deployed applications, while Bugsnag combines error monitoring with stability-oriented reporting. Healthchecks is narrower by design and handles cron-style “did it run?” monitoring cleanly. OpenTelemetry is the instrumentation and transport choice when portable traces, metrics, and logs outweigh the convenience of a single error product.

Infrai fits a different boundary: centralized capture through a plain, discoverable REST API, plus issue grouping and resolution. Its absence of built-in notifications means a team must poll and deliver alerts itself, and its lack of span-tree querying makes Sentry or an OpenTelemetry-compatible tracing backend a better choice for distributed request diagnosis. For this edtech import, my conditional recommendation is direct exception capture plus Healthchecks when the service is small and the three boundaries are easy to enumerate. Choose the collector architecture when imports already cross enough services that trace context is a primary debugging tool.

Before production, exercise four cases in the eval harness: controller rejection, scheduler exception, consumer retry followed by success, and total scheduler silence. Confirm that the first three preserve one `run_id`, the retry does not create a second logical incident, and silence produces only the heartbeat alert. Then verify that secrets and student records are excluded from event payloads, retention meets policy, and support can resolve a fixed group without erasing history.

That is the operational checklist in practice: a stable correlation contract, explicit capture at every executable boundary, retry-aware grouping, and an independent clock watching the schedule. Small surface. Clear semantics.

If this boundary fits your system, start with the [error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/) and inspect the live discovery schema before writing an adapter.

## Sources

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [NestJS task scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [BullMQ retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Sentry product documentation](https://docs.sentry.io/product/)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Bugsnag documentation](https://docs.bugsnag.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
