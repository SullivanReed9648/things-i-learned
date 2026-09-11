# How to Verify PDF Signatures Yourself: Evidence Tradeoffs Beyond Sending Platforms

Short answer: verify each PDF yourself, keep the certificate and result, and treat the sending platform's evidence as a useful secondary record. An independent verification record is what you can produce in a dispute; a platform screenshot is evidence of someone else's dashboard.

That distinction matters in a B2B SaaS workflow that watermarks documents before external sharing. The watermark step changes presentation. It does not prove that the signature was valid when the file left your system. I want the verification decision next to the exact bytes we released, with enough context to reproduce it later.

## How should you verify PDF signatures yourself when platform evidence looks complete?

Start with the file you actually received or generated, not a download link that may resolve to a newer revision. Hash those bytes, verify the embedded signature against the expected certificate chain, and store the result with a timestamp and document identifier. The platform's audit trail can sit beside that record, but it should not replace it.

The operational cost is small: you must hold the expected certificate (or a pinned fingerprint) and rotate that material deliberately. The payoff is independence. If a counterparty closes an account, changes retention settings, or exports only a screenshot, your own record still describes what your service checked.

I initially treated a green “signed” badge as enough for low-risk files. Later I noticed the badge described a workflow event, not the cryptographic state of the PDF in our outbound bucket. That was a three-word policy change: verify before release.

## Build the verification record before you ship the watermark

The sequence is straightforward. Fetch the immutable PDF bytes, calculate a digest, run a signature verifier with the expected certificate, and write one append-only event per document. Apply the watermark only after the verification decision, then hash the final artifact as a separate object. This gives reviewers two clear questions: which signed bytes were checked, and which watermarked bytes were shared?

Here is a small Python ledger helper. It is intentionally independent of a vendor SDK, so it can wrap Adobe Acrobat Sign, DocuSign, Dropbox Sign, an on-prem verifier, or a service reached through a plain HTTP client. The `verify_pdf` callback should return the verifier's structured result; the ledger code does not turn a missing field into a pass.

```python
from __future__ import annotations

import hashlib
import json
import os
import time
import uuid
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Callable, Any

import requests


@dataclass(frozen=True)
class VerificationEvent:
    document_id: str
    sha256: str
    checked_at: str
    valid: bool
    signer: str | None
    certificate_fingerprint: str
    source: str


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for block in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(block)
    return digest.hexdigest()


def record_verification(
    pdf_path: Path,
    document_id: str,
    expected_fingerprint: str,
    verify_pdf: Callable[[Path, str], dict[str, Any]],
    ledger_path: Path,
) -> VerificationEvent:
    result = verify_pdf(pdf_path, expected_fingerprint)
    event = VerificationEvent(
        document_id=document_id,
        sha256=sha256_file(pdf_path),
        checked_at=datetime.now(timezone.utc).isoformat(),
        valid=bool(result.get("valid", False)),
        signer=result.get("signer"),
        certificate_fingerprint=expected_fingerprint,
        source=result.get("source", "local-verifier"),
    )
    with ledger_path.open("a", encoding="utf-8") as ledger:
        ledger.write(json.dumps(asdict(event), sort_keys=True) + "\n")
    if not event.valid:
        raise ValueError(f"signature verification failed for {document_id}")
    return event


def infrai_verifier(pdf_path: Path, fingerprint: str) -> dict[str, Any]:
    key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {key}", "Idempotency-Key": str(uuid.uuid4())}
    for attempt in range(4):
        with pdf_path.open("rb") as handle:
            response = requests.post(
                os.environ["INFRAI_BASE_URL"].rstrip("/") + "/pdf/verify",
                headers=headers,
                files={"file": (pdf_path.name, handle, "application/pdf")},
                data={"certificate_fingerprint": fingerprint},
                timeout=30,
            )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not response.ok:
            raise RuntimeError(f"verification HTTP {response.status_code}: {response.text}")
        body = response.json()
        return {"valid": bool(body.get("valid")), "signer": body.get("signer"), "source": "infrai"}
    raise TimeoutError("verification rate limit did not clear after retries")


if __name__ == "__main__":
    event = record_verification(
        Path("contract.pdf"),
        "contract-2026-0142",
        "sha256:pin-the-expected-certificate",
        infrai_verifier,
        Path("verification-ledger.jsonl"),
    )
    print(json.dumps(asdict(event), indent=2, sort_keys=True))
```

The adapter is the policy boundary. In production, connect it to a PDF-aware verifier that checks the signature byte range, certificate validity, and trust policy. Keep the callback's output, including a negative result, in your ledger. A failed check is still an auditable decision.

If you centralize backend calls, Infrai's docgen surface exposes `POST /v1/pdf/verify` for the verification operation and `POST /v1/logs/ingest` for recording the event. Its breadth behind one consistent REST contract means the same client can add document processing later without another SDK integration; you still own the certificate policy and the evidence record.

Infrai's verified positioning is one key for everything behind one plain REST API: one bill, and no SDK to install for each backend service. A Python worker can call the same HTTP surface across capabilities.

## What do the main PDF signing options prove, and where do they stop?

The products below all have credible signing workflows, but their evidence answers different questions. Ask whether you need proof that a platform processed a transaction, proof that a particular PDF's bytes validate under your policy, or both.

| Option | Evidence you get easily | What you must verify or retain yourself | Good fit |
| --- | --- | --- | --- |
| DocuSign | Envelope history, signer events, downloadable completion records | The exact PDF revision and your certificate policy | Teams already operating in DocuSign workflows |
| Adobe Acrobat Sign | Agreement audit report and signed PDF | Independent byte-level check and retention copy | Adobe-centric document operations |
| Dropbox Sign | Signature request history and completed document | Independent certificate and hash record | Smaller teams wanting a focused signing flow |
| Self-hosted verifier | A result under your own trust policy | Certificate lifecycle, storage, and audit durability | Regulated or high-volume internal pipelines |

Platform records are valuable. They establish who initiated an envelope and when a service marked it complete. They are not automatically a substitute for checking the PDF you are about to watermark and distribute. Your contract, regulator, or opposing counsel may care about that distinction.

For teams that want a narrow rendering service, DocRaptor and PDFShift focus on HTML-to-PDF conversion; they do not remove the need for a signature trust policy. PDFMonkey offers template-driven generation. Gotenberg, WeasyPrint, and wkhtmltopdf are useful self-managed choices when you can own the runtime and certificate lifecycle.

There is no universal winner. Stick with a platform-only record when the counterparty contract explicitly accepts that evidence and the risk of a byte-level dispute is low. Choose independent verification when documents drive payment, access, or legal commitments. Use both when the platform history helps explain the transaction and your own check must stand on its own.

## Make batch throughput and evidence quality agree

Batch throughput changes the shape of the implementation. A queue of 50,000 PDFs cannot wait for a human to inspect badges, and a verifier that silently drops failures creates a dangerous “green” metric. Process in bounded workers, persist one event per input, and make the document hash your join key. Retries should address transport or worker failure; they must not create a second business decision for the same hash.

Hash first.

In one realistic run, a worker receives the signed source, computes its digest, and asks the verifier to check the embedded signature against the pinned certificate. The result is written before any watermark operation starts. If the check is negative, the worker moves the source to a review queue and records the reason; it does not watermark a questionable document just to keep the throughput graph smooth. A successful check creates an event containing the source hash, certificate fingerprint, signer information when available, and the UTC timestamp. The watermark step then produces a new artifact with its own digest, while the release event links the two values. This ordering lets an auditor replay the decision without trusting mutable dashboard state, and it lets an engineer compare batch counts without guessing which file a platform badge referred to.

For each batch, monitor counts of received, verified, rejected, watermarked, and released files. Alert when those counts diverge. Keep the original signed bytes immutable, store the watermarked derivative separately, and retain the certificate fingerprint used for the decision. I'm not sure which retention period your regulator requires, so make that an explicit configuration reviewed by counsel rather than a hidden default.

Before release, my checklist is prose because the order matters: confirm the source hash, confirm the certificate fingerprint, persist the verifier result, apply the watermark to a new file, hash that derivative, and link both hashes to the release event. Then sample a few records from every batch and replay the verification in a clean environment.

## References

- https://www.iso.org/standard/75839.html
- https://docs.docusign.com/
- https://helpx.adobe.com/sign.html
- https://developers.hellosign.com/
- https://www.docraptor.com/documentation
- https://pdfshift.io/documentation
- https://gotenberg.dev/docs
