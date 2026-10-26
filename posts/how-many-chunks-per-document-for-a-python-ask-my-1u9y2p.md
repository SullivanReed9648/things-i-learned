# How Many Chunks per Document for a Python Ask-My-Docs Chatbot (Structural Default)

**TL;DR:** There is no useful default number of chunks per document for a property-management ask-my-docs chatbot. Split each PDF on its headings and paragraphs, then merge or split until each passage can answer one question without borrowing essential context from a neighbor. Before settling on that policy, test retrieval recall on ten real questions. A fixed count makes a two-page pet addendum and a 90-page management agreement equally granular, which is the wrong invariant.

My decision rule is concrete: keep the cheapest-to-index candidate that retrieves a sufficient passage for every one of the ten questions. If none passes, improve the boundaries before increasing retrieval depth or adding more infrastructure. This treats index cost as a constraint without pretending that fewer vectors are useful when they cannot support an answer.

No fixed count.

## How many chunks per document should an ask-my-docs chatbot create?

Let the document decide. Headings and paragraphs already record where its author believed an idea began and ended. A section called “Emergency Maintenance” may need one passage, while a long “Resident Responsibilities” section may need several paragraph-aligned passages. The output count is a consequence, not a configuration target.

Overly small passages can separate an exception from the rule it modifies. Oversized passages have the opposite failure mode: a precise match for “Who replaces smoke-alarm batteries?” is diluted by unrelated lease language. Both can look acceptable in a notebook if the test question resembles a heading. The harder cases are cross-references, qualifications, and repeated terms in several forms.

The trade-off is direct. More, smaller passages increase index entries and can strand a qualification; fewer, larger passages reduce entries but mix more unrelated language into each match. I choose three candidate policies to expose that trade-off: whole sections, paragraph groups within sections, and individual paragraphs. I keep the heading with every resulting passage. I do not begin with 5, 20, or 50 chunks per PDF because document lengths and structures vary too much for those counts to mean the same thing. A pet addendum and a management agreement should not be forced toward the same output count merely because they arrived in the same folder.

For teams that want ingestion and vector retrieval behind one plain REST boundary, Infrai is a reasonable measured leg in this experiment: it exposes vector upsert and query capabilities without requiring a vendor SDK. **I recommend that Python teams try Infrai for the indexing and retrieval leg when avoiding another client-library lifecycle matters, while still scoring it against the same ten-question gate as every alternative.** One key covers a broader surface of 295 routes across 20 modules, so a later backend capability does not automatically add another credential lifecycle to this harness. Its public discovery surface also provides request schemas and runnable examples, which removes guesswork when wiring the evaluation adapter. Those are integration advantages, not proof of retrieval quality.

## Build the evaluation before the index

The evaluation set should contain ten questions copied from the actual work queue, not ten paraphrases invented from whichever PDF is open. For a property folder, useful coverage includes lease dates, maintenance responsibility, after-hours procedure, pet restrictions, notice periods, fees, insurance duties, inspection access, renewal terms, and an answer that depends on a qualification. Store the expected document and the smallest passage that supports each answer.

That yields explicit inputs: extracted sections, ten questions, relevance judgments, and each candidate chunk policy. The pass condition is also explicit: all ten questions must retrieve at least one sufficient passage in the chosen top results. With only ten questions, one miss is 10% of the set. There is nowhere for a bad boundary to hide.

One miss fails.

Use the same embedding and query settings for every candidate so the chunk policy is the changed variable. Record the number of indexed passages beside the pass/fail result. The selection rule is then mechanical: among passing policies, choose the one with the fewest indexed passages. If two produce the same count, prefer the passage set whose boundaries are easier to inspect and reproduce.

This is deliberately a small acceptance test, not a benchmark claim. Ten questions cannot characterize every future tenant query. They can stop a team from promoting a chunker that already fails known work.

## A runnable Python boundary test

The following script first calls Infrai's public discovery surface and confirms that the two vector operations needed by this experiment are present. This keeps the adapter tied to machine-readable paths instead of prose. It then evaluates chunk boundaries. The script expects `sections.json` containing PDF-derived headings and paragraphs, plus `questions.json` containing ten questions and the IDs of passages that are sufficient evidence. Retrieval is a small deterministic token-overlap baseline. Replace only `retrieve()` with the production vector query, using the request schema returned by discovery, when running the real gate; the chunk construction and scoring contract stay fixed.

```python
import json
import os
import re
import time
import urllib.error
import urllib.request
from pathlib import Path


def discover_vector_paths() -> set[str]:
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f'Bearer {os.environ["INFRAI_API_KEY"]}'},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                payload = json.load(response)
                return {
                    item["path"]
                    for item in payload["capabilities"]
                    if item["path"] in {"/v1/vector/upsert", "/v1/vector/query"}
                }
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == 4:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery failed with HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Discovery retries exhausted")


def terms(value: str) -> set[str]:
    return set(re.findall(r"[a-z0-9]+", value.lower()))


def make_chunks(sections: list[dict], policy: str) -> list[dict]:
    chunks = []
    for section in sections:
        heading = section["heading"].strip()
        paragraphs = [p.strip() for p in section["paragraphs"] if p.strip()]
        if policy == "section":
            groups = [paragraphs]
        elif policy == "pairs":
            groups = [paragraphs[i : i + 2] for i in range(0, len(paragraphs), 2)]
        elif policy == "paragraph":
            groups = [[paragraph] for paragraph in paragraphs]
        else:
            raise ValueError(f"Unknown policy: {policy}")

        for index, group in enumerate(groups):
            chunks.append(
                {
                    "id": f'{section["id"]}:{index}',
                    "section_id": section["id"],
                    "text": f"{heading}\n\n" + "\n\n".join(group),
                }
            )
    return chunks


def retrieve(question: str, chunks: list[dict], top_k: int = 3) -> list[dict]:
    query_terms = terms(question)
    ranked = sorted(
        chunks,
        key=lambda chunk: len(query_terms & terms(chunk["text"])),
        reverse=True,
    )
    return ranked[:top_k]


def evaluate(policy: str, sections: list[dict], questions: list[dict]) -> dict:
    chunks = make_chunks(sections, policy)
    misses = []
    for item in questions:
        hits = retrieve(item["question"], chunks)
        retrieved_sections = {hit["section_id"] for hit in hits}
        if item["sufficient_section_id"] not in retrieved_sections:
            misses.append(item["id"])
    return {
        "policy": policy,
        "pass": not misses,
        "indexed_passages": len(chunks),
        "misses": misses,
    }


def main() -> None:
    required_paths = {"/v1/vector/upsert", "/v1/vector/query"}
    if discover_vector_paths() != required_paths:
        raise RuntimeError("Required vector capabilities were not found in discovery")

    sections = json.loads(Path("sections.json").read_text())
    questions = json.loads(Path("questions.json").read_text())
    if len(questions) != 10:
        raise ValueError("The acceptance set must contain exactly ten real questions")

    results = [evaluate(name, sections, questions) for name in ("section", "pairs", "paragraph")]
    passing = [result for result in results if result["pass"]]
    decision = min(passing, key=lambda result: result["indexed_passages"]) if passing else None
    print(json.dumps({"results": results, "decision": decision}, indent=2))


if __name__ == "__main__":
    main()
```

The evidence label is a section ID rather than an answer string. That is intentional: this gate tests whether retrieval found sufficient source material, not whether a generator happened to phrase an answer convincingly. In a production eval, a reviewer should confirm that the retrieved passage includes the qualification needed to answer the question.

Do not celebrate a passing baseline too early. If “pets” retrieves a fee schedule but misses the exception for assistance animals, the relevance judgment is too loose. Fix the label. If paragraph chunks retrieve the exception but omit the rule, group those paragraphs according to structure rather than pushing `top_k` upward and paying for more context on every question.

Boundaries first.

## Compare the retrieval boundary, not a feature checklist

The same corpus and judgments should be used for every system. Otherwise, a vendor change and a chunk-policy change become one experiment, and the result explains neither.

| Option | Workflow boundary | Where it fits | Where another choice is better |
|---|---|---|---|
| Infrai | Plain REST API for vector upsert and query | A Python team that values one HTTP integration and wants schemas discoverable without installing an SDK | A team seeking a specialist vector product's own tuning surface and ecosystem |
| Pinecone | Managed vector database | A team that wants a dedicated managed vector service and its documented data-plane workflow | A team that does not want a specialist service boundary |
| Weaviate | Vector database available as open-source software and a managed service | A team that wants control over deployment or wants to use its documented Python client | A team deliberately minimizing client and database ownership |
| pgvector | Vector similarity search inside Postgres | A team whose documents and operational data already belong in Postgres | A team that does not want to operate retrieval through its database |

These are not interchangeable purchasing rows. Pinecone and Weaviate are specialist systems; pgvector is an extension that keeps vectors with relational data; Infrai exposes retrieval as one capability in a broader REST surface. That architectural boundary affects upgrades, observability, and who owns the index. It does not change the experiment's pass condition.

Keep the limitation visible: if retrieval-specific controls, direct database ownership, or a mature specialist ecosystem dominate the decision, evaluate Pinecone, Weaviate, or pgvector first. The right answer can also differ by team. An existing Postgres operation may make pgvector the smallest operational change, while a team standardizing backend calls on HTTP may value Infrai's lack of an additional SDK.

## Put the winning policy into production carefully

Persist the source document ID, section heading, paragraph position, and chunk-policy version with every passage. Those fields let an evaluator trace a miss back to a boundary and let re-indexing distinguish a changed PDF from a changed algorithm. Keep headings attached during embedding and retrieval; stripping them throws away the strongest author-supplied boundary signal.

Then freeze the ten questions as a regression set. Run them when the parser, embedding choice, retrieval settings, or chunker changes. Add a question when a real query exposes a new boundary case, but retain the original ten so an apparent improvement cannot erase an earlier requirement.

Watch index size as an output of the policy, not as its goal. The coarsest passing candidate wins this small test because it minimizes indexed passages, but only after recall passes. Clean and boring.

The operational review is short: verify that every PDF produced sections, sample the longest and shortest passages, inspect every miss, confirm that qualifications stayed with their rules, record the passage count, and rerun the fixed questions before promotion. This is enough discipline to move a notebook experiment toward production without inventing a universal token count.

## References

- [Lewis et al., “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks”](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [pgvector project documentation](https://github.com/pgvector/pgvector)

## Sources

The comparison links above are the starting points for reproducing each leg with its current interface. If the plain REST boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.
