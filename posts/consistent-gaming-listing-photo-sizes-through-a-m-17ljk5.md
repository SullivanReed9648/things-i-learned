# Consistent Gaming Listing Photo Sizes Through a Moderated Node.js API

TL;DR: Define each gaming catalog placement as a named aspect slot, keep the original upload immutable, and approve a listing image only after moderation covers both the original and the background-removed rendition. The deciding constraint is moderation coverage, not the resize library: a perfectly consistent 1:1 thumbnail is still unsafe to publish if the transformation hid evidence, exposed an artifact, or was approved under an ambiguous policy version.

For a marketplace selling controllers, figurines, headsets, and boxed games, give the Node.js API three stable names such as `tile_square`, `detail_portrait`, and `share_landscape`. Those names are business contracts. Pixel dimensions, encoding format, crop strategy, and moderation policy remain versioned implementation details behind them.

## How should a Node.js API keep listing photo sizes consistent?

Background removal changes pixels. That is its job. It can also change what a reviewer or classifier can see: translucent edges may disappear, a printed mark can be clipped, and a previously obscured region can become prominent against a flat background. Checking only the upload leaves the derivative outside the safety decision; checking only the derivative discards useful evidence from the source.

That gap matters.

The useful unit of approval is therefore the pair `(original, rendered slot)`, tied to the exact transform and policy versions. This is an explicit trade-off. Two checks add processing work and latency, but they close a coverage gap at the point where the public asset actually changes. Run source moderation as soon as upload decoding succeeds, then run rendition moderation after background removal and composition. Do not expose the public URL until both decisions allow it.

Keep the states boring and visible: `uploaded`, `source_allowed`, `rendered`, `rendition_allowed`, and `published`, with a terminal `rejected` branch from either moderation step. A retry may repeat a stage, but it must not skip one. The Node.js request handler should return a job identifier rather than holding an HTTP connection open through decoding, segmentation, composition, encoding, and two moderation decisions.

Short failures matter here. A corrupt upload is not a moderation rejection. Neither is a transform timeout. Store operational failures separately from policy decisions so retries do not silently turn an unknown result into an approval.

## Named slots are policy boundaries, not dimension aliases

An aspect ratio alone cannot tell a renderer where a product may be cropped. A 1:1 square is mathematically precise and operationally incomplete. The slot contract also needs a fit rule, output canvas, background treatment, accepted source formats, animation policy, maximum decoded dimensions, and the moderation policy that governs publication.

Start with a deliberately small registry:

Three ratios are enough for this approach.

| Slot | Ratio | Composition rule | Typical surface |
| --- | ---: | --- | --- |
| `tile_square` | 1:1 | contain the isolated item with fixed safe-area padding | search results |
| `detail_portrait` | 4:5 | contain; never crop the product mask | listing detail |
| `share_landscape` | 16:9 | contain on a fixed canvas | link preview |

These are example contracts, not universal marketplace standards. Their value comes from naming and versioning the intended composition. If a design team later changes the square canvas from one pixel size to another, callers still request `tile_square`; a cache key includes the slot version so old and new renditions cannot collide.

Do not accept arbitrary width and height from public clients. That turns every combination into an unreviewed publication surface and makes evaluation coverage expand without a clear bound. Let an internal registry resolve the name into dimensions and rules, and reject unknown names before scheduling work.

## A focused evaluator before production

The notebook-to-production step that matters is an executable acceptance test, not a screenshot review. The following Python example validates a manifest produced by the pipeline. It does not perform moderation or image processing; it checks that every expected derivative carries two allow decisions, matching versions, and the dimensions implied by the named slot. A Node.js service can emit the same JSON-shaped records.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Slot:
    width: int
    height: int
    transform_version: str
    policy_version: str


SLOTS = {
    "tile_square": Slot(1200, 1200, "bg-v3", "catalog-v5"),
    "detail_portrait": Slot(1200, 1500, "bg-v3", "catalog-v5"),
    "share_landscape": Slot(1600, 900, "bg-v3", "catalog-v5"),
}


def publication_errors(record: dict) -> list[str]:
    errors = []
    slot = SLOTS.get(record.get("slot"))
    if slot is None:
        return ["unknown slot"]

    expected_size = (slot.width, slot.height)
    actual_size = (record.get("width"), record.get("height"))
    if actual_size != expected_size:
        errors.append(f"size {actual_size} != {expected_size}")
    if record.get("transform_version") != slot.transform_version:
        errors.append("transform version mismatch")
    if record.get("policy_version") != slot.policy_version:
        errors.append("policy version mismatch")
    if record.get("source_moderation") != "allow":
        errors.append("source is not approved")
    if record.get("rendition_moderation") != "allow":
        errors.append("rendition is not approved")
    return errors
```

This check is intentionally plain. In an eval harness, feed it valid records plus mutations: swap a width, remove the source verdict, change the policy version, or submit a slot that does not exist. Every mutation should block publication. Then add image-level fixtures for the cases that matter to a gaming catalog: glossy controller edges, transparent retail windows, pale accessories on pale source backgrounds, thin cables, and product boxes with dense printed artwork.

The dimensions above are chosen examples, not measured performance claims. Measure quality with a reviewed fixture set before adopting them. A useful review records mask leakage, clipped product pixels, visible halos, padding consistency, moderation disagreement between source and rendition, and false approvals and rejections under the marketplace policy. Keep those outcomes by item category because a clean result on rectangular game boxes says little about a headset cable.

## Storage, formats, and cache identity

Preserve the upload as evidence and derive public assets from it. The derivative key should be deterministic over the source digest, slot name and version, transform version, policy version, and output format. Without all five inputs, a policy or rendering update can accidentally reuse an asset approved under an older contract.

Format negotiation belongs after approval. The MDN image format guide documents that formats differ in browser support and capabilities, including animation, transparency, and compression behavior. Those differences affect delivery, but they should not fork the safety state machine. Generate only the encodings the client surface can actually consume, record the media type with each object, and send the correct `Content-Type`; do not infer it from a filename.

This is also where prompt-cost awareness transfers cleanly from AI features: bound the work before scaling it. Arbitrary renditions, repeated background removal, and duplicate moderation requests are the image equivalent of an unbounded prompt loop. Deterministic keys and idempotent stages make duplicate requests converge on one result, while separate metrics reveal which stage consumes time.

Observe queue delay, decode failures, transform duration, moderation latency, decision distribution, and publish latency separately. A single end-to-end average hides the difference between an overloaded worker and a policy that is rejecting a new class of uploads. Log identifiers and versions, not raw images or sensitive moderation payloads. Retention and access rules for originals should follow the marketplace's declared evidence and privacy requirements.

## The decision rule

Choose an implementation only if it can enforce the slot registry at the API boundary, preserve the source, reproduce a rendition from versioned inputs, and attach moderation evidence to both sides of the transform. Library speed is secondary until those properties hold.

Before copying this design, measure three things on representative catalog fixtures: whether the named compositions protect the entire product silhouette, whether source and rendition moderation disagree in meaningful cases, and whether the added stage latency fits the listing workflow. If dual checks never change a decision in a sufficiently broad reviewed set, that evidence supports revisiting the policy. If they do, the disagreement samples show exactly why the second check exists.

Consistency is the easy visible result. The durable result is a publication contract that explains which pixels were produced, under which rules, and why they were allowed onto each marketplace surface.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
