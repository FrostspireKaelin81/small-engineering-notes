# Paid Receipt Recovery: SMS Outage Alerts API Evidence Before Batch Replay

Release a receipt recovery batch only after each settled payment has a durable receipt record, a current suppression decision, and a disposition for any send whose outcome is unknown. An SMS API's acceptance response is too weak a basis for declaring an incident over.

**Short answer:** For a startup serving US and EU customers, evaluate an SMS outage alert API through a recovery exercise: per-recipient batch outcomes, identifiable template revisions, current suppression checks, and reconciliation of uncertain submissions. For B2B SaaS order receipts, page on unresolved receipts after payment settlement and prove recovery against individual orders before releasing the next batch.

## What page fired when the receipt never arrived?

Start the incident review there. A dashboard showing successful API requests doesn't establish that the paid order has a resolved receipt, and a red request counter doesn't tell the responder which customer needs action. Make the settled order the unit of reconciliation, retaining separate evidence for receipt creation, submission, and whatever delivery status the selected transport actually exposes.

Keep states separate.

For this proposed runbook, an unresolved receipt is a settled payment whose receipt workflow has neither met the team's documented completion condition nor reached an explicit disposition such as suppression or manual review. Set the alert threshold from the receipt promise made to customers. There is no universal threshold for this workflow; choosing one without a service objective would produce an attractive chart and an arbitrary page.

Distinguish the customer's receipt from the responder's outage alert. Give them separate templates and recipient policies. An internal alert can carry an order identifier that helps locate evidence without copying the customer's receipt contents onto responders' phones. If the receipt travels by email, evaluate its transport evidence separately; the email documentation in References covers a different channel from the SMS segmentation documentation.

The page should open the order record (including its unresolved attempt). Otherwise the responder starts with a graph and still has to reconstruct the work.

## How should a startup test an SMS outage alerts API before batch recovery?

Take an incident-recovery approach to API selection: ask the candidate integration to explain what happens when the connection ends before the application knows the submission outcome. Require a documented answer and a reproducible test. Treat an ambiguous submission as work to reconcile, with a recorded owner and next action, rather than immediately authorizing another send.

The trade-off is slower recovery and more operational work in exchange for control over duplicate receipts. A startup needs to decide who owns unresolved records before choosing an interface that can leave them there.

Consider this illustrative ledger. These are invented order identifiers, not incident measurements.

| Settled order | Receipt evidence | Recovery decision |
| --- | --- | --- |
| 7101 | No submission attempted | Eligible for a fresh policy check |
| 7102 | Submission outcome unknown | Reconcile before considering replay |
| 7103 | Suppression recorded | Hold; retain the reason |

All 3 orders belong in the reconciliation view. Only the first is ready to be considered for submission, and even its eligibility can change before dispatch. A batch success flag cannot express these distinctions. Ask for per-item identifiers and results, and verify partial-result behavior rather than assuming an entire request either succeeded or failed.

For US and EU operation, make destination coverage and applicable messaging requirements explicit acceptance criteria. Obtain current documentation for the destinations and message purpose involved. A region label is insufficient evidence of suitability; this runbook does not infer consent, sender eligibility, or legal permission from a settled payment.

## Put suppression and template evidence on the dispatch path

Link the settled payment, intended recipient, message purpose, and approved template revision in the durable receipt record. Freeze the candidate recovery set for review, but read suppression policy again at dispatch: a snapshot is useful incident evidence, while a stale eligibility decision is a poor authorization to send. Record the decision and its policy revision so the responder can explain the worker's action.

Keep suppression scope explicit. A hold for internal outage alerts and a hold for customer receipts need named purposes and documented behavior.

This Go example is a small worker decision function. Its inputs are illustrative internal state, not fields promised by an SMS API. It returns a disposition without sending anything, making the decision easy to exercise in a recovery test.

```go
package receipt

type Candidate struct {
	OrderID          string
	Settled          bool
	Suppressed       bool
	SubmissionState  string
	TemplateRevision string
}

func RecoveryDecision(c Candidate, approvedRevision string) string {
	if c.OrderID == "" || !c.Settled {
		return "hold: missing settled order"
	}
	if c.Suppressed {
		return "hold: suppression"
	}
	if c.TemplateRevision == "" || approvedRevision == "" {
		return "hold: missing template approval"
	}
	if c.TemplateRevision != approvedRevision {
		return "hold: template revision changed"
	}
	switch c.SubmissionState {
	case "never_attempted":
		return "eligible"
	case "unknown":
		return "reconcile"
	default:
		return "hold: inspect existing submission"
	}
}
```

“Eligible” only means the worker may proceed with its remaining controls. A limitation of this function is that it cannot reserve the record or prevent concurrent dispatch. Add a durable claim, define what happens when a worker loses ownership, and persist the attempt before crossing the external submission boundary. Keep the same logical receipt identity across recovery attempts, but verify the interface's documented duplicate-handling behavior before relying on an idempotency mechanism.

A policy race remains possible if suppression changes after the worker's last read. Define the authorization boundary explicitly and test that pause or revocation reaches workers at that boundary. This function provides no atomicity between a policy store and an external sender.

Template support deserves the same skepticism. Ask whether a revision can be identified in dispatch evidence and whether the rendered message can be inspected under the team's data-handling rules. The SMS character-limit reference distinguishes GSM-7 and UCS-2 encoding and explains segmentation. Inspect final text, including variable order references and punctuation: a template name doesn't establish the rendered message's segment count.

Treat segmentation as a capacity and readability concern here. Price cannot resolve an ambiguous send.

If the surrounding application uses Node.js, preserve these decision inputs and durable transitions at its worker boundary. Check SDK syntax separately. The operational contract should survive a language change.

## Verify recovery with deliberate uncertainty

Before deployment, run a bounded test set through the complete worker, using controlled destinations and a submission adapter that can simulate uncertain outcomes. Reproduce the decision the responder will face: a settled payment exists, a submission may already have occurred, and the next worker must determine what it is authorized to do.

Use these cases as a proposed acceptance suite:

- A settled order with no attempt becomes eligible after current policy and template checks.
- A suppression change after batch planning prevents dispatch when observed at the defined authorization boundary.
- An unknown submission outcome stays in reconciliation instead of entering automatic replay.
- A mixed batch preserves item identities and dispositions when only some results are available.
- A template revision change holds a queued item pending approval.

Then test the page. Inject an unresolved receipt, verify that the configured threshold triggers the responder path, and confirm that the page opens evidence for the affected order. Exercise outage alerts and customer receipts separately so an alert test cannot accidentally authorize receipt replay. A green dashboard is useful only when the underlying records explain why it is green.

For rollout, begin with a bounded recovery cohort, inspect its dispositions, and expand after the unresolved set can be accounted for. Choose the cohort size from the team's ability to review it; this article establishes no safe batch-size benchmark. Preserve evidence that compares the planned set with what workers actually attempted, including items held by policy.

This design adds storage, review effort, and explicit worker states. It is unsuitable as a claim of guaranteed delivery: it helps account for receipts, but cannot establish that a recipient read a message. Accept the extra operational complexity only with a clear owner for exceptions.

## Roll back dispatch without erasing the incident

Define rollback as pausing new recovery submissions at the worker authorization boundary while preserving settled payments, receipt records, and attempt evidence. Verify that workers observe the pause before relying on it; already-started external submissions still need reconciliation.

Don't turn rollback into a bulk reset of unknown outcomes. Reverting a template also doesn't settle what happened to messages already submitted with another revision. Retain their original evidence and review them under the same recovery policy.

Close the incident when the affected settled-order set has explicit dispositions, remaining exceptions have owners, and the alert has been verified against the team's completion condition. Make API acceptance depend on that recovery exercise: suppression decisions, template evidence, and uncertain submissions must remain explainable after the dashboard returns to green.

## References

- SMS character limits, GSM-7/UCS-2 encoding, and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Email transport scope and service documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
