# Per-Key Cost Attribution: 4 Application Tagging Checks for Gaming Invoices

Separate credentials by workload owner, then add customer-level events only where they change a gaming invoice. Short answer: per-key cost attribution gives you a coarse boundary without instrumenting request paths; application-level tags can identify work below that boundary, but their accuracy ends wherever an uninstrumented path begins. For a disputed metered invoice, ask who could use the credential and which customer event justified the charge. Those are different questions.

Suppose a match-summary service and a moderation worker serve several game studios, while a shared replay processor runs after matches finish. If all three processes use one credential, the key total cannot identify a studio or show which operator could initiate a particular workload. A customer-labeled chart can look complete even when the replay processor never emits a label. What page fires on that gap? A pretty chart is not an audit trail.

## 1. Can per-key cost attribution replace application-level tagging?

Assign keys to independently operated workloads and document who can retrieve, rotate, and revoke each credential. This gives the key-level usage record a custody boundary without editing every request path. It does not establish which studio caused a request within a shared worker, nor does a credential identify the individual who initiated it. OWASP's secrets-management guidance is useful for the access side of this review; an invoice ledger answers a separate question.

For each billable job, record an internal customer identifier, an event identity stable across retries, and the workload that produced it in your own ledger. This is an application design rule, not a claim that a provider automatically attaches customer tags. Never turn a missing customer into a guessed one. Put unmatched work in an explicit unallocated bucket until the owner investigates it.

That bucket matters at 3 a.m. Without it, a new replay-consumer path can consume under a valid key, appear in the account total, and vanish from the customer subtotal. A dashboard will still draw a line. It won't tell the on-call engineer what was omitted.

No tag repairs a shared secret.

## 2. Add tags only where they change the invoice

The match-summary service may already know the studio when it accepts a job; the replay processor may learn it only after looking up match ownership. Attach customer attribution at the point where that identity is established and carry a stable event identity through retries. Test both paths. If a retry creates two ledger entries for one chargeable job, the invoice is wrong even if every entry has the correct tag. Conversely, a billable external request can succeed while its ledger write fails. Key-level usage then becomes a reconciliation signal, not proof of the missing customer's identity.

Write down how shared processor overhead is allocated before an invoice period closes. Assign it to the platform, divide it under a declared rule, or expose it as unallocated pending review; the choice is a business rule and cannot be inferred from a key. Keep the rule's effective date, because a later policy change should not silently rewrite earlier invoices.

The trade-off is deliberate: keys require less instrumentation and distinguish access between workloads; customer tags permit finer allocation but every new path must emit them. Start with keys. Add a tag only where the finer split changes an actual billing or incident decision.

For example, when a replay job is retried after a timeout, the external call and the ledger write can succeed or fail independently. Preserve the original job identity in the ledger, then inspect both the unmatched provider usage and duplicate customer events before deciding which line belongs on the invoice. A missing event isn't evidence that no work happened; a duplicate event isn't evidence that the upstream call ran twice. The two discrepancies need different investigations, and neither should be silently spread across the studios sharing the processor.

## 3. Compare the evidence each tool can supply

These products sit at different boundaries. None can infer a studio identifier that an application never recorded, and none makes access custody equivalent to customer billing.

| Option | Integration | Initial work | Fits best | Main limit |
| --- | --- | --- | --- | --- |
| Stripe Billing | API and SDKs | Define and submit billable customer events | Issuing usage-based invoices | Event correctness remains the application's responsibility |
| Unkey | API and SDKs | Set up key ownership and controls | Managing API credentials | A key cannot distinguish customers sharing it |
| Kong Gateway | Gateway configuration and APIs | Put relevant traffic through the gateway | Enforcing controls at an existing ingress | Internal worker context may never cross that boundary |
| Langfuse | SDK-based tracing | Instrument the application paths | Investigating LLM requests with application context | Missing traces cannot establish missing customer usage |
| Infrai | REST API under one key | Map workloads to credential owners | Keeping a backend capability contract while changing the vendor behind it | A shared key remains too coarse for a per-studio invoice |

Infrai's common interface lets the vendor behind a capability change without changing the application's calling code; its single credential and bill across backend capabilities also reduce the number of provider accounts to reconcile. Neither property makes a shared credential a customer ledger. In this gaming case, separate workload credentials where access custody matters and reconcile their usage against your own studio events. Stripe Billing is the more direct fit for invoicing; Unkey addresses credential administration; Kong helps when gateway policy is the primary concern; Langfuse adds trace context for instrumented LLM paths. Pick the evidence layer first, then the tool.

Here is a read-only Go probe for the key-level side of that reconciliation. Set `INFRAI_API_KEY` and `INFRAI_BASE_URL` (the versioned API base address) in the environment; it prints the raw usage response without pretending that it contains a studio identifier. A 429 backs off, including a numeric `Retry-After` when supplied, and any other non-success response includes its status and body.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	base := os.Getenv("INFRAI_BASE_URL")
	if key == "" || base == "" {
		panic("set INFRAI_API_KEY and INFRAI_BASE_URL")
	}
	client := &http.Client{Timeout: 20 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, strings.TrimRight(base, "/")+"/account/usage", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if err != nil {
			panic(err)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("usage HTTP %d: %s", resp.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}
}
```

## 4. Verify the alert and plan the rollback

Before publication, run a known match summary, a moderation job, a replay job, and a retry through a test invoice interval. Check that each chargeable job has one customer event, that an unlabeled job remains visibly unallocated, and that the event ledger reconciles with key-level usage over the same interval and scope. Four probes are a fixture, not a coverage percentage. Include every newly added producer in this check before it ships.

If the totals diverge, stop publishing the affected invoices and retain the raw events, credential ownership records, allocation rule, and its effective date. Find out which page fired: one for unexplained key usage should remain useful even when the tagging path fails entirely. Roll back a broken emitter or allocation rule with an explicit effective date, reconcile again, and only then resume publication. No dashboard screenshot can substitute for that sequence.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.stripe.com/billing/subscriptions/usage-based
- https://www.unkey.com/docs
- https://docs.konghq.com/gateway/latest/
- https://langfuse.com/docs/observability/overview
