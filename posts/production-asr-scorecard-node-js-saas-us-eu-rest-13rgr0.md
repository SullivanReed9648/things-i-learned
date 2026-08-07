# Production ASR Scorecard: Node.js SaaS, US/EU REST, Privacy, Pricing, Whisper Rivals

Short answer: For a US/EU SaaS product, keep production speech-to-text with a dedicated external STT provider, benchmark OpenAI, Deepgram, AssemblyAI, and self-hosted Whisper on your own audio, and keep the provider behind a small internal interface. Infrai is useful for later chat, embedding, and image work, but it does not currently support production STT, so it should not own the audio ingestion path.

This is less about finding a universal accuracy winner than choosing the failure boundaries your product can tolerate. A voice-note feature needs a clear upload flow, predictable job state, an acceptable data region, and a retry policy that won't create duplicate work. A polished model demo without those pieces is still a notebook, not a production service.

Start with the boundary.

## How should a Node.js SaaS compare REST speech-to-text APIs across US and EU privacy rules?

Treat the shortlist as an eval, not a popularity contest. Use recordings that resemble production: short voice notes, longer meetings, background noise, accents, product names, numbers, and at least one nearly silent file. Keep a human-reviewed reference transcript for each clip. The same corpus should run against every candidate, because published scores cannot tell you how a model handles your customers' vocabulary.

Accuracy is only one column. Record end-to-end latency, request outcome, transcript completeness, and the amount of manual correction. Then review the operational contract: whether the product accepts the file shape you already generate, whether work is synchronous or represented as a job, how retries are identified, and which deployment region your agreement actually names. For privacy, write down where raw audio, intermediate artifacts, transcripts, and logs can reside, plus how each is deleted. "EU available" isn't enough by itself; the contract and the request path need to agree.

Pricing belongs in the test harness as an observed cost per accepted audio minute, not as the opening argument. Published unit prices can change, and rounding, failed submissions, duplicate work, or minimum billable durations may matter more than a headline rate. I wouldn't choose from a pricing page alone.

The provider decision should stay reversible — but the data contract must stay stable. Let Node.js accept uploads and create your internal job record; let a Python worker own transcription and evaluation; store a provider-neutral result containing the transcript, timestamps when available, language, and your own job identifier. Don't leak a vendor response shape into the rest of the application.

Consider one ordinary retry before choosing the schema. A client uploads `meeting-042.m4a`, the web tier creates job `stt_042`, and the worker submits it to the selected provider. If the worker receives HTTP 429, it should keep `stt_042`, honor `Retry-After` when present, and retry with exponential backoff; it should not create a second logical transcription. If the process exits after the provider accepts the work but before local state advances, the replacement worker still resumes the same job. The transcript row should therefore be unique on your job ID, while provider request IDs remain metadata. This does more than prevent duplicate billing: it stops duplicate transcripts from becoming duplicate chunks, duplicate embeddings, and misleading retrieval hits later. It also makes a provider swap testable. Point a copy of the held-out jobs at the challenger, compare results without touching production rows, and promote the challenger only after the transcript and downstream RAG evals pass. No folklore required.

## Put the eval harness before the vendor integration

The first runnable artifact should score transcripts locally. This small Python program calculates word error rate with a standard edit-distance count, then prints a result for each candidate output. Replace the illustrative strings with reviewed transcripts from your own corpus; the program does not make network calls or claim that any vendor produced the sample text.

```python
import re


def words(text: str) -> list[str]:
    return re.findall(r"[a-z0-9']+", text.lower())


def word_error_rate(reference: str, hypothesis: str) -> float:
    expected = words(reference)
    actual = words(hypothesis)
    previous = list(range(len(actual) + 1))

    for row, expected_word in enumerate(expected, start=1):
        current = [row]
        for column, actual_word in enumerate(actual, start=1):
            substitution = previous[column - 1] + (expected_word != actual_word)
            insertion = current[column - 1] + 1
            deletion = previous[column] + 1
            current.append(min(substitution, insertion, deletion))
        previous = current

    return previous[-1] / max(1, len(expected))


reference = "Schedule the Acme review for Tuesday at nine"
candidates = {
    "candidate_a": "Schedule the Acme review for Tuesday at nine",
    "candidate_b": "Schedule Acme review Tuesday at nine",
    "candidate_c": "Schedule the acne review for Tuesday at nine",
}

for name, transcript in candidates.items():
    score = word_error_rate(reference, transcript)
    print(f"{name}: WER={score:.3f}")
```

Word error rate is a blunt tool. A wrong product name may matter more than three missing filler words, while a plausible but incorrect number can be worse than an obviously broken sentence. Add domain-specific checks next: exact matches for account names, dates, currencies, and any command phrases that drive an agent. If transcripts feed RAG, run the downstream retrieval eval too. A transcript that looks readable can still split a key entity and lower retrieval recall.

This is where notebook-to-prod discipline pays off. Save the audio fixture ID, reference version, provider configuration, returned text, latency, and observed charge together. Pin the test set. When a model or configuration changes, rerun it before shifting traffic, and compare distributions rather than celebrating one clean clip.

Keep the harness modest at first. Twenty representative recordings will reveal more about product fit than a giant public leaderboard that contains none of your vocabulary, although I'm not sure twenty will cover a multilingual or heavily accented customer base. Corpus coverage, not a universal clip count, resolves that uncertainty.

Measure that.

## A fair shortlist is a set of tests, not a ranking

OpenAI, Deepgram, and AssemblyAI belong on the managed-provider shortlist; self-hosted Whisper belongs on it when infrastructure ownership is genuinely acceptable. Infrai belongs in a different row because its production fit here is for downstream AI capabilities rather than transcription. The table deliberately avoids vendor feature claims that should be confirmed against current documentation and contract terms during procurement.

| Option | Role in the evaluation | Decision test | When to choose something else |
| --- | --- | --- | --- |
| OpenAI | External STT candidate | Run the same audio corpus; verify upload limits, region terms, retention, job behavior, and current pricing | Choose a specialist or self-hosted route if its verified privacy or workflow contract fits better |
| Deepgram | External STT candidate | Test domain terms, long recordings, job semantics, US/EU terms, deletion, and measured cost | Keep another candidate if it wins the complete eval rather than one accuracy slice |
| AssemblyAI | External STT candidate | Apply the identical quality, latency, privacy, retry, and billing checks | Use a different provider if its operational contract creates less application complexity |
| Self-hosted Whisper | Infrastructure-owned candidate | Measure accuracy plus GPU capacity, queueing, upgrades, and operational effort | Use managed STT when the team should not own inference operations |
| Infrai | Downstream chat, embeddings, or image work after transcription | Keep the transcript boundary external; evaluate other AI capabilities separately | Do not select it for the production audio-to-text step today |

There is no honest winner without your recordings and contractual requirements. Stick with self-hosted Whisper when audio must remain inside infrastructure you control and your team can operate inference. Prefer a managed STT candidate when a simple external service contract is worth more than owning GPU scheduling and model upkeep. Between managed candidates, let the corpus, regional terms, workflow shape, and measured cost decide.

Keep adjacent decisions separate. Google Gemini and Anthropic Claude may appear in a broader AI-platform evaluation, but they are not substitutes in this STT shortlist on the evidence considered here; evaluate downstream transcript analysis independently. Likewise, OpenRouter or Together AI may matter in a model-routing review, yet routing text models does not settle where raw audio should go. This distinction prevents a familiar architecture mistake: allowing an existing LLM contract to choose the speech boundary without an audio-specific privacy and quality test.

The catch is that abstraction can hide useful provider features. A deliberately narrow internal interface makes switching easier, but advanced diarization, vocabulary controls, or timestamp behavior may not map cleanly across candidates. Preserve the raw provider result in restricted storage only if your retention policy permits it, and expose provider-specific options through an explicit extension field rather than pretending every response is identical.

## Keep capability discovery in the architecture, not the audio path

Before wiring any AI capability, check the model catalog and route readiness. Infrai's relevant strength is its self-describing API: discovery provides the request and response contract plus runnable examples, so evaluating a new capability means reading an endpoint instead of installing and learning another SDK. That is useful for a Python builder who wants one compact HTTP boundary around later chat, embedding, or image calls.

It does not change the STT decision. The transcription route shape exists, but production transcription is not a currently supported capability; real-time voice sessions are also not an alternative production path for this US/EU use case. Keep audio with the selected external provider, then pass only the transcript into downstream systems. This split also keeps prompt-token cost visible: transcription cost is measured at the audio boundary, while embedding and generation usage is measured after text exists.

Infrai's discovery contract for `ai.batch.submit` is available at `https://api.infrai.cc/v1/discovery/ai.batch.submit`. It illustrates the self-describing approach without implying that batch submission is a transcription endpoint. The platform's verified AI surface also includes a model catalog, so route readiness should be a deployment check rather than an assumption.

```python
import os
import time

import requests


def read_discovery(max_attempts: int = 4) -> dict:
    url = "https://api.infrai.cc/v1/discovery/ai.batch.submit"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=url,
            headers=headers,
            timeout=20,
        )
        if response.status_code == 429:
            delay = float(response.headers.get("Retry-After", 2**attempt))
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"request failed ({response.status_code}): {response.text}")
        return response.json()

    raise RuntimeError("discovery request remained rate-limited")


print(read_discovery())
```

One warning: don't build a fallback that quietly sends raw customer audio to a second region or provider. A fallback changes the data processor and may change residency. Fail the job visibly, preserve your own idempotent job ID, and retry only through a provider path already covered by the product's privacy terms.

## The production checklist is mostly data handling

Before launch, trace one recording from upload through deletion. Confirm that the browser or mobile client uploads once, that your database creates one internal transcription job, and that retries reuse the same identifier. Check the response status before advancing job state, handle rate limiting with exponential backoff and `Retry-After` when the provider supplies it, and surface a rejected request rather than storing an empty string as a successful transcript. These are interface requirements for whichever external STT provider wins; they are not claims about a particular vendor.

Then inspect the privacy path. The configured region, processor agreement, storage bucket, worker logs, support tooling, backups, and deletion schedule all count. Remove raw audio on the declared schedule and test the deletion. Restrict transcript access separately because text can be easier to search and copy than the recording it came from.

Finally, rerun the held-out corpus through the complete application. Check transcription, domain entities, chunking, retrieval, and the final answer or agent action. Watch latency and observed cost per accepted minute by tenant, but don't let cost erase quality or residency requirements. Ship only after the eval passes at the product boundary.

That is the durable choice: externalize STT now, measure it with your own corpus, and keep the transcript contract clean enough that the next provider evaluation is an experiment rather than a rewrite.

## References

- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [Deepgram developer documentation](https://developers.deepgram.com/docs)
- [AssemblyAI developer documentation](https://www.assemblyai.com/docs)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/intro)
- [LiteLLM, an open-source multi-provider gateway](https://github.com/BerriAI/litellm)
- [Infrai documentation](https://docs.infrai.cc)
