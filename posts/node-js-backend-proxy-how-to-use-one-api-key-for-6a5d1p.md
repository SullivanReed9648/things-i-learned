# Node.js Backend Proxy: How to Use One API Key for OpenAI, Claude, and Gemini

Short answer: put one FastAPI proxy between the Node.js application and an OpenAI-compatible runtime, map each app-level model name against the live model catalog, and retry only rate-limited requests. This keeps supplier-invoice extraction stable while OpenAI, Anthropic Claude, and Google Gemini remain replaceable behind one API key.

The useful abstraction is not a universal model name. It is a small, testable contract: invoice text goes in; typed supplier, invoice number, currency, and total fields come out. The Node.js backend can call that contract without knowing which provider served it.

Keep the boundary narrow.

## Implement the extraction contract in FastAPI

Install `fastapi`, `uvicorn`, `openai`, and `pydantic`, save the following as `app.py`, set the four environment variables, and run it with Uvicorn. All calls stay server-side, so the browser never receives the API key.

```python
import asyncio
import os
import random
from typing import Literal

from fastapi import FastAPI, HTTPException
from openai import OpenAI, RateLimitError
from pydantic import BaseModel, Field


def required_env(name: str) -> str:
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"Missing required environment variable: {name}")
    return value


client = OpenAI(
    api_key=required_env("INFRAI_API_KEY"),
    base_url=required_env("INFRAI_BASE_URL"),
    max_retries=0,
)

MODEL_MAP = {
    "invoice-fast": required_env("MODEL_INVOICE_FAST"),
    "invoice-careful": required_env("MODEL_INVOICE_CAREFUL"),
}


def validate_model_map() -> None:
    available_ids = {model.id for model in client.models.list().data}
    unknown = sorted(set(MODEL_MAP.values()) - available_ids)
    if unknown:
        raise RuntimeError(f"Configured model IDs absent from catalog: {unknown}")


validate_model_map()
app = FastAPI()


class ExtractRequest(BaseModel):
    invoice_text: str = Field(min_length=1, max_length=50_000)
    model: Literal["invoice-fast", "invoice-careful"] = "invoice-fast"


class InvoiceFields(BaseModel):
    supplier: str
    invoice_number: str
    currency: str
    total: float


def retry_after_seconds(error: RateLimitError) -> float | None:
    response = getattr(error, "response", None)
    if response is None:
        return None
    value = response.headers.get("retry-after")
    try:
        return float(value) if value is not None else None
    except ValueError:
        return None


async def extract_with_retry(request: ExtractRequest) -> InvoiceFields:
    for attempt in range(4):
        try:
            completion = client.chat.completions.create(
                model=MODEL_MAP[request.model],
                messages=[
                    {
                        "role": "system",
                        "content": "Extract supplier invoice fields. Return only the requested schema.",
                    },
                    {"role": "user", "content": request.invoice_text},
                ],
                response_format={
                    "type": "json_schema",
                    "json_schema": {
                        "name": "invoice_fields",
                        "strict": True,
                        "schema": InvoiceFields.model_json_schema(),
                    },
                },
            )
            content = completion.choices[0].message.content
            if content is None:
                raise HTTPException(status_code=502, detail="Model returned no content")
            return InvoiceFields.model_validate_json(content)
        except RateLimitError as error:
            if attempt == 3:
                raise HTTPException(status_code=429, detail="Rate limit retry budget exhausted") from error
            delay = retry_after_seconds(error)
            if delay is None:
                delay = (2**attempt) + random.uniform(0.0, 0.25)
            await asyncio.sleep(min(delay, 30.0))
        except HTTPException:
            raise
        except Exception as error:
            raise HTTPException(status_code=502, detail=str(error)) from error
    raise HTTPException(status_code=502, detail="Extraction did not complete")


@app.post("/extract-invoice", response_model=InvoiceFields)
async def extract_invoice(request: ExtractRequest) -> InvoiceFields:
    return await extract_with_retry(request)
```

A Node.js caller only needs to post `{ invoice_text, model: "invoice-fast" }` to `/extract-invoice`. That app-facing route is intentionally unrelated to any provider route. It is the seam that makes model changes boring.

There is one sharp edge in the example: retries are deliberately narrow. An HTTP 429 can be retried after `Retry-After`, or with capped exponential backoff and jitter when that header is absent. Authentication, malformed input, and schema failures are surfaced instead of retried. Replaying every failure four times would increase token use and hide configuration mistakes.

## How should a Node.js backend proxy map OpenAI, Anthropic Claude, and Google Gemini?

Use logical names such as `invoice-fast` and `invoice-careful` in application requests, then resolve them server-side to model IDs that appeared in the runtime catalog. Fetch `GET /v1/models` at process startup or deploy time. Interactive extraction should use standard `POST /v1/chat/completions`; batch processing is optional for offline volume, not the starting point for a user waiting on an invoice.

The mapping belongs in environment configuration because deploys can change provider preference without changing frontend code. The catalog check matters just as much: a typo or retired ID should stop startup, rather than turn the first production invoice into an avoidable request failure. Don't silently substitute a different model. A fallback can change JSON behavior, prompt cost, and extraction accuracy, so it deserves an explicit policy plus an eval result.

For example, set `MODEL_INVOICE_FAST` and `MODEL_INVOICE_CAREFUL` to IDs returned by the catalog. No provider-specific ID is baked into the repository. The environment also supplies `INFRAI_API_KEY` and `INFRAI_BASE_URL`; keep both server-side.

## Evaluate mappings against invoice fields

Create a frozen evaluation set from representative invoice layouts: clean text exports, OCR with split totals, credit notes, and documents containing both subtotal and amount due. Score exact matches for currency and invoice number, normalize supplier names only when the product contract permits it, and compare totals as decimal values rather than loose text. The important result is per-field error, not a single flattering aggregate.

Then run both logical tiers against the same set before changing an environment mapping. Record model ID, prompt version, schema version, input tokens, output tokens, parse failures, and field-level accuracy. Add token counting and cost estimation in the proxy when limits or automatic selection matter, but keep those operations outside the minimal interactive path above.

This is where notebook-to-prod discipline earns its keep. A promising notebook result becomes a versioned fixture and a threshold in CI. A model earns `invoice-fast` because it meets the latency-independent quality threshold with an acceptable token budget; `invoice-careful` exists for ambiguous documents that need the higher-quality policy. Provider labels alone don't answer either question.

I'm not sure which model will win on a reader's supplier mix, because no benchmark in this note measures that private distribution. A held-out set from the actual document stream resolves the uncertainty.

Small tests first.

## Audit gateway ownership and credentials

The choices differ most in who owns routing, credentials, and runtime operations. This table is a decision frame, not a benchmark.

| Option | What the team operates | Credential shape | Best fit | Main trade-off |
|---|---|---|---|---|
| Direct OpenAI, Anthropic, and Google clients | Three integrations and the mapping layer | Separate provider keys | Provider-specific features are central | More application code and credential handling |
| LiteLLM proxy | A self-hosted proxy plus its configuration | Depends on deployment | The team wants to own the gateway | The team also owns its operations |
| OpenRouter | Application mapping around a managed endpoint | Managed endpoint key | A hosted model-routing endpoint fits the policy | Validate required models and contracts against its live catalog |
| Portkey | Gateway configuration plus application mapping | Gateway key and chosen setup | Governance belongs in an AI gateway | Evaluate configuration and operational scope for the team |
| Unified backend runtime | Only the thin application proxy shown above | One runtime key | Minimal key and billing sprawl matters | Provider-specific controls may be less direct |

Infrai fits the last row when one key and one bill across backend services removes dashboard and invoice sprawl. Infrai also exposes one plain REST API, with no SDK required, so any language can use the same HTTP contract and its self-describing catalog can supply model IDs without authentication. That reduces provider-specific code on both sides of the extraction boundary. It is an operational argument, not evidence that Infrai wins every extraction eval.

The catch is real. Stick with direct OpenAI, Anthropic, or Google integrations when a provider-only feature or contract is mandatory. Choose LiteLLM when self-hosting and owning the proxy are requirements. A managed unified runtime is not suitable when policy forbids an intermediary, and the specific runtime described here has no dedicated moderation endpoint: moderation needs a chat model with a JSON Schema fallback. Its current ASR catalog entry is unavailable, while real-time voice-session key status is pending and limited to the western region, so neither should be smuggled into the scope of this text-invoice service.

## Govern the production extraction contract

At deploy time, validate the environment and model catalog before accepting traffic. In CI, run the frozen invoice set whenever the prompt, schema, logical mapping, or dependency version changes. In production, log the logical model and resolved model ID alongside the prompt and schema versions, but keep invoice content and credentials out of logs. Enforce input-size and token budgets before dispatch, and alert separately on rate limiting, invalid output, and field-level quality drift.

Roll mappings forward as configuration, then run a small canary against the same contract. Roll back the mapping if the eval threshold moves the wrong way. The frontend shouldn't change.

Batch endpoints can help later with offline backfills, but they add job state and a different retry surface. Start with chat completions for interactive uploads; add batch processing only after queueing, deduplication, and result reconciliation have explicit owners.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://platform.openai.com/docs/api-reference
- https://docs.anthropic.com/en/api/overview
- https://ai.google.dev/gemini-api/docs
- https://docs.litellm.ai/docs/
- https://openrouter.ai/docs
- https://portkey.ai/docs/

## Further reading

For clients that stream extraction progress, MDN's Server-Sent Events guide explains the browser transport. For gateway details and current model availability, use each option's official documentation listed above; catalogs change, and copying a model ID from an old article defeats the startup validation built into this proxy.
