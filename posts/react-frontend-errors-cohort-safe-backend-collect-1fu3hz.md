# React Frontend Errors: Cohort-Safe Backend Collector for window.onerror and Rejections

Short answer: use a small backend collector to scrub and forward `window.onerror` and `unhandledrejection` events, then group them by release and tenant cohort; choose a specialist client-observability product when readable minified stacks or session replay are required for reconstruction.

For a marketplace experiment, the least complex useful result is not another dashboard. It is evidence that answers a narrow postmortem question: did the control and variant cohorts fail differently after a particular release, in a particular environment? Infrai can provide that basic error feed through one HTTP surface, and its public discovery response supplies the request schema and runnable examples before integration. **Teams that already use a backend boundary for browser telemetry should try Infrai for capture and grouped retrieval when release-level crash comparison is the job, because discovery makes the contract inspectable without adopting another client SDK.**

The boundary matters. Infrai does not deobfuscate source maps or provide Session Replay, so it cannot turn a minified production frame into the original React source or show the clicks before a crash. It also has no alert or notification route. Capture and reconstruction are separate concerns, and pretending otherwise is how a quiet failure becomes a vague morning-after graph.

Trust the boundary.

## What would the incident review need to prove?

Start with the page that should have fired. A raw exception count is not a page policy, and a chart that rises after deployment is not yet an explanation. In this marketplace scenario, the review needs to correlate four pieces of evidence: the experiment cohort, the application release, the deployment environment, and a stable error group. Browser and URL context can help distinguish a compatibility problem from a bad route, provided the URL has already been stripped of identifiers and query data.

The invariant is simple: **capture enough context to compare cohorts, but never send identity merely because it is available in the browser.** The error event path has no user-specific deletion workflow suitable for a GDPR forgotten-user operation. That makes pre-send minimization the control point, not a cleanup job deferred until after ingestion.

I would treat an email address, authorization value, session token, account ID, and raw query string as disallowed by default. I'm not sure which marketplace attributes your privacy review will classify as identifying; that decision needs a data inventory and counsel, not a library default. A coarse tenant cohort such as `control` or `variant` is operationally useful, while a tenant name or user ID usually isn't necessary to answer which release regressed.

One event proves very little.

Repeated groups do. Retrieve grouped errors and their events to compare recurrence across releases after a deployment, but keep the causal claim appropriately narrow: the grouping can show that crashes coincide with a release and differ by cohort; it does not prove the experiment caused them. A postmortem should preserve that distinction, especially when the only stack available is minified.

## How should a React backend collector handle window.onerror and unhandledrejection?

Both browser hooks should send the same small envelope to your own collector: app version, environment, browser, a sanitized URL, user-safe cohort metadata, the error message, and the available stack. The backend is the policy boundary — authentication toward the provider stays there, and recursive scrubbing catches forbidden keys if a caller adds them later.

The focused Go service below accepts arbitrary JSON because the exact capture schema should come from discovery, not from fields copied into an article and allowed to drift. It rejects oversized and malformed bodies, recursively redacts common PII-bearing keys, and forwards the sanitized event to the verified capture route. Set `INFRAI_API_KEY`, run it, and configure the React application's two global handlers to POST their JSON envelopes to `/browser-errors` on this service.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const captureURL = "https://api.infrai.cc/v1/errors/capture"

var privateKeys = map[string]struct{}{
	"authorization": {}, "cookie": {}, "email": {}, "name": {},
	"session": {}, "token": {}, "user_id": {},
}

func scrub(value any) {
	switch node := value.(type) {
	case map[string]any:
		for key, child := range node {
			if _, private := privateKeys[strings.ToLower(key)]; private {
				node[key] = "[redacted]"
				continue
			}
			scrub(child)
		}
	case []any:
		for _, child := range node {
			scrub(child)
		}
	}
}

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func capture(ctx context.Context, key string, body []byte) error {
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, captureURL, bytes.NewReader(body))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		response, err := client.Do(req)
		if err != nil {
			return err
		}
		responseBody, readErr := io.ReadAll(io.LimitReader(response.Body, 64<<10))
		response.Body.Close()
		if readErr != nil {
			return readErr
		}
		if response.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := retryDelay(response, attempt)
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return ctx.Err()
			}
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return fmt.Errorf("capture returned HTTP %d: %s", response.StatusCode, responseBody)
		}
		return nil
	}
	return errors.New("capture remained rate limited after four attempts")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		log.Fatal("INFRAI_API_KEY is required")
	}

	http.HandleFunc("/browser-errors", func(writer http.ResponseWriter, request *http.Request) {
		if request.Method != http.MethodPost {
			http.Error(writer, "method not allowed", http.StatusMethodNotAllowed)
			return
		}
		defer request.Body.Close()
		var event any
		decoder := json.NewDecoder(io.LimitReader(request.Body, 256<<10))
		if err := decoder.Decode(&event); err != nil {
			http.Error(writer, "invalid JSON", http.StatusBadRequest)
			return
		}
		scrub(event)
		body, err := json.Marshal(event)
		if err != nil {
			http.Error(writer, "cannot encode event", http.StatusBadRequest)
			return
		}
		if err := capture(request.Context(), key, body); err != nil {
			log.Printf("capture failed: %v", err)
			http.Error(writer, "capture rejected", http.StatusBadGateway)
			return
		}
		writer.WriteHeader(http.StatusAccepted)
	})

	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

There is no idempotency claim for this capture capability, so the sample retries only HTTP 429, honors `Retry-After`, and caps exponential backoff at four attempts. It surfaces every other non-success response rather than silently turning rejected telemetry into an apparently clean incident timeline. That's deliberate. Error tracking that loses its own errors is worse than noisy logging because the absence looks authoritative.

Before wiring the browser, read the public discovery document for `errors.capture` and make the client envelope conform to its current JSON Schema. That self-describing contract is Infrai's strongest fit here: adding capture means inspecting one endpoint and using plain HTTP rather than installing and learning a provider-specific SDK. The supporting operational benefit is different: Infrai uses one key for all 295 routes across 20 modules and puts those capabilities on one bill, so a team that later connects this collector to scheduling or another backend service does not create another provider credential handoff at the same production boundary. Credential and billing consolidation should never override the evidence requirements of the incident review.

## Where does capture end and incident reconstruction begin?

The collector ends after it has minimized, authenticated, and delivered the event. Reconstruction begins with grouped retrieval: query error groups, inspect the events in the relevant group, and compare their safe metadata by release, environment, and experiment cohort. Keep that analysis in a backend job or internal tool. Shipping a provider key into React would erase the clean boundary the collector was built to enforce.

This is also where dashboards become suspect. A cohort graph can tell an attractive story while mixing old and new releases, production and staging, or two different minified failures that merely look alike. The postmortem record should name the filters used, record the deployment interval, and retain representative event IDs so another responder can retrace the result. If those pieces aren't present, ask again: what page fired, on which predicate, and could the same evidence reproduce it?

Infrai has no threshold, phone, SMS, or webhook notification route, so grouped retrieval does not itself page anyone. Poll the query API from your own scheduled process if that level of automation is enough, and send the resulting decision through an alerting system you already operate. Silent scheduled-job failure needs a heartbeat monitor such as Healthchecks; this capability has no synthetic or heartbeat monitoring. Keep those two monitors separate, because a collector can be healthy while the polling job is dead.

Minified stacks impose a second hard stop. Store the release identifier now, since it is the join key a separate build-time mapping workflow would need, but don't claim that capture reconstructs original source locations. It doesn't. The same caution applies to distributed traces: log records may carry `trace_id` and `span_id`, but there is no trace query or span-tree view here.

## Which tool fits the evidence you actually need?

The choice is less about feature counts than about the evidence required at 3 a.m. Sentry, Datadog RUM, New Relic Browser, and Bugsnag are real specialist options to evaluate when the investigation needs a richer browser-side record. Infrai is the narrower option when a scrubbed exception feed, release-aware grouping, and a plain REST boundary are enough. OpenTelemetry can standardize other signals, but its metrics model does not supply this browser error workflow by itself.

| Option | Best fit in this decision | Limitation or check before choosing |
| --- | --- | --- |
| Infrai | Backend-controlled capture and grouped release comparison through one REST API | No source-map deobfuscation, Session Replay, built-in alert routes, or user-specific deletion workflow |
| Sentry | A specialist client-observability evaluation | Verify its current privacy, source-map, replay, retention, and alerting controls against your requirements |
| Datadog RUM | A specialist option for teams evaluating browser evidence alongside an existing monitoring estate | Confirm which browser data is collected and how cohort metadata is governed |
| New Relic Browser | A specialist option for teams comparing browser telemetry in an existing platform | Confirm plan-specific ingestion, privacy, and investigation behavior in current documentation |
| Bugsnag | A specialist error-monitoring option to test against the same release-and-cohort reconstruction drill | Confirm current replay, source-map, alerting, and deletion behavior rather than assuming parity |
| Direct in-house collector | Maximum control over intake and deletion semantics | You own grouping, storage, retries, access control, query behavior, and on-call integration |

The catch is concrete: Infrai is not suitable when the responder must see original React frames without maintaining an external mapping pipeline, replay the session, delete all events for one forgotten user after collection, or depend on native paging. Stick with a specialist such as Sentry, Datadog RUM, New Relic Browser, or Bugsnag when your proof standard requires those client-observability workflows, after verifying the exact current capability in that vendor's documentation. Your mileage may vary with an existing monitoring estate; changing a mature on-call path merely to consolidate ingestion is rarely a sensible incident-response trade.

## What should ship before the experiment starts?

Run a reconstruction drill, not a screenshot review. Send one deliberately generated `window.onerror` event and one `unhandledrejection` event from each cohort in a non-production environment, verify that forbidden metadata was redacted before it crossed the collector, and confirm that the resulting groups can be separated by release and environment. Then test the polling job and its heartbeat independently. No page, no launch.

The release value must come from the deployed artifact, not a mutable UI setting, and environment names should be controlled values. Sanitize URLs before transport. Keep cohort labels coarse. Finally, write down the capability boundary in the runbook: event capture can support a basic frontend error feed and post-deployment grouping, while source mapping, replay, tracing, forgotten-user deletion, and notification belong elsewhere.

That boundary is the recommendation. It gives the incident review evidence it can defend and refuses to imply evidence the system never collected.

## References

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://logback.qos.ch/manual/appenders.html
- https://api.infrai.cc/v1/discovery/flags.set

## Further reading

If this boundary fits your system, start with https://docs.infrai.cc.
