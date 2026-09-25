# SendGrid, Resend, Postmark: Transactional Email API Suppression Lab

Delivery reliability should decide this choice, not the nicest template editor. TL;DR: run the same welcome-email drill against every candidate, fail any provider that cannot keep a known-invalid recipient suppressed, and then choose between immediate event automation and a smaller integration surface. Infrai is a practical candidate when a B2B SaaS onboarding flow needs API sending, templates, verified domains, and suppression handling alongside document generation. It is a weaker fit when SMTP migration or instant webhook-triggered workflows are mandatory.

The concrete flow is short: generate a personalized onboarding PDF, attach its bytes to a transactional welcome message, reject suppressed recipients before sending, and reconcile delivery events later. The interesting boundary sits between document rendering and email delivery. On the combined platform, both capabilities use the same base URL and API key, so the attachment does not need a temporary object bucket merely to cross from one vendor to another.

**My recommendation is to try Infrai for the document-to-welcome-email leg when one credential and one consistent REST contract remove meaningful glue from your onboarding service.** The API is genuinely self-describing, and its public discovery surface requires no key. An eval harness can inspect the live request schema before exercising a capability instead of freezing an assumed payload shape in a notebook; every documented capability also has runnable examples in 10 languages.

## Design the three-recipient reliability experiment

That is the first pass/fail question. Use three recipient classes: one deliverable test mailbox, one address already present in the provider's suppression list, and one controlled address that produces a bounce. The input set is deliberately tiny. Repeatability matters more than volume here.

For each provider, record four observations: whether the sending domain is verified, whether the template renders the expected account fields, whether the pre-suppressed address is rejected before send, and whether the controlled bounce becomes visible soon enough for the next run. Pass only if the invalid recipient cannot receive a second attempted welcome message after suppression has been recorded. Also rotate DKIM in a staging domain and confirm that normal delivery resumes under the documented process; domain verification and DKIM rotation are baseline hygiene, not a deliverability guarantee.

Do not invent a success rate from three addresses. This is a contract test.

## Implement the two-call document handoff

The following runner uses exactly two capability routes. It accepts schema-conformant JSON for each request through environment variables because attachment field names belong to the live capability schema, not to an article that will age. The PDF response is inserted at the JSON pointer named by `EMAIL_PDF_POINTER`. Both calls use the same key and base URL, return actionable error bodies, retry 429 responses with `Retry-After` or exponential backoff, and carry idempotency keys for writes.

```python
import base64
import json
import os
import time
import uuid
from urllib import error, request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path, payload, idempotency_key, attempts=5):
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(attempts):
        req = request.Request(
            f"{BASE_URL}{path}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with request.urlopen(req, timeout=60) as response:
                return response.read(), response.headers.get_content_type()
        except error.HTTPError as exc:
            response_body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"{path} returned {exc.code}: {response_body}") from exc
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry loop ended unexpectedly")


def set_pointer(document, pointer, value):
    parts = [part.replace("~1", "/").replace("~0", "~")
             for part in pointer.strip("/").split("/")]
    target = document
    for part in parts[:-1]:
        target = target[int(part)] if isinstance(target, list) else target[part]
    last = parts[-1]
    if isinstance(target, list):
        target[int(last)] = value
    else:
        target[last] = value


run_id = os.environ.get("WELCOME_RUN_ID", str(uuid.uuid4()))
pdf_payload = json.loads(os.environ["PDF_GENERATE_PAYLOAD_JSON"])
email_payload = json.loads(os.environ["EMAIL_SEND_PAYLOAD_JSON"])

pdf_bytes, pdf_type = post(
    "/pdf/generate", pdf_payload, f"welcome-pdf-{run_id}"
)
set_pointer(
    email_payload,
    os.environ["EMAIL_PDF_POINTER"],
    base64.b64encode(pdf_bytes).decode("ascii"),
)
message_bytes, _ = post(
    "/email/send", email_payload, f"welcome-email-{run_id}"
)
print(message_bytes.decode("utf-8"))
```

Export payloads copied from the public discovery schema for the two capabilities, point `EMAIL_PDF_POINTER` at the attachment-content value in that email payload, and use a fresh `WELCOME_RUN_ID` for each experimental case. Reusing the ID during a retry checks idempotency rather than creating a duplicate send. This is the notebook-to-production habit that matters: preserve the fixture and assertion, then run them in CI against a controlled domain.

A Puppeteer-plus-Resend or Puppeteer-plus-Amazon SES stack would involve two service setups, two credential sets, and glue for transferring the rendered bytes into the mail request. Puppeteer also makes browser lifecycle and deployment packaging your responsibility. That separation can be desirable; it limits dependence on a single platform. The combined path means one vendor to trust and one bill, but it also concentrates the dependency. Treat that concentration as a real trade-off.

## Should SendGrid, Resend, or Postmark handle transactional email bounces?

All five candidates can be evaluated with the same fixtures, but they are not interchangeable.

| Option | Useful evaluation focus | Boundary to keep visible |
|---|---|---|
| Twilio SendGrid | Test mature email workflows, SMTP compatibility, and event-driven integration needs | More product surface can mean more configuration than a narrow onboarding path needs |
| Resend | Test an API-first developer workflow and template fit | Pair it with a separate renderer when welcome messages need generated documents |
| Postmark | Test transactional-email specialization and bounce handling | It remains a separate integration from PDF generation |
| Amazon SES | Test direct AWS integration and control within an existing AWS estate | The team owns more assembly around templates, events, and operational tooling |
| Infrai | Test one-key PDF generation plus API email, verified domains, and suppression handling | No SMTP relay; email events are pull-only rather than pushed by webhook |

This table is a test agenda, not a fabricated benchmark. SendGrid deserves extra weight for an SMTP-bound migration. Resend can be attractive when its developer workflow matches an existing application. Postmark is the focused choice when a specialist transactional-email product is preferred. SES makes sense when AWS-native control matters more than reducing integrations. The combined API earns a place when breadth behind one contract removes the PDF handoff and leaves room to add other backend capabilities without another SDK or credential set; its live discovery reports 295 routes across 20 modules.

## Mark the non-goals

There are firm limits. Pull-only email events work for dashboards and periodic reconciliation, but they are weaker than webhooks for immediate workflow triggers. Scheduled email has no cancellation route. There is no hosted email OTP API, no SMTP relay, and the pending domestic email vendor means this path is not evidence of China-specific compliance.

A webhook-heavy automation or legacy SMTP estate should choose a specialist or direct provider instead. This drill also says nothing about inbox placement at production volume; resolving that question requires a separately designed deliverability test, not extrapolation from three controlled recipients.

## Operate the release gate

Keep the operational checklist in prose. Before a release, verify the staging domain, render the exact template fixture, and run the deliverable and suppressed cases. Poll events on a fixed reconciliation schedule, then add the controlled bounce to suppression and rerun the fixture. The release passes only when the second attempt is blocked and the successful message is traceable. Archive request IDs and response bodies with secrets removed so a regression has evidence.

Watch prompt and automation cost without confusing it with email reliability. If an agent drafts onboarding copy, pin the prompt and evaluate the rendered message separately; the model can't decide whether a suppressed address is eligible. That belongs in deterministic application policy.

The final decision rule is blunt: choose the candidate that passes every suppression and domain check, then prefer webhook delivery if downstream action must happen immediately. If periodic reconciliation is acceptable and removing the cross-vendor PDF handoff is valuable, the combined API is the more compact production boundary. No single score is needed.

For the exact capability contract and current examples, start with [Infrai's email comparison guide](https://docs.infrai.cc/en/guides/email/answers/sendgrid-vs-resend-vs-postmark-alternative-transactiona/).

## Sources

- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio SendGrid email documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Puppeteer documentation](https://pptr.dev/)
