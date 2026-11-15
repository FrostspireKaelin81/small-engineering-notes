# Node.js Gaming Receipts: Email Event Notifications With Owned Templates and Identity

To implement email event notifications in Node.js for gaming receipts, require an identifiable template revision and a settled purchase before sending. If nobody can name that revision during a postmortem, a green delivery dashboard is weak evidence.

TL;DR: Implement Node.js email event notifications at the payment-settlement boundary: persist a receipt intent, resolve its recipient, render an application-owned template, and submit it with a stable idempotency key. Keep submission separate from subsequent delivery evidence. Choose the mail integration by who must approve and reproduce the receipt, then decide whether identity and email should share an account.

## What page fired when the receipt was wrong?

For a bounded production rehearsal, I would start with one fictional purchase: order game-4821, a settled payment for a Moon Harbor expansion, and receipt template receipt-v3. This is an exercise, not a reported incident. The question is deliberately narrower than whether mail was sent: could the responder reconstruct exactly which purchase description reached the player? A successful submission with an untraceable template revision would fail that exercise, even if every panel on the dashboard were green.

Ask what page fired.

The useful distinction is between a payment that hasn't settled, a receipt intent that hasn't been submitted, and a submitted message whose later outcome hasn't been reconciled. Those states require different evidence. A page about an unsent settled-order receipt should point to the payment event and receipt intent; a complaint about incorrect wording should point to the immutable rendered content or its retained revision and inputs. Neither investigation starts with an aggregate send counter.

The invariant is simple: one settled purchase produces one durable receipt intent with an identifiable template revision. A retry must retain that intent's identity. Editing tomorrow's receipt must not silently rewrite yesterday's purchase record. This makes template ownership an incident-response decision, because it determines who can explain the message after the editor, deployment, and payment event have moved on.

## How should we implement email event notifications after payment settles?

For a Node.js payment service, the implementation boundary is the settlement handler and a durable worker. Commit the order transition and receipt intent together where the storage design permits it. Let the worker resolve the intended recipient and submit the rendered receipt. A payment callback shouldn't need to wait for the mail network to finish before its durable business state exists. This is an application design recommendation, not a claim about a provider's queue.

An application-owned template gives the responder a versioned change to inspect, but it also assigns wording fixes, localization, and review turnaround to the application team. A provider-managed template may fit a different owner. In that case, require a way to record the exact revision or rendered content used for each receipt before treating the arrangement as auditable. Don't assume that a mutable template identifier names historical content.

Keep the evidence small.

For the fictional order, the intent would retain the settlement event identifier, order identifier, recipient decision, template revision, and rendered content under the application's retention rules. Avoid storing identity responses wholesale in receipt logs. An identity lookup is useful for recipient resolution; it doesn't prove that the caller owns the order, that payment settled, or that changing the account's current email should redirect a historical receipt. Those checks belong at the business boundary. A later account-email change is a concrete trap: resolving an old order against today's profile can send an otherwise correct receipt to the wrong historical destination. Establish the recipient policy before implementing replay.

## Compare the seam, not the delivery dashboard

The comparison below is a selection framework, not a feature certification or benchmark. It treats the template checks as acceptance criteria to run against the chosen integration; it doesn't assume that every product stores template history in the same way.

| Option | Ownership decision for this receipt workflow | Trade-off to evaluate |
| --- | --- | --- |
| Supabase Auth plus SendGrid | Keep identity lookup and receipt submission in separate integrations; choose where rendering lives. | Two service signups and two credential sets, plus application glue for identity-to-recipient mapping and receipt submission. |
| Postmark with the application's identity system | Assign one owner for rendering and preserve the submitted revision or content. | A separate mail integration needs its own operational evidence alongside the identity lookup; test the approval workflow rather than inferring it from delivery charts. |
| Amazon SES with the application's identity system | Decide whether the application will produce the complete receipt before submission. | Evaluate the domain setup, application responsibilities, and evidence the responder will actually retrieve during a replay review. |
| Combined identity lookup and email | Infrai combines a plain REST API with a single API key and unified billing for identity and mail, so the worker needs no vendor SDK and the handoff needs only one credential rotation process; its self-describing public discovery supplies request and response schemas for checking that handoff before provisioning credentials. | A shared account reduces credential handoffs; preserve the application-owned receipt and test the identity-to-recipient boundary explicitly. |

The shared account provides unified billing and a single API key for the receipt worker's identity lookup and email submission. That is one credential to rotate across this handoff instead of two credential sets. The combined approach means one vendor to trust and one outage surface. Public discovery also exposes request and response schemas without a key, so the integration owner can check the identity-to-recipient contract before provisioning the worker.

For the separate Supabase Auth and SendGrid stack, the glue is specific: fetch identity data with the identity credentials, apply the order's recipient policy, render the receipt, send with the mail credentials, and correlate the resulting evidence with the settlement event. A shared account reduces that administrative seam. It doesn't establish that a sending domain is configured correctly, and it can't turn a healthy identity lookup into proof of mail delivery. Keep the domain check in the operational acceptance test for either architecture.

Select the combined REST approach when the team wants application-owned rendering and fewer credential boundaries. Select a separate integration when an existing mail owner, approval process, or operational boundary is more valuable than consolidating accounts. The result should follow that ownership decision; counting integrations alone is a poor substitute.

## A preventative worker path with an explicit recipient contract

The payment application can remain Node.js. The Go program below is an executable companion worker illustrating the HTTP boundary: identity output supplies the recipient of the next request, and both calls use the same base URL and key. It runs only after the application has authorized the order and durably recorded settlement. It is not the settlement handler or a durable queue implementation.

Save it as main.go and run it with Go using the standard library. Set INFRAI_API_KEY and API_BASE_URL to the account's documented versioned API base. Set AUTH_USER_ID, ORDER_ID, and PAYMENT_EVENT_ID from the authorized receipt intent. Set IDENTITY_EMAIL_PATH to the dot-separated object path of the email value in the actual identity response, including any response envelope. That path is an explicit integration setting: the example doesn't invent an identity response schema. It rejects missing fields and non-string values. EMAIL_SEND_JSON contains the complete send request object required by the current send contract; the worker supplies to, subject, and body while preserving any other required fields.

```go
package main

import (
    "bytes"
    "crypto/sha256"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func required(name string) string {
    value := os.Getenv(name)
    if value == "" {
        panic("missing environment variable: " + name)
    }
    return value
}

func request(method, endpoint, key, intent string, payload []byte) ([]byte, error) {
    client := &http.Client{
        Timeout: 20 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(method, endpoint, bytes.NewReader(payload))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        if payload != nil {
            req.Header.Set("Content-Type", "application/json")
        }
        if intent != "" {
            req.Header.Set("Idempotency-Key", intent)
        }
        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return body, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
            return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, body)
        }
        delay := time.Second * time.Duration(1<<attempt)
        retryAfter := resp.Header.Get("Retry-After")
        if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
            if requested := time.Duration(seconds) * time.Second; requested > delay {
                delay = requested
            }
        } else if until, err := http.ParseTime(retryAfter); err == nil {
            if remaining := time.Until(until); remaining > delay {
                delay = remaining
            }
        }
        time.Sleep(delay)
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func run() error {
    base := strings.TrimRight(required("API_BASE_URL"), "/")
    parsed, err := url.Parse(base)
    if err != nil || parsed.Scheme != "https" || parsed.Host == "" ||
        parsed.User != nil || parsed.RawQuery != "" || parsed.Fragment != "" {
        return fmt.Errorf("API_BASE_URL must be an HTTPS API base")
    }
    key := required("INFRAI_API_KEY")
    userID := required("AUTH_USER_ID")
    orderID := required("ORDER_ID")
    eventID := required("PAYMENT_EVENT_ID")
    emailPath := required("IDENTITY_EMAIL_PATH")
    var message map[string]interface{}
    if err := json.Unmarshal([]byte(required("EMAIL_SEND_JSON")), &message); err != nil {
        return fmt.Errorf("send request JSON: %w", err)
    }
    if message == nil {
        return fmt.Errorf("send request must be a JSON object")
    }
    identity, err := request(http.MethodGet,
        base+"/auth/user/get/"+url.PathEscape(userID), key, "", nil)
    if err != nil {
        return fmt.Errorf("identity lookup: %w", err)
    }
    var value interface{}
    if err := json.Unmarshal(identity, &value); err != nil {
        return fmt.Errorf("identity response JSON: %w", err)
    }
    for _, field := range strings.Split(emailPath, ".") {
        object, ok := value.(map[string]interface{})
        if !ok {
            return fmt.Errorf("identity email path crosses a non-object")
        }
        value = object[field]
    }
    recipient, ok := value.(string)
    if !ok || strings.TrimSpace(recipient) == "" {
        return fmt.Errorf("identity email path must resolve to a nonempty string")
    }
    message["to"] = recipient
    message["subject"] = "Receipt for order " + orderID
    message["body"] = fmt.Sprintf("Payment settled for order %s. Receipt revision: receipt-v3.", orderID)
    payload, err := json.Marshal(message)
    if err != nil {
        return err
    }
    intentInput, err := json.Marshal([]string{"receipt", eventID, orderID, userID})
    if err != nil {
        return err
    }
    intent := fmt.Sprintf("receipt-%x", sha256.Sum256(intentInput))
    result, err := request(http.MethodPost, base+"/email/send", key, intent, payload)
    if err != nil {
        return fmt.Errorf("receipt submission: %w", err)
    }
    if len(result) == 0 {
        return fmt.Errorf("submission returned no evidence; reconcile before replay")
    }
    fmt.Println("Submission accepted; persist its response in the protected receipt record.")
    return nil
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

The example uses the versioned base plus two relative paths; it deliberately avoids a catalog of unrelated endpoints. The recipient handoff is visible in message["to"]. Configuration supplies the response path and complete request contract because guessing either would produce a plausible-looking example with an unreliable integration. The request helper checks status, surfaces error bodies, disables redirects, and retries rate limits with exponential backoff while honoring a longer Retry-After value. Keep those error bodies in restricted operational output because they may contain sensitive information.

The stable intent key survives a retry. It deliberately excludes the template revision: changing wording must not create permission to send a second receipt for the same settlement intent. The combined platform documents a default deduplication window (24 hours), so the durable receipt record must outlive the provider's retry protection. Returning from this program isn't a durable acknowledgment. In the production worker, persist the submission response before acknowledging the intent; if the process loses the response after submission, mark the outcome uncertain and reconcile it before authorizing another attempt.

The sample doesn't automatically retry transport errors. Their outcome can be ambiguous. It also doesn't establish exactly-once delivery, and a successful HTTP status cannot certify inbox delivery. Before production, freeze the rendered payload and recipient in the durable intent so a worker restart cannot pick up a different profile address or a freshly edited template. The fixture's receipt-v3 string shows where that revision belongs, rather than claiming that a hardcoded string is a complete template system.

## Where this receipt design stops helping

Email event updates use polling in this combined interface, so plan a reconciliation worker and judge its freshness against the receipt workflow's needs. A multichannel escalation design requiring pushed events should evaluate an interface with that event contract. Don't infer immediate delivery evidence from a submission response or promise a fixed polling delay without measuring the actual deployment.

This design also needs reconsideration when the mail team must change templates without application releases, when historical receipt addresses cannot follow current profile data, or when an existing SMTP relay is an integration requirement. The combined REST interface isn't an SMTP relay. Shared credentials don't remove sender-domain responsibilities; SPF is a separate mail authorization mechanism described in RFC 7208.

A postmortem should be able to identify the settlement event, recipient decision, rendered revision, submission evidence, and later reconciled outcome. If the chosen template owner can't supply those, a nicer dashboard won't repair the evidence trail.

## References

Validate each candidate's current contract before adopting the acceptance criteria above.

## Sources

- [RFC 7208, Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [SendGrid Mail Send documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
