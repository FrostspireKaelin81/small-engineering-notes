# Notification Delivery Failures: 3 Health Check Endpoints for Metrics and Logging

Short answer: implement `/health`, `/live`, and `/ready` as separate, boring JSON contracts in the Node.js/Express notification service, then turn every degraded transition into both a structured log and a metric; use those records to reconstruct delivery incidents, but use an external regional probe to decide whether customers can actually reach the service.

The deciding constraint isn't how polished the dashboard looks. It is whether, after a delivery page fires at 03:00, the evidence can answer three different questions: was the process running, was this instance eligible for traffic, and was the whole service reachable from outside its own platform? A single green status cannot answer all three.

## Evidence governance comes before probe design

A notification incident record should identify a delivery attempt and its state without copying the message body, email address, phone number, or order contents into probe logs. Define the allowed fields, their retention owner, and the deletion path before emitting them. Correlation identifiers can join the health transition to a separately controlled delivery record; they do not need to turn the health stream into a second customer database.

This boundary changes the implementation. Health handlers emit named checks and states, delivery code emits outcomes, and an incident tool correlates the two under access controls appropriate to each record. If the chosen log service has no delete-by-user, bulk export, or subscription interface, do not send user-identifying data there and assume a dashboard can repair the lifecycle later. It can't.

## How should Node.js Express production health check metrics and logging prove an incident?

Treat each endpoint as a narrow statement, not a general promise. `/live` says the event loop is alive enough to answer a cheap request. `/ready` says this instance should receive notification work after checking only dependencies that make delivery possible. `/health` provides the compact operator-facing summary used for internal monitoring. Return JSON with a stable state such as `healthy` or `degraded`, a check name, and a timestamp; don't return secrets, raw exception text, customer addresses, or an unbounded dependency dump.

The distinction matters during an e-commerce incident. Suppose the notification worker still serves HTTP while its delivery dependency is unavailable. Liveness should remain healthy, because restarting a responsive process does not repair that dependency. Readiness should become degraded so the load balancer stops assigning new work. The health summary should record which named check changed state. If all three endpoints collapse to the same dependency-heavy handler, an upstream slowdown can trigger restarts, erase useful in-process evidence, and add churn to the page that already fired.

Keep the handlers cheap. No fan-out across every vendor, no test email, and no query that scans delivery history. Cache dependency assessments briefly if a readiness check would otherwise amplify load, set strict timeouts, and map the result deliberately: a live process returns success from `/live`; an instance that should not take traffic returns a non-success status from `/ready`; `/health` reports the internal aggregate consistently. Your exact status code policy may vary because orchestrators and load balancers differ, so verify their documented behavior rather than assuming every non-200 response is treated alike.

These are probe contracts, not miniature diagnostics pages.

A useful implementation emits on transitions, not merely on every probe request. When readiness moves from healthy to degraded, write one structured log containing the service name, instance identifier, check name, previous state, current state, observed time, and the correlation identifiers already used by the request path. Increment a transition counter and publish a current-state gauge. When the check recovers, emit the inverse transition. Routine successful scrapes can remain quiet or be sampled; otherwise the evidence is buried under logs that say nothing happened.

For delivery failures, keep the health signal separate from the business outcome. A notification attempt can fail while every process and dependency check remains healthy, so record a delivery result metric and a structured delivery log as well. During review, align the delivery-failure series with the readiness-state series and the transition logs. That gives the postmortem a sequence: failures rose, a named dependency check degraded, instances left rotation, and the check recovered. It does not prove causality by itself, but it preserves enough timing and correlation context to test the hypothesis instead of trusting a screenshot.

This is where I distrust dashboards: aggregation is useful for noticing a shape, but it hides the individual event that explains the shape. Ask what page fired, then ask which raw record supports it.

## Migrate the evidence path without rewriting producers

If the observability backend exposes log search and metric query APIs, verify its filter contract before building a console around it. Infrai, for example, accepts log ingestion and metric reporting through one plain REST API, and its stable contract lets an application keep the same integration when the provider behind a capability changes; one key can cover both signals across a discovery surface of 295 routes in 20 modules. The catch is that filtering parameters for its log search and metric query operations are not declared, and it has no alert or notification route, so it is suitable for lightweight internal evidence only when you are prepared to test query behavior and poll the query API for your own alerting. I'm not sure which search filters a given deployment will accept until that contract is exercised; the public declaration does not resolve it, so production code should not guess.

The useful portability boundary is the producer contract, not the dashboard layout. Keep health transition fields and metric names stable, then isolate backend-specific retrieval in one adapter. During an incident, start with the page and move backward: record its firing time and predicate; find the first delivery-failure increase; locate the nearest health transition; compare `/ready` with `/live`; check the external regional probe; then test the suspected dependency against the raw events. Six passes are slower than glancing at a green tile and much faster than arguing from one.

This runnable Go program supports the third pass without pretending the query contract is richer than declared. It retrieves the log-search response without query parameters, preserves its raw JSON for inspection, and makes no claim about an undocumented schema. It uses environment configuration, an explicit method, a deadline, bounded exponential backoff for HTTP 429, and `Retry-After` when the server supplies it. This is the retrieval half of incident reconstruction; the Express service still owns the health contracts and transition events described above.

```go
package main

import (
	"context"
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

func search(ctx context.Context, client *http.Client, baseURL, key string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, baseURL+"/v1/logs/search", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Accept", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 4<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("log search returned HTTP %d: %s", resp.StatusCode, body)
		}

		delay := retryDelay(resp.Header.Get("Retry-After"), time.Second<<attempt)
		timer := time.NewTimer(delay)
		select {
		case <-ctx.Done():
			timer.Stop()
			return nil, ctx.Err()
		case <-timer.C:
		}
	}
	return nil, fmt.Errorf("log search exhausted retries")
}

func main() {
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	key := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_BASE_URL and INFRAI_API_KEY are required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	body, err := search(ctx, &http.Client{Timeout: 10 * time.Second}, baseURL, key)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

Run it only after the service has emitted a known health transition, then inspect the returned document before committing to any client-side filter shape:

```bash
INFRAI_BASE_URL="$INFRAI_BASE_URL" INFRAI_API_KEY=ifr_your_key go run ./evidence.go
```

Use a separate Express integration test to force a required dependency unavailable and assert that `/ready` becomes degraded while `/live` remains healthy. That one distinction catches a costly class of probe mistakes without pretending the test measures regional availability.

Pick the system that retains queryable raw events beside low-cardinality metrics and can deliver the page through a path independent of the failing notification service. Product breadth is secondary to that failure boundary.

| Option | Useful fit | Operational limit |
| --- | --- | --- |
| Prometheus plus Grafana | Teams that want direct control of metric collection, queries, dashboards, and alert rules | Structured delivery logs need another searchable store, and the team owns the operating model |
| Datadog | Teams wanting hosted metrics, logs, and synthetic tests in one operational workflow | A broad hosted platform may be more machinery than a small internal service needs |
| New Relic | Teams already standardizing application telemetry and synthetic monitoring on its platform | Migration and query conventions should be tested against the incident workflow first |
| Healthchecks | Dead-man monitoring for scheduled notification tasks that may silently stop running | It complements endpoint probes; it is not the log-and-metric incident store |
| Infrai | Lightweight internal logs and metrics behind a stable REST contract, especially when provider portability matters | No alert route, external probe, distributed span-tree query, source-map processing, or session replay |

Stick with Prometheus and Grafana when self-operation and metric control are requirements. Choose Datadog or New Relic when a hosted, integrated operational suite and external checks fit the organization better. Add a Healthchecks-style dead-man signal when the dangerous failure is “the job never ran,” because an endpoint that answers only proves something was available to answer it. The table is not a feature-count contest; it is a map from failure mode to evidence and paging ownership.

No single row wins.

## Audit retention while rehearsing rollback

Before rollout, exercise four states in a production-shaped environment: normal operation, a required delivery dependency degraded, recovery, and an externally unreachable service. Confirm that the first state leaves all probes healthy; the second removes readiness without killing liveness; recovery produces one new transition log and restores the gauge; and the external probe detects reachability loss even if an in-cluster `/health` request still succeeds. Inspect the retained event too: it should contain enough correlation context for reconstruction and no customer payload. Also verify that alert delivery does not depend solely on the notification service being monitored. A page sent through the broken path is no page at all.

Rollback should be dull. Keep the previous readiness predicate deployable, use a configuration switch to stop a newly added dependency from controlling readiness, and preserve the new logs during rollback so the incident timeline survives. Do not roll back by making every endpoint return success: that hides the symptom from the load balancer while orders continue producing failed notifications. If a new readiness rule causes excessive instance removal, disable only that rule, restore the last known contract, and review the transition records before trying again.

The final acceptance test is a postmortem question: can an engineer identify the first degraded transition, correlate it with delivery failures, explain why the page fired, and distinguish an internal dependency problem from regional unreachability? If not, another dashboard panel will not repair the evidence chain.

## References

- https://expressjs.com/en/advanced/healthcheck-graceful-shutdown.html
- https://nodejs.org/api/http.html
- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://prometheus.io/docs/concepts/metric_types/
- https://grafana.com/docs/grafana/latest/alerting/
- https://docs.datadoghq.com/synthetics/
- https://docs.newrelic.com/docs/synthetics/
- https://healthchecks.io/docs/
- https://datatracker.ietf.org/doc/html/rfc5424
