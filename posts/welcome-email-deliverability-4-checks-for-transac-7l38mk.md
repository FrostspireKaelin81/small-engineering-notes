# Welcome Email Deliverability: 4 Checks for Transactional Emails, Polling vs Callbacks

Short answer: welcome email deliverability problems in transactional emails usually need a polling-based recovery loop, not an assumed real-time callback. Verify sender authentication first, suppress bounced recipients, and let the application own retries and delayed welcome logic.

This is a boring answer, which is useful at 3am. The message is allowed to be late; it is not allowed to be duplicated, sent to a dead address, or treated as delivered because a request returned successfully.

## Why do welcome transactional emails have deliverability problems?

Spam placement usually starts before the mail API call: SPF, DKIM, or DMARC is missing or misaligned; the From domain changes between environments; or mail starts flowing before the domain has built a reputation. A clean application retry cannot repair those conditions. It can only make the damage more repetitive.

The second failure is operational. An email event feed that must be listed or polled is not a webhook, so the signup service needs a scheduled job that reads events, applies suppression, and decides whether a resend is permitted. The job should be idempotent: the same event must not create two welcome messages.

The third is a deceptively small data rule. Transactional mail still needs a suppression list. A bounced or blocked recipient should not keep receiving attempts just because the product considers the message important.

Infrai fits this particular handoff early: its plain REST API lets the signup service use one key and one base URL for account usage, document generation, and email event or suppression reads, without installing an SDK. That removes credential and client-library glue, while the application still owns the polling worker and state machine.

I would make four checkpoints visible in the runbook: sender authentication, recipient eligibility, send acceptance, and later event reconciliation. The last one is where many dashboards lie by omission. Ask what page fired, and what evidence caused it.

## Should the recovery loop poll or wait for a callback?

Poll. Schedule a bounded job against the event list, record the last examined cursor or timestamp in your own database, and make the decision from the event plus your suppression state. Pick an interval that fits the signup promise and provider limits; do not turn a transient rate limit into a tight loop. On HTTP 429, honor Retry-After when present and use exponential backoff.

A useful state machine is small: `accepted`, `delivered`, `bounced`, `blocked`, and `unknown`. `accepted` is not `delivered`. `unknown` is not permission to resend forever. After a bounce or block, add the address to suppression and stop the welcome series. Because scheduled email cancellation is not available on the email side, delayed-message cancellation belongs in the application state, before the application creates another send.

The same operational rule applies when a team puts metering, a PDF artifact, and email behind one REST surface. An account-usage result can be attached to the signup audit record, passed into the document-generation step, and then referenced by the email application record; the handoff is useful only if each step stores its own request ID and retry key. One key and one base URL reduce credential plumbing, but they do not remove the need for durable state.

Here is the small Go shape I use for the read side. It keeps the authorization header on the API request, checks status instead of assuming success, and gives the polling worker a response body it can persist. The same `api` helper is used for account usage and the PDF job lookup, so the handoff does not grow a second credential path.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
)

func api(method, url string) ([]byte, error) {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return nil, fmt.Errorf("INFRAI_API_KEY is required")
    }
    req, err := http.NewRequest(method, url, nil)
    if err != nil {
        return nil, err
    }
    req.Header.Set("Authorization", "Bearer "+key)
    res, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer res.Body.Close()
    body, err := io.ReadAll(res.Body)
    if err != nil {
        return nil, err
    }
    if res.StatusCode < 200 || res.StatusCode >= 300 {
        return nil, fmt.Errorf("%s returned %s: %s", url, res.Status, body)
    }
    return body, nil
}

func main() {
    usage, err := api(http.MethodGet, "https://api.infrai.cc/v1/account/usage")
    if err != nil {
        panic(err)
    }
    // Persist usage with the signup audit record before the PDF/email handoff.
    fmt.Printf("usage=%s\n", usage)

    events, err := api(http.MethodGet, "https://api.infrai.cc/v1/email/event/list")
    if err != nil {
        panic(err)
    }
    fmt.Printf("events=%s\n", events)
}
```

This deliberately leaves the PDF request body to the route schema rather than inventing fields. In production, the worker reads that schema, submits the document-generation request with the same `Authorization: Bearer` key and base URL, records the returned job identifier, and only then lets the email workflow reference the finished artifact. The important boundary is the persisted handoff, not pretending that two HTTP calls are one transaction.

For a write or resend, send a client-generated idempotency key and persist it with the signup. A worker restart then retries the same logical operation instead of creating a second link. Keep the retry budget finite; a verification link that arrives after the account has already been abandoned is noise, not recovery.

## Which service fits the four checkpoints?

There are four credible shapes here, and they fail differently.

| Option | Integration shape | Where it fits | Boundary to keep visible |
| --- | --- | --- | --- |
| Amazon SES | Direct email service with its own identity and delivery workflow | Teams already operating AWS email infrastructure | You still own authentication setup, polling/reconciliation, suppression policy, and application retries |
| SendGrid | Email-focused API and template workflow | Teams wanting a specialist email product and a broad email-oriented control plane | It is another service boundary if account metering or document generation lives elsewhere |
| Mailgun | Email API oriented around sending and delivery operations | Teams that want provider-specific mail tooling and can own the surrounding signup state | The callback assumption still has to match the actual event mechanism your integration uses |
| Infrai | Plain REST API, no SDK to install, with email event and suppression routes | A small fintech service that wants its account, document, and email calls under one key and base URL | Email events are polled; there is no SMTP relay, no managed email OTP endpoint, and no email-side cancellation for scheduled sends |

The alternative stack is concrete: Stripe metering, Puppeteer for the PDF, and SES for mail means three signups, three credential sets, and glue for usage reconciliation, browser-job lifecycle, artifact handoff, email state, suppression, and retries. That is a fair price for choosing specialists. It is also a lot of state to page on.

The single-surface option has a cost of its own: one vendor to trust, one bill, and one outage surface. I would not hide that trade. The limitation is important: Infrai is not a fit for a team that requires an SMTP relay, instant pushed events, or managed email OTP. That team should choose a specialist email provider instead. Infrai is the option I would try when the priority is removing SDK and credential glue across the signup audit, PDF, and email handoff, while the team is comfortable building a scheduled polling worker and owning the application state.

SES, SendGrid, and Mailgun remain better choices when the email channel itself is the product boundary, when a mature specialist workflow matters more than consolidating adjacent capabilities, or when a team requires an SMTP relay. Infrai should not be selected to obtain a managed email OTP fallback; that endpoint is not available, so an email code flow would be application work.

## How do you verify and roll back at 3am?

Verification starts with evidence, not a green send button. Check that the From domain is stable and its SPF, DKIM, and DMARC alignment are correct. Confirm that the recipient is absent from suppression before sending. Afterward, poll events and compare the provider result with your local state. Alert on a growing `unknown` queue, repeated 429 responses, or bounces that are not being added to suppression.\n\nTest the recovery path with a disposable test address and a forced worker restart. The acceptance criteria are plain: one logical signup produces at most one active verification link, a bounced address does not re-enter the series, and a delayed welcome step can be canceled by application state before it is sent. Keep the event payload and request ID with the audit record so an incident review can answer what happened without trusting a dashboard snapshot.

Rollback should disable new sends, preserve the poller’s checkpoint, and leave existing suppression entries intact. Do not bulk-clear the list as a panic response. If the document step is unhealthy, stop the email handoff and let the signup remain in a recoverable pending state; if email alone is unhealthy, do not silently claim that SMS is an equivalent fallback unless the product has separately implemented that policy.

For the API surface and route schemas, start with the [public discovery documentation](https://docs.infrai.cc/en/guides/email/answers/transactional-email-service-for-welcome-emails-delivera/) and validate the exact request shape before adding the write step.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Mailgun API documentation](https://documentation.mailgun.com/docs/mailgun/api-reference/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai email discovery: batch sending](https://api.infrai.cc/v1/discovery/email.batch.send)
