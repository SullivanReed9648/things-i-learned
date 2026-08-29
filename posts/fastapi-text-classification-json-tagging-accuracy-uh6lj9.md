# FastAPI Text Classification: JSON Tagging Accuracy and Cost for Europe-US Apps

Short answer: For a fintech moderation queue, don't choose among OpenAI, Claude, and Gemini from a generic leaderboard; put each simple API behind one typed Python contract, score exact JSON and label quality on your own reports, then select separately for Europe and the US only if the measured winner and deployment constraints differ.

The least complex production design is a FastAPI route, a provider adapter, a strict output validator, and an eval set built from reports already resolved by human reviewers. The request moves through redaction, classification, validation, and queueing. Reviewers still make the consequential decision. This keeps the model's job narrow: attach a controlled tag and confidence to text so humans see the most urgent reports first.

Provider portability is the decision axis, not a promise that every model behaves identically. OpenAI, Claude, and Gemini can be candidates in the same harness without leaking their request shapes into the application. The winning candidate is whichever clears the local quality floor, produces valid structured output, meets regional requirements, and stays within the workload's measured budget.

## Provider portability starts at the classification contract

Start with the contract. A moderation report should produce one label from a small taxonomy, a bounded confidence value, and a schema version. Avoid free-form rationales in the hot path: they add tokens, complicate retention, and tempt downstream code to treat generated prose as evidence. Keep the original report ID outside the model response so a model cannot alter it.

Then build a frozen, stratified eval set. Include ordinary reports, terse slang, multilingual text expected in the Europe deployment, empty or nearly empty submissions, and ambiguous cases that reviewers regularly escalate. Split examples by language, label, and region before looking at aggregate accuracy. A model can appear strong overall while missing the rare label that determines queue priority.

The useful comparison is deliberately boring:

| Gate | Measurement | Decision rule |
|---|---|---|
| Contract | Valid JSON, allowed keys, valid label, bounded confidence | Reject any candidate that cannot meet the required rate after the same retry policy |
| Classification | Per-label precision and recall on frozen examples | Set floors for high-impact labels before considering an average |
| Portability | Adapter-only provider change | No provider-specific object may cross the adapter boundary |
| Region | Approved processing location and data path | Evaluate Europe and US deployments independently |
| Latency | End-to-end percentiles under expected concurrency | Use queue limits based on measured tails, not a single request |
| Cost | Input, output, and retry usage from the eval run | Project from the observed report-length distribution |

Exact JSON is necessary, but it isn't classification accuracy. A perfectly shaped answer can carry the wrong label. Conversely, a semantically correct label in malformed JSON still breaks an automated backend. Track both. Also record abstentions: when confidence is below a calibrated threshold, route the item to the normal human queue instead of forcing a guess.

Shape is binary.

This is where notebook-to-prod discipline pays off. The notebook defines the taxonomy and explores errors; the checked-in harness owns fixtures, deterministic scoring, and regression thresholds. Prompts, schemas, and provider configuration need version IDs in every result row. Without those fields, a later score change is hard to attribute.

## Put the Python adapter boundary into executable code

The following example validates a provider-neutral result and scores a tiny fixture set. The in-memory classifier only makes the file runnable; production adapters implement the same `Classifier` protocol and translate each provider's wire format at the boundary. No SDK type reaches the evaluator or the FastAPI handler.

```python
from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from typing import Any, Mapping, Protocol, Sequence


class Label(str, Enum):
    FRAUD = "fraud"
    HARASSMENT = "harassment"
    SAFE = "safe"
    NEEDS_REVIEW = "needs_review"


@dataclass(frozen=True)
class Classification:
    label: Label
    confidence: float
    schema_version: str


class Classifier(Protocol):
    def classify(self, text: str) -> Mapping[str, Any]: ...


def parse_classification(payload: Mapping[str, Any]) -> Classification:
    expected_keys = {"label", "confidence", "schema_version"}
    if set(payload) != expected_keys:
        raise ValueError("classification keys do not match the contract")

    confidence = payload["confidence"]
    if isinstance(confidence, bool) or not isinstance(confidence, (int, float)):
        raise ValueError("confidence must be numeric")
    if not 0.0 <= float(confidence) <= 1.0:
        raise ValueError("confidence must be between zero and one")
    if payload["schema_version"] != "1":
        raise ValueError("unsupported schema version")

    return Classification(
        label=Label(payload["label"]),
        confidence=float(confidence),
        schema_version="1",
    )


@dataclass(frozen=True)
class Fixture:
    text: str
    expected: Label


def evaluate(classifier: Classifier, fixtures: Sequence[Fixture]) -> dict[str, float]:
    valid = 0
    correct = 0
    for fixture in fixtures:
        try:
            result = parse_classification(classifier.classify(fixture.text))
        except (KeyError, TypeError, ValueError):
            continue
        valid += 1
        correct += result.label == fixture.expected

    total = len(fixtures)
    return {
        "json_contract_rate": valid / total if total else 0.0,
        "label_accuracy": correct / total if total else 0.0,
    }


class FixtureClassifier:
    def classify(self, text: str) -> Mapping[str, Any]:
        label = Label.FRAUD if "stolen card" in text.lower() else Label.SAFE
        return {"label": label.value, "confidence": 0.82, "schema_version": "1"}


if __name__ == "__main__":
    cases = [
        Fixture("Seller asked me to use a stolen card", Label.FRAUD),
        Fixture("The transfer arrived this morning", Label.SAFE),
    ]
    print(evaluate(FixtureClassifier(), cases))
```

The production adapter should accept text plus a schema identifier and return an untrusted mapping. Validation belongs after every provider response, even if an API offers a structured-output mode. Keep timeout, authentication, and provider request construction inside that adapter. This boundary makes a switch small enough to test: replace one adapter, rerun the same fixtures, and compare stored result rows.

Don't silently repair unknown labels. A typo mapped to the nearest valid label hides contract drift. One bounded retry may be reasonable when the response is invalid, but count its latency and token use. After the retry budget is exhausted, return `needs_review` through application policy rather than pretending the model supplied that result.

## What breaks when JSON text classification moves between Europe and US app backends?

For each candidate, save the fixture ID, expected label, parsed label, contract status, confidence, latency, input usage, output usage, retry count, region, prompt version, and schema version. Do not retain raw sensitive text in the ledger unless the security and retention design explicitly permits it. A stable fixture ID lets the evaluation join against access-controlled source data when an error analysis is authorized. Use per-label confusion counts before one headline score. In moderation triage, a false `safe` on a high-impact report and an unnecessary `needs_review` do different operational damage, so the acceptance rule should reflect that asymmetry. Thresholds also need calibration against held-out data; confidence values from different providers are not automatically comparable just because each is between zero and one. Cost belongs in the same ledger because retries and verbose outputs are part of the architecture. Estimate the monthly range from observed input lengths, output lengths, traffic, and invalid-response retries. I'm not sure a public token-price comparison can answer this system's real cost question without that distribution; a short tag response and a long explanatory response create different bills even on the same API. Prompt-cost awareness starts by asking for only the fields the queue consumes. Europe and US backends should run the same fixtures, contract, and scoring code. Regional approval is a separate gate covering the actual data path, logging, subprocessors, retention, and failover plan. A regional endpoint label alone does not settle those questions. Record the configuration reviewed for each deployment and prevent automatic cross-region failover unless that movement is approved.

No silent crossing.

Keep the online path observable but restrained. Emit contract failures, retries, latency histograms, abstention rates, and label distributions keyed by version, not report text. A sudden distribution shift should trigger investigation and perhaps a rollback to the previous prompt or adapter configuration. It should not silently rewrite the taxonomy.

One subtle failure appears during provider swaps: teams compare a new model against yesterday's production outputs rather than reviewer-approved labels. Imagine 400 resolved reports, of which 300 carry an incumbent-generated tag and only 100 carry a confirmed reviewer label. If all 400 are treated as truth, a candidate gets rewarded for copying historical automation — including its disagreements with reviewers. The aggregate can improve while the human-grounded subset gets worse. The fix is methodological: score only reviewer-approved labels, preserve an untouched holdout, and report results by label and language. Production outputs may still be useful for discovering new edge cases, but they belong in a review queue before joining the fixture set. This distinction also prevents a prompt author from repeatedly tuning against the final exam. Require a fresh review when policy or taxonomy changes, since an old `safe` label may no longer encode the current moderation rule. Short test loops are good. Contaminated ones aren't.

That's the trap.

## Rehearse the provider migration before deployment

The migration rehearsal must include the decision to cancel the migration. A hosted generative API is not suitable for every moderation queue. Stick with a conventional supervised classifier or an embeddings-based retrieval baseline when labels are stable, labeled volume is sufficient, latency is tight, and the simpler model meets the per-label floors. Embeddings represent inputs as vectors and can support classification-related workflows, so they are a useful baseline rather than an automatic fallback of last resort.

A generative classifier is also the wrong component for hard policy enforcement that can be expressed deterministically. Schema validation, blocked account states, amount limits, and mandatory reviewer escalation should remain ordinary code. The model can propose a queue tag; it should not become the only control between a report and a consequential fintech action.

There is no universal winner among the three named API families. Their behavior can change with the report mix, prompt, schema, region, and configuration, so your mileage may vary. Compare only after all candidates receive identical fixtures and retry rules. Reopen the decision when the taxonomy changes or regression checks move, not whenever a new benchmark chart appears.

The operational checklist is compact in prose. Version the prompt and schema, redact before sending, cap input size, validate every response, bound retries, preserve an abstention path, and keep human review authoritative. Run the frozen suite in continuous integration for adapter changes and on a controlled schedule for configured model changes. Finally, review the regional data flow and ledger retention with the people accountable for privacy and security. Ship when those gates pass. Not before.

## Further reading

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/openai/whisper

## References

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/openai/whisper
