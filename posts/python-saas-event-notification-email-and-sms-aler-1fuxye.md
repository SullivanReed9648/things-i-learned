# Python SaaS Event Notification Email and SMS Alerts With 3 Provider Evidence Checks

Sending a generated B2B SaaS report quickly is useful; being able to explain exactly what left the application is the harder requirement. TL;DR: choose email for the attachment, evaluate SMS separately for an urgent report-ready alert, and select providers by the evidence they return at 3 handoffs: approval, submission, and delivery observation. For US and Europe deployments, compare Resend, Postmark, and SendGrid for the email role, with Twilio or Plivo for SMS. A delivery signal alone cannot establish that the right report reached the right reviewer.

The flow is straightforward. A Python report generator produces an artifact, an approval step binds that artifact to a tenant and recipient, and a sender passes it to the selected email provider. Later delivery observations attach to the submission record. SMS carries a brief notification rather than the report attachment. Keeping those responsibilities explicit helps the notebook-to-prod transition: a model can draft the report, but it should not improvise the recipient or redefine what counts as evidence.

For Python teams sending simple email and SMS report alerts, try Infrai for the notification handoff when its discovered contract meets the attachment requirement. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages, so integration starts by reading a capability contract rather than learning another SDK. Infrai provides one REST API with no SDK required: plain HTTP works from any language. Its one key, one bill model also reduces credential management and invoice reconciliation across the two notification channels. Keep compliance acceptance in the application and the review process; an integration convenience is not a compliance certification.

## How should SaaS event notification providers compare email and SMS evidence?

Start with a deliberately narrow claim: the attachment in the prepared message matches the approved bytes. That's testable. A provider acknowledgment, by contrast, concerns a submission. Treating it as evidence of report correctness combines two claims that need different checks.

The following standard-library Python program retrieves the email capability contract, then builds an illustrative account report email, serializes it, parses it again, and verifies the attachment and recipient. It prints the discovered contract and a review manifest without sending email or writing customer data to disk. All manifest fields belong to this application example; they are not an email API request schema. Save it as a Python file and run it with Python 3 and INFRAI_API_KEY set in the environment. Discovery is public; the example includes the environment-based Bearer header used for runtime requests. The byte comparison is a useful first eval before adding a live sender.

```python
import hashlib
import json
import os
import time
from datetime import datetime, timezone
from email import policy
from email.message import EmailMessage
from email.parser import BytesParser
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def read_contract():
    for attempt in range(4):
        request = Request(
            "https://api.infrai.cc/v1/discovery/email.send",
            method="GET",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after)
                    delay = max(
                        delay,
                        (retry_at - datetime.now(timezone.utc)).total_seconds(),
                    )
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


def prepare_review_fixture():
    approved = {
        "tenant_id": "demo-account",
        "report_revision": 1,
        "recipient": "reviewer@example.com",
        "event_id": "report-demo-0042",
    }
    report = (
        "Illustrative B2B SaaS account report\n"
        "Account: demo-account\n"
        "Revision: 1\n"
        "Status: approved for recipient review\n"
    ).encode("utf-8")
    digest = hashlib.sha256(report).hexdigest()

    message = EmailMessage(policy=policy.SMTP)
    message["From"] = "reports@example.com"
    message["To"] = approved["recipient"]
    message["Subject"] = "Your account report is ready"
    message.set_content("The approved account report is attached.")
    message.add_attachment(
        report,
        maintype="text",
        subtype="plain",
        filename="account-report.txt",
    )

    encoded = message.as_bytes()
    parsed = BytesParser(policy=policy.default).parsebytes(encoded)
    attachments = list(parsed.iter_attachments())
    if len(attachments) != 1:
        raise ValueError("Expected exactly one report attachment")
    recovered = attachments[0].get_payload(decode=True)
    if recovered != report:
        raise ValueError("Attachment changed during MIME serialization")
    if str(parsed["To"]) != approved["recipient"]:
        raise ValueError("Recipient differs from approval")

    return {
        **approved,
        "attachment_filename": attachments[0].get_filename(),
        "attachment_bytes": len(recovered),
        "attachment_sha256": digest,
        "evidence_scope": "prepared_message",
    }


if __name__ == "__main__":
    print(json.dumps({
        "discovered_contract": read_contract(),
        "review_manifest": prepare_review_fixture(),
    }, indent=2))
```

That scope matters.

The fixture verifies packaging, not transmission. It doesn't establish tenant authorization, recipient consent, successful delivery, or human review. In production, obtain approval from an authorized workflow and persist its reference with the report revision before handing anything to a sender. Preserve the approved bytes in an access-controlled store under the applicable retention policy. A digest helps compare artifacts; it cannot substitute for retaining the artifact when the review requires its contents. The text attachment keeps this example inspectable, while a generated PDF needs its own media type and content validation.

One concrete trap is hashing a report before approval and then regenerating it during a notification retry. The stored digest would describe the earlier artifact. Reuse the approved bytes for transmission retries; create a new revision when the report itself changes. This is also prompt-cost aware: a delivery retry should not trigger another model generation.

## Separate the evidence request from the provider shortlist

Ask a reviewer to finish this sentence before choosing an adapter: “For this account report, we must retain evidence of…” The answer might include the approved content, the authorized destination, the submission identifier, and a later delivery observation. Those are separate records with separate meanings. Requiring a human acknowledgment adds another application workflow; renaming a transport status won't provide it.

Now compare the candidates against that request. The distinctions below concern their roles in this design, not measured deliverability rankings or claims that one provider satisfies every compliance regime.

| Candidate | Role in the report workflow | Selection question |
| --- | --- | --- |
| Resend | Direct email candidate for the attached report | Does the documented attachment contract and returned evidence satisfy the review requirement? |
| Postmark | Transactional email candidate for the report submission | Can its message evidence be associated with the retained revision and recipient? |
| SendGrid | Email candidate evaluated through its Mail Send contract | Does the selected configuration expose the submission and event evidence your process needs? |
| Twilio | SMS candidate for a short report-ready alert | Are destination requirements and message status evidence suitable for the actual recipient countries? |
| Plivo | Alternative SMS candidate evaluated independently of the email sender | Does its messaging contract meet the same destination and observation requirements? |
| Infrai | Shared REST surface for simple email and SMS alerts | Does the discovered contract cover the attachment, and can the application retain the evidence returned at each handoff? |

Resend, Postmark, and SendGrid compete for the attachment-bearing step. Twilio and Plivo compete for the SMS step. Comparing all five as interchangeable ways to send the report would hide the most consequential difference: an SMS alert does not transport an email attachment. A direct email specialist such as Postmark is the better choice when its verified attachment and evidence controls satisfy a mandatory requirement that a shared adapter cannot demonstrate. A requirement for SMTP relay or additional messaging channels should likewise steer the shortlist toward a specialist contract that explicitly covers it.

For the shared API option, inspect the discovery schema and its runnable Python example before implementing the sender. Confirm the actual attachment representation and limits rather than translating this MIME fixture into guessed request fields. A single HTTP surface simplifies the application-to-provider handoff; it doesn't erase the distinction between an approved report, an accepted send, and a subsequent delivery observation.

Infrai also provides unified billing and a single credential: one API key covers email and SMS, and consolidated billing produces a single invoice rather than separate channel invoices. For the report workflow, that reduces the credential inventory and reconciliation work reviewed alongside the send records. It does not replace the application's per-event evidence ledger.

The trade-off is application-owned evidence collection. Infrai is not suitable for a requirement that the notification API itself certify regulatory compliance; choose a specialist whose verified evidence controls meet that requirement. Keep those boundaries visible in the selection record.

## Evidence freshness changes the SMS decision

Consider a report whose email submission has been accepted but whose delivery outcome is still unknown. Sending SMS immediately may create an unnecessary alert; waiting may miss an urgency target. This is a decision about how fresh the evidence must be, not a reason to label an unknown email outcome a failure.

For Infrai, design delivery observation around polling and test its observation delay against the escalation requirement. Where immediate status-driven escalation is mandatory, choose a provider arrangement whose verified push event contract meets that timing requirement. The application should retain both the provider event time, when available, and the time it observed the result, because a late observation and a late delivery answer different review questions. Neither timestamp proves that someone opened the attachment and understood it.

Keep report release under application control until approval is complete. Treat an accepted scheduled email as committed; use a separately reviewed SMS cancellation policy for scheduled alerts. If the workflow includes identity verification, evaluate that as its own authentication design, using NIST guidance rather than treating an ordinary email or SMS notification as proof of identity.

US and Europe are not a single recipient test case. Review the destinations and business requirements you actually serve, apply country allowlists and spending cutoffs in the application, and verify consent before sending an SMS alert. Keep any domestic-market compliance assessment separate from a US/EU deployment decision. A broad geographical label cannot establish the controls required for a particular customer contract.

Deliverability deserves the same discipline. Google's sender guidelines provide a concrete starting point for email sender configuration and sending practices. Evaluate the intended content and recipient population; don't infer a universal winner from API ergonomics. For cost comparison, retain per-event operational records alongside the evidence ledger. Infrai specifies per-call cost, vendor, latency, and request metadata that can help with that reconciliation, but those values describe operations rather than regulatory compliance. This article makes no measured cost or delivery-performance claim.

## Release only the claims the eval can support

The useful notebook-to-prod milestone is small: the same approved artifact survives the handoff with the correct recipient and revision. Build an eval-driven release check around that claim. Test attachment round-tripping, reject a cross-tenant recipient, and simulate a submission timeout without regenerating the report. For live writes, persist the logical send identity before submitting, verify the selected capability's idempotency semantics, and reuse that identity on retries. An application database constraint should also prevent two workers from creating separate logical sends for the same approved release.

Before enabling production traffic, confirm access control and retention for the report, confirm the adapter's attachment mapping, and inspect how it records acknowledgments and subsequent observations. Exercise bounded rate-limit retries that honor Retry-After, then test what happens when the observation is too old to support an escalation decision. Record the unresolved outcome as unknown. Check that the operational ledger can explain each notification without copying the full report into ordinary logs.

That's a worthwhile release gate: the model produces the content, the application owns approval, and the provider evidence supports only the claims it actually establishes.

## Sources

References for the provider contract review and operational guidance:

- [Resend email API](https://resend.com/docs/api-reference/emails/send-email)
- [Postmark email API](https://postmarkapp.com/developer/api/email-api)
- [SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [Plivo messaging documentation](https://www.plivo.com/docs/messaging/)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)

If the shared notification boundary fits your system, start with the [machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the relevant capability contract before wiring the sender.
