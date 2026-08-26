# Production Error Tracking: 5 Ways to Capture HTTP, Cron, Queue Worker Exceptions

Short answer: in NestJS production error tracking, let a global exception filter own HTTP exceptions, use an interceptor only for shared request context, and wrap cron jobs and queue workers separately; rollback must remain independent of every capture call.

For a property-management checkout, the invariant is blunt: a failed payment or lease update must not leave a reservation half-committed just because the reporter is slow, rate-limited, or unavailable. Error capture is evidence. It is not part of the transaction.

Consider a bounded incident: an HTTP checkout reserves a unit, a queue worker confirms payment, and a cron job releases expired holds. The HTTP handler returns `409`, the worker retries three times, and the release job is expected every five minutes. A dashboard full of captured stack traces can still miss the dangerous case: the release process never started, so it threw nothing. The postmortem question is therefore not “did the graph turn red?” It is “what page fired, and did rollback finish before it fired?”

## 1. How does a NestJS production error tracking filter capture HTTP exceptions, cron jobs, and queue workers?

Use a global NestJS exception filter as the final HTTP boundary. It should preserve the framework's intended response, attach request and correlation context, report the thrown exception once, and avoid converting expected client errors into urgent pages. An interceptor can add timing or shared context, but relying on an interceptor alone is risky when an exception is transformed elsewhere in the pipeline. The filter sees the final HTTP failure.

Cron jobs and queue processors need separate wrappers because they don't cross the HTTP exception layer. Wrap the top-level scheduled callback and the queue processor, record the job name, attempt, stable checkout ID, and correlation ID, then rethrow so the scheduler or queue retains control of its normal retry semantics. Don't swallow it.

The same exception can cross several layers. Pick one reporting owner for each path: the HTTP filter for requests, the processor wrapper for queue attempts, and the cron wrapper for scheduled executions. Lower services should return or throw typed errors, not report and rethrow them, or a single payment failure becomes four events and makes grouping less trustworthy.

This is the first check I would put in an incident review: can one checkout ID reconstruct the HTTP attempt, worker retries, and final disposition without treating a retry as a new customer action? If the answer is unclear, more dashboards won't fix it.

## 2. Why is the transaction boundary placed before error reporting?

The preventative code path should make the ordering executable, not a convention buried in a runbook. A NestJS service should roll back first, make the failed operation externally safe, then send diagnostic evidence on a bounded context and return the original error. Reporting failure must never replace the business exception.

After that cleanup boundary, this runnable Go poller reads unresolved error groups through the verified Infrai route. It keeps credentials in environment variables, declares the method, honors `Retry-After` on `429`, applies bounded exponential backoff, and surfaces a non-success response body. Set `INFRAI_BASE_URL` to the API origin so the unlinked example contains no vendor URL.

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

func retryDelay(value string, fallback time.Duration) time.Duration {
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if deadline, err := http.ParseTime(value); err == nil {
		if delay := time.Until(deadline); delay > 0 {
			return delay
		}
	}
	return fallback
}

func main() {
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || apiKey == "" {
		panic("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+"/v1/errors/groups", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := retryDelay(resp.Header.Get("Retry-After"), time.Second<<attempt)
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("error groups request returned %s: %s", resp.Status, body))
		}

		fmt.Println(string(body))
		return
	}
	panic("error groups request remained rate-limited after four attempts")
}
```

Two details matter. First, this query belongs after rollback or in a separate alerting process; it cannot be allowed to extend the checkout transaction. Second, the code prints the response without inventing a local response schema, leaving the public discovery contract as the authority. In a real checkout, the idempotency key belongs at every externally retried write boundary as well, especially payment confirmation, because an at-least-once queue can deliver the same message again.

Keep this code small.

I'm not sure that ten seconds is the right polling budget for every deployment; measure the tail latency and choose a bound outside the checkout's own deadline. The invariant does not vary: telemetry waits on business cleanup, never the reverse.

## 3. How can staging evaluate separate HTTP, cron, and queue signals?

Capture broadly and page narrowly. An HTTP `409` caused by a concurrent reservation may be useful as a grouped event and useless as a wake-up. A queue exception on attempt one may be routine; the same error after attempt three, with a checkout stuck in `payment_pending`, can demand action. A cron exception matters, but a missing cron execution cannot be captured by an exception filter because no code ran.

That distinction suggests three policies rather than one generic “backend error” alarm:

1. HTTP: group by stable exception type and operation, preserve status, and page only when the rate or customer impact crosses an owned threshold.
2. Queue: capture each failed attempt with the same job and checkout identifiers, but alert on exhausted retries or an age limit, not every retry.
3. Cron: capture thrown exceptions and send a success ping to a Healthchecks-style monitor after the release run completes; alert when the expected ping is late.

No pulse, no proof.

Infrai can centralize exception groups behind one plain REST contract, keep the application contract fixed if the provider behind the capability changes, and use one key across its broader backend surface, reducing the checkout adapter's credential handling. Public discovery publishes full request and response schemas plus runnable examples, which makes that adapter inspectable before deployment. Its error surface supports capture, group queries, group detail, and resolution, so support staff can mark a fixed group resolved without deleting its event history. The catch is operational: it has no alert or notification routing, synthetic checks, heartbeat monitoring, source-map decoding, session replay, or distributed span-tree query. Polling group queries can feed an alert you own, while Healthchecks.io covers the silent cron case; teams that require those functions in one product should select a broader observability suite.

## 4. Which context data is safe to retain for a postmortem?

Grouping should answer “is this the same failure mode?” Use stable fields such as exception class, normalized operation, and top application frame; keep volatile values such as checkout IDs, tenant IDs, timestamps, and retry numbers as event context. Otherwise every affected property creates a new group, support sees noise, and the real blast radius disappears.

Resolution is a state transition. After a fix or rollback, mark the group resolved and keep its events for the postmortem. If it recurs, reopen or create a new incident according to the tool's semantics. Deleting history to clean a dashboard damages the evidence needed to compare pre-fix and post-fix behavior — and a tidy dashboard is not a reliability outcome.

Set a privacy boundary too. Error context should use internal identifiers rather than resident names, email addresses, card data, or lease documents. This matters especially for tools without a per-user log deletion API or bulk export path: data minimization at capture time is safer than hoping for a cleanup operation later.

## 5. How do the available options differ for checkout teams?

The useful comparison is not a feature-count contest. It is which failure can escape your chosen boundary at 3 a.m.

| Option | Best fit for this checkout | Material trade-off |
| --- | --- | --- |
| Sentry | NestJS teams that need JavaScript stack diagnosis, source maps, and application error workflow | Pair it with a heartbeat when a scheduler can fail to start |
| Datadog | Teams already correlating application errors with logs, metrics, traces, and managed alerting | A larger operating surface than a focused exception pipeline |
| Grafana | Teams with an existing open observability stack and a preference for assembling signals around shared telemetry | Integration and alert ownership remain engineering work |
| Better Stack | Teams seeking hosted logs, error context, and on-call workflows in one operational product | Validate framework-specific debugging depth against the checkout's needs |
| Rollbar | Teams centered on grouped application exceptions and triage workflow | Still validate cron liveness separately rather than inferring it from exception volume |
| Infrai | Teams that value a stable REST capability contract and centralized error-group operations across changing providers | Bring your own alert routing and heartbeat service; it is not suitable when replay, source maps, or span-tree investigation is mandatory |
| Healthchecks.io | Scheduled jobs where “did not run” is the primary failure mode | It complements exception capture rather than replacing HTTP or worker error tracking |

Stick with Sentry when decoded JavaScript stacks and replay are central to support investigations. Choose Datadog when the on-call decision depends on trace topology and existing monitors. Grafana fits teams prepared to compose their own observability stack, Better Stack fits teams looking for a hosted operational workflow, and Rollbar is a reasonable focused alternative for exception triage. Use the stable REST option when minimizing SDK and provider coupling matters more than an integrated on-call surface, and add Healthchecks.io wherever a missing execution is itself an incident.

Before rollout, run four failure injections in staging: throw before the reservation write, throw after the write but before commit, fail a queue attempt until retries exhaust, and suppress one expected cron execution. Verify the business state first, then the event group, then the page. The fourth test is the one an exception-only design cannot pass.

That is the decision rule: the capture tool documents failures that happened; rollback containment and heartbeat monitoring protect you from the failures it cannot observe.

## References

- https://docs.nestjs.com/exception-filters
- https://docs.nestjs.com/interceptors
- https://docs.sentry.io/platforms/javascript/guides/nestjs/
- https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/dd_libraries/nodejs/
- https://grafana.com/docs/
- https://betterstack.com/docs/
- https://docs.rollbar.com/docs/nestjs
- https://healthchecks.io/docs/
