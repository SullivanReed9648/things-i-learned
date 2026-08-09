# Fail-Closed Backend: Typed JSON Moderation for Node.js User Content

## TL;DR

Put a durable queue between the Node.js write path and moderation, require each chat completion to match a versioned JSON decision schema, and send ambiguous or unscored user-generated content to review. The best simple backend is the one whose decisions can be replayed against an eval set before a prompt, policy, model, or threshold change reaches production.

Don't let a model response become a database action directly.

The tempting implementation is synchronous: receive a post, call a model, parse `allow` or `remove`, then return. It is short enough for a notebook and brittle enough to hide the questions that matter in production. What happens after a timeout? Can the same message be classified twice without creating two review tasks? Which policy version produced yesterday's removal? How does the team compare a new prompt without sending real posts through it again?

Treat moderation as a decision pipeline instead. The API stores the content, emits an immutable job, and returns a pending state. A worker produces a typed decision envelope. Deterministic policy maps that envelope to `allow`, `remove`, or `review`, while every transport error, invalid response, missing context, and low-confidence result fails closed into the review queue. This is less exciting than prompt tweaking. Good.

## How should a Node.js backend route JSON chat completions into a review queue?

Keep four boundaries explicit: ingestion, scoring, policy, and action. Node.js can continue to own authentication, rate limits, and the user-facing write request; nothing about this design requires moving the application backend to Python. The queue message is the language-neutral boundary between that API and the Python scoring worker.

The ingestion boundary should assign a content ID, preserve the unmodified text, record the applicable policy version, and create one moderation job. Use a deduplication key derived from the content ID and policy version so a delivery retry does not create another logical decision. The job should contain references rather than a sprawling copy of account data; the worker needs only the context the policy permits it to inspect.

The scoring boundary calls a chat-completions-compatible endpoint and validates the returned JSON locally. A response is evidence, not an action. The policy boundary then applies thresholds and any deterministic rules. Finally, the action boundary commits one state transition and, for `review`, creates one queue item with the evidence a moderator needs.

That separation pays off during notebook-to-prod work. An experiment can cache decision envelopes and replay the policy function without another model call, which keeps threshold sweeps prompt-cost aware. A production deploy can change queue handling without changing the classifier. An evaluator can compare candidate prompts on the same frozen inputs, while the live service remains boring.

A compact contract is enough:

| Field | Purpose | Reject or review when |
| --- | --- | --- |
| `decision` | Model's proposed disposition | Value is outside the declared enum |
| `category` | Policy label used for audit and routing | Label is unknown to this policy version |
| `confidence` | Ranking signal for threshold experiments | Missing, nonnumeric, or outside 0 to 1 |
| `evidence` | Short excerpt supporting human verification | Empty or unrelated to the submitted text |
| `policy_version` | Connects the result to an eval snapshot | It differs from the queued job |

Confidence deserves restraint. A number emitted by a model is not automatically a calibrated probability, so use it as a candidate ranking signal until held-out labels show otherwise. The useful experiment is a threshold sweep over cached results: for each pair of allow/remove thresholds, calculate harmful content allowed, benign content removed, review rate, and tokens consumed. Pick a point constrained by safety targets and actual reviewer capacity, not by a pleasing round number.

## A focused Python worker that fails closed

The worker below uses a pseudonymous gateway and a standard chat completions route. It deliberately has one outcome for an unanswered scoring request: `review`. A `429`, a timeout, invalid JSON, or a schema mismatch never turns into `allow`. Retries belong in queue delivery policy, where backoff and attempt counts are observable, instead of inside a recursive request wrapper.

```python
import os
from typing import Literal

import requests
from pydantic import BaseModel, ConfigDict, Field, ValidationError


GATEWAY_URL = os.environ["MODERATION_GATEWAY_URL"].rstrip("/")
API_KEY = os.environ["MODERATION_API_KEY"]
MODEL = os.environ["MODERATION_MODEL"]


class Decision(BaseModel):
    model_config = ConfigDict(extra="forbid")

    decision: Literal["allow", "remove", "review"]
    category: Literal["none", "harassment", "spam", "self_harm", "sexual", "violence"]
    confidence: float = Field(ge=0.0, le=1.0)
    evidence: str = Field(min_length=1, max_length=240)
    policy_version: str


def score_content(text: str, policy: str, policy_version: str) -> Decision:
    payload = {
        "model": MODEL,
        "temperature": 0,
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify the submitted user content under the supplied policy. "
                    "Use review when context is missing or the decision is ambiguous. "
                    "Return one JSON object and no surrounding prose.\n\n"
                    f"POLICY VERSION: {policy_version}\nPOLICY:\n{policy}"
                ),
            },
            {"role": "user", "content": text},
        ],
    }
    response = requests.post(
        f"{GATEWAY_URL}/v1/chat/completions",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json=payload,
        timeout=(3.0, 20.0),
    )
    response.raise_for_status()
    raw_content = response.json()["choices"][0]["message"]["content"]
    decision = Decision.model_validate_json(raw_content)
    if decision.policy_version != policy_version:
        raise ValueError("policy version mismatch")
    return decision


def moderate(job: dict, policy: str) -> dict:
    try:
        decision = score_content(
            text=job["text"],
            policy=policy,
            policy_version=job["policy_version"],
        )
        return {
            "content_id": job["content_id"],
            "job_id": job["job_id"],
            "status": decision.decision,
            "category": decision.category,
            "confidence": decision.confidence,
            "evidence": decision.evidence,
            "policy_version": decision.policy_version,
        }
    except (KeyError, ValueError, ValidationError, requests.RequestException):
        return {
            "content_id": job.get("content_id"),
            "job_id": job.get("job_id"),
            "status": "review",
            "category": "unscored",
            "confidence": None,
            "evidence": "Automated decision unavailable",
            "policy_version": job.get("policy_version"),
        }
```

The exception path is intentionally broad at the worker boundary, but it should not be silent. Emit a structured event with the exception class, model identifier, latency, attempt number, job ID, and policy version; never log the raw text by default. Queue redelivery should use the stable job ID, and the database write should enforce uniqueness on that ID. Those two details turn at-least-once delivery from a duplicate-moderation problem into a routine retry.

There is one subtle failure worth testing explicitly. In a synthetic queue test, submit content ID `post_1042` under policy `ugc-7`, deliver job `mod_1042_ugc7` twice, and make the fake endpoint return status `200` with prose wrapped around the JSON. Transport monitoring stays green, but local validation must reject both payloads. The first delivery should create one review item with `schema_invalid`; the second should encounter the same job key and leave that item unchanged. Now replay the job with valid JSON after the queue's backoff interval. The stored state should move through the one transition the policy permits, with the earlier diagnostic event still attached for audit, rather than producing a second action. Repeat the test with `429`, a read timeout, an unknown category, confidence `1.4`, and policy version `ugc-6`. Every case should preserve the content, avoid an automatic allow or remove, and expose a distinct operational reason without copying raw user text into ordinary logs. Keep a malformed body in a restricted diagnostic store only if the retention policy permits it. This one exercise checks schema enforcement, fail-closed routing, idempotency, redelivery, version binding, and privacy boundaries together. A green HTTP chart is not a moderation-quality metric.

## Evaluate the whole path, not just the prompt

Start with a frozen, versioned set of policy examples. Include clear allows and removals, but spend annotation effort on quoted abuse, reclaimed language, coded harassment, multilingual text, prompt injection, missing conversational context, and policy categories that annotators themselves dispute. Keep disagreement visible. Collapsing every difficult label into a forced consensus makes the offline score look cleaner while erasing the exact traffic the review queue exists to catch.

For each candidate, store the raw input reference, expected disposition, predicted envelope, prompt hash, model ID, policy version, latency, and token usage. Embeddings can help retrieve near-duplicate historical decisions or organize an annotation set, but semantic similarity is not a safety verdict; the OpenAI embeddings guide describes embeddings as vector representations useful for search, clustering, recommendations, anomaly detection, and classification. If retrieval supplies prior examples to the classifier, freeze and version that retrieval corpus too. Otherwise a prompt comparison quietly becomes a prompt-plus-data comparison.

Measure false allows and false removals by policy category, not only as one aggregate. Add schema-valid rate, unscored rate, review rate, queue age at the 50th and 95th percentiles, appeals, tokens per item, and duplicate-action count. The last metric should remain zero. Cost belongs beside quality and capacity because a prompt that consumes twice the tokens for the same error profile is a worse production choice, but cost alone cannot choose a safety boundary.

Run three tests before rollout. First, replay the labeled set and reject candidates that violate category-specific safety constraints. Second, shadow the candidate on live traffic without changing user-visible state, then inspect drift and queue projections. Third, canary the complete action path with a rollback keyed to policy and prompt versions. Model output changes are only one source of drift; traffic mix, policy edits, and reviewer behavior move too.

I'm not sure any static benchmark can settle the hardest context-dependent cases. A double-annotated sample from the application's own traffic, plus adjudication guidance and appeal outcomes, is the evidence that can reduce that uncertainty.

## Limits and the decision to keep it simpler

This architecture is not suitable for every surface. If a message must become visible in a few milliseconds, an asynchronous model decision cannot be the only pre-publication control; use deterministic synchronous checks, product friction, and post-publication review according to the risk. Image, audio, and video moderation need modality-specific inputs and evaluators. Legally regulated reporting workflows need specialist review and formal procedures beyond a generic JSON classifier.

The catch is operational: routing uncertainty to people creates a capacity constraint. When queue age rises, safety latency rises with it, so staffing, prioritization, and overload policy are part of the backend design. A low review rate is not automatically good; it can mean the classifier is confident, or it can mean failures are being dropped. Track both review volume and unscored outcomes.

Stick with deterministic rules when the policy can be expressed precisely as account state, exact matches, rate limits, or known identifiers. Use a self-hosted gateway when deployment control, provider routing, or a unified compatible interface matters enough to own its operations; LiteLLM is one open-source example of that gateway pattern. Use a managed endpoint when the team would rather outsource that operational layer. Neither choice removes the need for local schema validation, eval replay, idempotent actions, and a staffed review path.

Before copying this design, measure the labeled error trade-offs, schema-valid rate, token use, projected review volume, and the age distribution the moderation team can sustain. Those numbers determine whether the pipeline is simple in production, not the number of lines in the request handler.

## Further reading

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/BerriAI/litellm
