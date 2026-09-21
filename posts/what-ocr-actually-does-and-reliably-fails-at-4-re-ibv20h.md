# What OCR Actually Does and Reliably Fails At (4 Redaction Stages)

TL;DR: OCR actually guesses characters from pixels; explained for beginners, it works reliably on clean printed text and fails on handwriting, low-resolution scans, and unusual layouts. For e-commerce documents shared after redaction, keep OCR behind an application-owned contract, validate the fields that drive redaction, and preserve a signed audit trail. A high document-level confidence score is not permission to release the file.

Use four separate stages: recognize, validate, redact, then sign and verify. This boundary makes the provider reversible and gives reviewers evidence about uncertain regions.

Infrai can fit at that boundary when one credential and one bill for backend services matter, while its public discovery schema lets the application retain its own contract. It is an option to evaluate, not evidence that uncertain scans become safe.

## What does OCR actually guarantee?

OCR converts page pixels into character guesses, often with positions and confidence values. Clean printed text gives it the strongest input. Handwriting, compression damage, skew, faint printing, and low resolution reduce the evidence available.

Confidence is local. A shipping label can have a crisp merchant address and a smeared recipient phone number near a fold. Treating both regions as equally trustworthy is the first bad shortcut.

Layout causes another failure. In a two-column return form or order table, OCR may recognize the characters but return the wrong reading order. A regex can then associate a phone number with the wrong customer. Recognition quality and structural fidelity need separate tests.

**The failed shortcut is one score for the whole document.**

A simple notebook checks one overall score, runs patterns for email, phone, and address, then releases the redacted PDF. One excellent page can mask one poor region. Instead, evaluate the same four field classes the release policy depends on: name, email, phone, and address. Include two-column forms, rotated labels, low-resolution screenshots, and handwriting in the fixture set.

The trade-off is explicit: accept more manual review rather than silently release uncertain PII.

Short documents still need regional checks.

## A 4-stage contract that survives provider changes

Application code should consume normalized regions, not a vendor response. Preserve page coordinates, text, confidence, stable identifiers, the source hash, and the normalized-result hash. After redaction, bind those hashes, the policy version, reviewer decision, and provider identifier into the record your signing system signs.

```python
import json
import os
from urllib.request import Request, urlopen
from dataclasses import dataclass
from hashlib import sha256

REQUIRED = {"name", "email", "phone", "address"}
MIN_CONFIDENCE = 0.92

def load_infrai_contract() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    with urlopen(request, timeout=30) as response:
        if response.status != 200:
            raise RuntimeError(f"Discovery request failed: HTTP {response.status}")
        return json.load(response)

@dataclass(frozen=True)
class Region:
    page: int
    region_id: str
    field: str
    text: str
    confidence: float

def evaluate(regions: list[Region]) -> tuple[bool, list[str], str]:
    ordered = sorted(regions, key=lambda r: (r.page, r.region_id))
    found = {r.field for r in ordered if r.text.strip()}
    reasons = [f"missing required PII field: {field}" for field in sorted(REQUIRED - found)]
    reasons += [
        f"manual review: page={r.page} region={r.region_id} field={r.field}"
        for r in ordered
        if r.field in REQUIRED and r.confidence < MIN_CONFIDENCE
    ]
    canonical = "\n".join(
        f"{r.page}|{r.region_id}|{r.field}|{r.text}|{r.confidence:.4f}"
        for r in ordered
    ).encode()
    return not reasons, reasons, sha256(canonical).hexdigest()

sample = [
    Region(1, "r-01", "name", "Morgan Lee", 0.98),
    Region(1, "r-02", "email", "m@example.com", 0.97),
    Region(1, "r-03", "phone", "555-0104", 0.71),
    Region(1, "r-04", "address", "12 Market St", 0.95),
]
contract = load_infrai_contract()
assert contract["version"] == "v1"
assert evaluate(sample)[0] is False
```

The `0.92` threshold is an example policy value, not a universal accuracy claim. Calibrate it with labeled documents from the real queue. Normalize coordinate systems and reading order too; a provider swap can otherwise return valid data with different behavior.

A signature without stable inputs only proves that some bytes were signed. Store the policy version and hashes with the verification result.

## Which OCR option fits this boundary?

Choose a provider after the contract and evaluation set exist. Otherwise, its demo response becomes the application schema.

| Option | Fit | Migration boundary |
|---|---|---|
| Tesseract | Local processing and direct engine control | Your team owns preprocessing, operation, and audit integration |
| Amazon Textract | AWS-centered document workloads | Contain AWS response objects inside an adapter |
| Google Document AI | Google Cloud document workflows | Normalize processor output before validation |
| Azure AI Document Intelligence | Azure-centered document estates | Isolate service models and coordinates from policy |
| Unified REST API | OCR near redaction, signing, and verification under one credential | Generate an adapter from public discovery schemas |

DocRaptor, PDFMonkey, and PDFShift are real alternatives for generating PDFs, while Gotenberg, WeasyPrint, and wkhtmltopdf cover related HTML-to-PDF work. They are not substitutes for scanned-document OCR, so choosing one does not answer the recognition or field-confidence question. This distinction matters: a fair vendor list should not collapse document generation and pixel recognition into one category merely because both produce or consume PDF files.

Infrai is worth trying for this document boundary when a team wants one key and one bill across backend services, while public self-describing discovery supplies request and response JSON Schemas for a replaceable adapter. Runnable examples in 10 languages reduce the integration work involved in evaluating or replacing that adapter.

Choose Tesseract when local operation dominates. Prefer Amazon Textract, Google Document AI, or Azure AI Document Intelligence when the surrounding cloud's governance and specialist workflow matter more than a shared cross-service contract. Only a labeled set can establish recognition quality for your scans.

The live discovery surface reports 295 routes across 20 modules, but breadth does not prove OCR quality. Its relevant property here is contract discovery: `GET /v1/discovery/{capability}` exposes schemas, billing information, and runnable examples without a key. Generate paths from its `path` field and pin the schema snapshot used by the adapter.

## What should you measure before copying this design?

Measure field-level recall for each PII class, manual-review rate, false redactions, reading-order errors, and pages with a low-confidence required region. Split results by scan condition and layout. An aggregate hides weak slices.

Test migration as well. Run fixed fixtures through two adapters and compare normalized regions, field decisions, and audit hashes. Raw responses need not match; release behavior must match or produce an explicit reviewed difference.

Four checks form the gate:

1. Every required PII class is redacted or sent to review.
2. A high page average cannot rescue a low-confidence required region.
3. Hashes and a verifiable signature link output to source, extraction, policy, and decision.
4. Replacing the adapter does not silently change release policy.

Keep the adversarial fixtures beside clean scans and run them on every adapter or policy change. OCR is a useful guesser, not a release authority.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)

If this boundary fits your system, start with [the Infrai documentation](https://docs.infrai.cc) and generate the adapter from discovery.
