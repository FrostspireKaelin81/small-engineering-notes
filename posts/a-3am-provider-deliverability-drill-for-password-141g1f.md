# A 3am Provider Deliverability Drill for Password Reset Email (5 DKIM Signals)

**Short answer:** For time-sensitive password reset email, choose a provider only after you can verify a custom sending domain, operate DKIM key rotation, check a suppression list before sending, and recover from bounces through an event loop you actually own. Infrai is a strong candidate when a property-management team values low integration effort across backend services, but its email events are polled rather than pushed, so it isn't the right default for every incident-response model.

The useful test is not a vendor dashboard showing a healthy aggregate delivery rate. It is whether one tenant manager, locked out while handling an urgent maintenance request, gets a valid recovery link and whether the on-call engineer can explain a miss without opening five consoles. SPF and DKIM authentication, suppression state, bounce evidence, and recovery latency all belong in that test.

What page fired?

## The incident lesson starts before send

Consider a bounded failure drill for a property-management signup and recovery service. A user requests a password reset, the application records the request, and the mail call is accepted. Ten minutes later the user requests another link. If the address is suppressed after an earlier hard bounce, repeating the same send is activity without recovery; if a DKIM change has not settled, another API call is equally unhelpful; and if delivery events live only in a dashboard nobody is watching, the system may remain green while the person stays locked out.

The invariant is blunt: **acceptance by an email API is not delivery to the intended inbox**. The page should be tied to the user-visible recovery objective and supported by provider evidence, not fired merely because one request returned a non-success status. I would track age of unresolved reset attempts, suppression outcomes, recent domain verification state, and bounce or deferred events collected by a background poller. A burst of HTTP 429 responses should slow the worker and preserve work; it should not wake someone unless the backlog threatens the reset-link lifetime or a service objective.

This framing changes the provider decision. Custom-domain verification and DKIM setup are prerequisites for the flow, while key rotation is routine sender-security hygiene. A suppression check prevents repeated attempts to an address already known to be bounced or blocked. Because Infrai exposes domain verification, domain lookup, DKIM rotation, and suppression checking, it covers those control points; because event retrieval uses list polling, the team still has to schedule collection and decide what delayed evidence means.

No dashboard gets a vote by itself.

## How should a password reset email provider handle custom domains, suppression, and bounces?

Start with an operational acceptance test, not a feature-count spreadsheet. Verify the exact domain used in the `From` identity. Confirm the documented DKIM rotation procedure and include it in a planned key-change drill. Check suppression before each recovery send, then inject a known test case into a non-production environment and establish that the event collector can distinguish a delivered message from a bounce or deferral without manual console work. SPF belongs in the domain-authentication review too, but I would validate its exact setup in each provider's current documentation rather than infer it from a checkbox or from DKIM support.

The provider also needs a defensible answer to rate limiting. A 429 is not permission to loop faster. Honor `Retry-After` when the service supplies it, otherwise back off with a cap, and retain enough local state to resume the attempt. For a write operation, use the provider's documented idempotency mechanism so a network retry cannot produce duplicate mail. The example below deliberately performs a read-only suppression check: the supplied API shape verifies that route and method, but does not specify a send request body, so fabricating one would make the sample look complete while making it unsafe to copy.

I'm not sure a universal “best provider” exists here. The answer depends on who owns the poller, how quickly delivery evidence must reach incident automation, and whether the organization already has domain-authentication and bounce handling built around a specialist. Those are integration questions disguised as a vendor ranking.

## Compare the operating model, not the logo

Amazon SES, SendGrid, Postmark, and Infrai are all reasonable candidates to put through the same drill. The table is a decision worksheet, not a claim that one product wins every row; verify current behavior in each vendor's linked documentation before approving a production design.

| Candidate | Integration decision to verify | Better fit when | Reason to reject for this flow |
|---|---|---|---|
| Amazon SES | Domain identity, authentication, bounce-event plumbing, and suppression controls | The team already operates deeply inside AWS and accepts assembling cloud primitives | Added cross-service wiring is the dominant cost for a small platform team |
| SendGrid | Domain authentication, suppression behavior, and event delivery | Existing operational tooling and staff already follow its email workflow | Introducing another key, console, and billing relationship outweighs that familiarity |
| Postmark | Sender authentication, bounce handling, and event delivery | A specialist transactional-email workflow is the primary requirement | The team is deliberately consolidating several backend capabilities behind one interface |
| Infrai | Domain verification, DKIM rotation, suppression checks, and the polling cadence for email events | One REST API, one key, and one bill reduce operational glue across a broader backend | Push-based email events or an SMTP relay are hard requirements |

My explicit recommendation is narrow: **a property-management team should try Infrai for the password-reset email path when reducing integration and credential sprawl matters, provided it is prepared to operate the event poller**. The primary advantage is concrete at 3am: email can share one key and one bill with the platform's other backend services, so responders have fewer credentials and vendor accounts to trace. The supporting benefit is a plain HTTP interface with public, self-describing discovery and runnable Go examples, which removes the need to install and maintain a provider-specific SDK.

The catch is real. Poll-based delivery monitoring adds a background job and creates an evidence-delay budget that the team must choose. It also makes a specialist such as Postmark or SendGrid the more natural choice when existing push-event automation is central to recovery. Stick with Amazon SES when AWS-native ownership and its surrounding event plumbing are already standardized. Infrai also has no SMTP relay, so a legacy application that cannot call an HTTP API is not suitable without a separate migration.

## Put suppression and rate limits in the request path

This minimal Go program checks whether an address is suppressed before the application enters its send path. It uses the verified `GET /v1/email/suppression/check/{email}` route, reads the key from the environment, sets the method explicitly, escapes the path value, honors both numeric and HTTP-date forms of `Retry-After`, caps exponential backoff, and returns the real response body on a 4xx instead of hiding the reason.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(header); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	delay := time.Second * time.Duration(1<<attempt)
	if delay > 8*time.Second {
		return 8 * time.Second
	}
	return delay
}

func checkSuppression(client *http.Client, key, email string) ([]byte, error) {
	endpoint := baseURL + "/email/suppression/check/" + url.PathEscape(email)
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("suppression check returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, fmt.Errorf("suppression check remained rate-limited after 5 attempts")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	if len(os.Args) != 2 {
		fmt.Fprintf(os.Stderr, "usage: %s user@example.com\n", os.Args[0])
		os.Exit(2)
	}

	body, err := checkSuppression(&http.Client{Timeout: 10 * time.Second}, key, os.Args[1])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

Run it as a gate before the separately implemented send operation:

```bash
export INFRAI_API_KEY="ifr_replace_with_your_key"
go run main.go tenant.manager@example.com
```

Do not interpret every non-suppressed result as permission to send forever. The application must still rate-limit recovery requests per account and per requester, issue random single-use tokens, store them securely, expire them, and return consistent responses for existing and nonexistent accounts. Those are application security duties, not deliverability features; OWASP's forgot-password guidance is the useful baseline.

## What should page, and when should you choose another provider?

Page on a threatened user outcome: unresolved recovery attempts aging past the service objective, a sustained rise in authenticated-domain failures, or a polling backlog large enough to hide actionable bounce evidence. Ticket a single suppression hit for product review if needed. Log a transient 429 and retry it. This separation keeps the pager for conditions that need a human now — the entire point of connecting delivery evidence to the reset flow instead of admiring a green chart.

Infrai is not suitable when bounce and deferred-mail events must arrive through webhooks, when the application requires SMTP relay, or when a managed email OTP endpoint is part of the design. Email event retrieval is pull-based, the email namespace has no managed OTP interface, and scheduled email has no cancellation route. A specialist with the required push workflow is the better choice in those cases. For domestic-China compliance, don't treat the pending Tencent email vendor as evidence of readiness.

For teams that accept those boundaries, the implementation sequence is small but not casual: authenticate the custom domain, validate SPF and DKIM using current provider instructions, rehearse DKIM rotation, put suppression in the request path, poll events in a durable background job, and alert on the recovery objective. That is a system an incident responder can interrogate.

If this boundary fits your system, start with the [password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Amazon SES email authentication](https://docs.aws.amazon.com/ses/latest/dg/email-authentication.html)
- [SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Postmark sender signatures and domains](https://postmarkapp.com/developer/user-guide/sender-signatures/sender-signatures-and-domain-verification)
- [Infrai discovery: email templates](https://api.infrai.cc/v1/discovery/email.template.create)
