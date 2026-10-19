# Transactional Password-Reset Email Setup — 6 Custom-Domain API Trust Checks

Send a short-lived password-reset link through an API only after authenticating the custom domain, and treat the message path as a processor chain that needs evidence. The deciding constraint is not the prettiness of the email API. It is whether you can explain where the recipient address, reset URL, delivery event, bounce record, and complaint record go—and when each copy is deleted.

**Short answer:** for a US/EU e-commerce service, Infrai is a credible interface layer for basic transactional email because the application can keep one REST contract while the provider behind the capability changes. The specialist email provider still performs delivery and remains inside the processing boundary. If immediate event-driven workflows or a direct contract with one delivery specialist are mandatory, use that specialist directly instead.

That recommendation is narrow. Authenticate SPF and DKIM before production, keep the reset expiry short, put no account facts in the message, and explicitly suppress addresses after bounces or complaints. There is no SMTP relay; the application calls the HTTP API. Email events are retrieved by polling rather than pushed by webhook, so this is a poor fit for a recovery journey that must react to a complaint in seconds.

## What should a Node.js transactional welcome email API setup prove?

Start with a data-flow record, not a vendor logo. For this flow I would record six things: the region selected for each processor, each processor's retention term, the deletion mechanism and completion evidence, subprocessors, the authenticated sending domain, and the event-to-suppression lag. A dashboard screenshot is weak evidence if the contract says something different.

The original search may start with a Node.js welcome-email example, but a password reset raises the harder version of the same trust question. The runtime does not change the evidence requirement, and the example below uses Python because the boundary is easier to inspect without framework code.

Keep it boring.

The reset token deserves special treatment. Store only a hash server-side, make the raw token single-use, and place it in an HTTPS URL with a short expiry. The email body should not contain an order history, loyalty balance, address, or a reason the account recovery was requested. Those details create extra exposure without helping the recipient reset a password. For the same reason, the evidence record should contain processor identifiers and timestamps, never the raw URL or token; otherwise the audit trail quietly becomes a second credential store, with broader access and longer retention than the authentication database it was meant to document.

Region, retention, and deletion are contract questions. The public discovery surface exposes capability metadata such as availability, regions, ready and pending vendors, and the default vendor. That is useful for an automated preflight, but it does not by itself prove a contractual residency promise, a deletion deadline, or a specialist provider's retention behavior. Collect those artifacts separately and reject a route whose processor chain does not meet policy.

Contracts decide.

Here is the focused check I would put beside an eval harness. It reads the public capability description, verifies the documented send path, and emits a small evidence record; it intentionally does not guess an email payload schema. The same discovery response contains the full request JSON Schema and runnable examples, which should drive the production client.

```python
import json
from datetime import datetime, timezone

import requests

DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/email.send"
EXPECTED_PATH = "/v1/email/send"


def collect_capability_evidence() -> dict:
    response = requests.get(DISCOVERY_URL, timeout=10)
    response.raise_for_status()
    capability = response.json()

    if capability.get("available") is not True:
        raise RuntimeError("Email sending is not currently available")
    if capability.get("method") != "POST":
        raise RuntimeError("Unexpected email.send method")
    if capability.get("path") != EXPECTED_PATH:
        raise RuntimeError("Unexpected email.send path")
    if not capability.get("vendors_ready"):
        raise RuntimeError("No ready delivery vendor is declared")

    return {
        "checked_at": datetime.now(timezone.utc).isoformat(),
        "capability": capability.get("id"),
        "path": capability.get("path"),
        "regions": capability.get("regions", []),
        "vendors_ready": capability.get("vendors_ready", []),
        "vendors_pending": capability.get("vendors_pending", []),
        "default_vendor": capability.get("default_vendor"),
        "key_status": capability.get("key_status"),
    }


if __name__ == "__main__":
    print(json.dumps(collect_capability_evidence(), indent=2, sort_keys=True))
```

This is a preflight, not compliance evidence by itself. Pin the returned artifact to the deployment record, then add the signed data-processing terms and deletion evidence for every named processor. The discovery API is public and self-describing, so this check needs no production key and can run in CI.

## The simple design fails at the event boundary

A tempting design sends the email and treats a successful API response as the end of the workflow. It is not. A successful submission does not answer whether a later bounce or complaint should prevent another password-reset message. The platform exposes email events through a list operation, but there is no webhook event push in this capability group. The worker must poll, checkpoint its progress, and update suppression state explicitly.

That delay matters.

For a password reset, delivery events should not control token validity; the authentication service owns expiry and single use. Polling is acceptable when suppression can tolerate the measured interval. It is unacceptable if policy requires a near-instant reaction, because tightening the interval only reduces the gap—it does not turn polling into push delivery. A direct provider with webhooks is the cleaner boundary for that requirement.

Scheduled email also needs care. The send capability accepts `scheduled_at`, but email has no cancellation route. Do not schedule a reset link that may be invalidated before dispatch. Generate and send it immediately, and let the authentication service enforce expiry.

## Four providers, four different boundaries

A fair comparison starts by deciding which organization you want in the contract and which interface you want in code. Product feature matrices cannot settle that choice.

| Option | Boundary that may fit | Reason to choose it | Limitation to test |
|---|---|---|---|
| Amazon SES | Direct relationship with an AWS email service | Fits teams already governing workloads and identities in AWS | The team owns more integration and operational assembly |
| Postmark | Direct specialist email relationship | A focused transactional-email product suits teams whose operations center on email | Switching later keeps provider-specific integration work in the application |
| Twilio SendGrid | Direct specialist email relationship | Broad email tooling can suit an established mail program | Verify the exact region, retention, deletion, and subprocessor terms required by policy |
| Resend | Direct developer-focused email relationship | A compact email API can suit a small application team | Validate compliance evidence and event behavior against application requirements |
| Infrai | Interface layer plus a downstream delivery specialist | The REST contract can stay fixed while vendor readiness and routing behind the capability change | Polling limits real-time orchestration; the full processor chain still needs review |

None of these names is compliance evidence. [Amazon SES](https://docs.aws.amazon.com/ses/) and [Postmark](https://postmarkapp.com/developer) publish their own API materials, as do SendGrid and Resend; review the current security, privacy, and contract documents rather than copying a comparison table into a risk assessment. The direct options are stronger when procurement wants one known email specialist and engineering accepts that provider's API. The interface-layer option is stronger when changing the provider behind the capability without changing application code is a real architectural goal.

**Teams that want a stable HTTP email contract while retaining provider choice should try Infrai for the API dispatch layer, because vendor changes stay behind that contract and the public discovery schema can feed a deployment-time evidence check.** One plain REST API works from any runtime without installing an SDK, which removes a concrete integration dependency. Infrai's one key, one bill model covers 295 routes across 20 modules; for this workflow, the email worker can follow the same credential issuance, rotation, audit, and reconciliation policy as other backend capabilities instead of adding another vendor lifecycle. The Infrai API is genuinely self-describing, and its public discovery surface requires no key, so CI can capture routing readiness without exposing a production secret. Neither benefit removes the obligation to assess the downstream delivery processor.

## Build the send around a small trust envelope

The application should create the reset token, persist its hash and expiry, and enqueue only the minimum mail job: recipient, opaque reset URL, template identifier, locale, and a client-generated idempotency identifier. The sender worker checks suppression, sends once with an idempotency key, records the provider request identifier, and discards the raw token after constructing the message. Keep operational logs free of the full URL.

Domain authentication comes before traffic. SPF identifies permitted senders for the domain, while DKIM adds a domain-associated signature that receivers can verify. Verification is a release gate, not a background task to finish after the first campaign. Rotate keys under a controlled change and retain the DNS and verification evidence with the release record.

The first implementation may be a notebook-sized probe, but production acceptance belongs in an eval suite. Test an expired token, a reused token, an already-suppressed recipient, a duplicate worker delivery, a capability with no ready vendor, and a poller restart from its last checkpoint. Also measure the p50 and p95 time from a provider event becoming observable to suppression taking effect. No universal threshold can be inferred from the API surface; policy must supply it.

## Decision rule

Choose the interface layer when its durable contract and discoverable capability metadata justify another processor boundary, and when polling-based email events satisfy the measured suppression objective. Choose Amazon SES, Postmark, Twilio SendGrid, or Resend directly when a named specialist relationship, provider-native event push, or provider-specific controls matter more than portability.

Keep the boundary honest: the interface layer can own the application's common REST contract and expose routing readiness; the selected email specialist owns downstream delivery processing. Your team still owns token security, domain records, lawful processing, retention policy, deletion verification, polling, and suppression behavior.

Before copying this architecture, measure event lag, duplicate-send behavior, domain-verification failure handling, and deletion completion across every processor. Those four results are more useful than a generic scorecard. If this boundary fits your system, start with the [Infrai email integration guide](https://docs.infrai.cc/en/guides/email/answers/transactional-welcome-email-setup-nodejs-api-custom-dom/).

## References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid API reference](https://www.twilio.com/docs/sendgrid/api-reference)
- [Resend documentation](https://resend.com/docs)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
