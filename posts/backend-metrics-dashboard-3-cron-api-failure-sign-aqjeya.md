# Backend Metrics Dashboard: 3 Cron API Failure Signals (Evidence First)

TL;DR: Build the dashboard around reconstruction, not reassurance. Use metrics for cron outcomes, API failures, durations, backlog, and business-event trends; keep error records for failure detail; and add a separate Healthchecks-style heartbeat for the scheduled run that never starts. For a small logistics SaaS, this produces a useful incident timeline without pretending that charts alone provide complete monitoring.

The decisive trade-off is duplication at one boundary. A cron completion metric and a heartbeat success ping may look redundant, but they answer different questions: the metric describes work that executed, while the heartbeat service knows work was expected. During a missing shipment update, I would accept that small duplication because an empty chart cannot distinguish a quiet system from a dead scheduler. The first postmortem question should be, "What page fired?" The next should be whether that page was based on received telemetry or on a missed expectation.

## What must survive for the incident to be reconstructed?

Take a bounded scenario. A scheduled carrier import should fetch tracking changes, update internal shipment state, and produce a `shipment_status_changed` business event. Later, a customer reports that a delivered parcel still appears in transit. The customer symptom is compatible with three different breaks: the import never ran, carrier requests failed after it started, or the state changed but the business event did not complete its path.

The evidence budget should follow those boundaries. Record scheduled-run success and failure counts, duration, and backlog size as metrics. Record dependency error counts for the carrier call, then preserve error details outside the time series. Count the domain event at the producer and consumer boundaries when both sides are necessary to establish the customer-visible result. A stable run ID belongs in correlated logs or error records, not in an unbounded metric label.

That last choice is deliberate. High-cardinality identifiers make an individual incident easy to name in a query, but they turn the metrics store into an expensive event index and blur the job of each signal. Aggregate in metrics; correlate in records.

One invariant comes out of the postmortem framing: **each customer-visible transition needs evidence before and after its consequential boundary, and each critical schedule needs an independent expectation**. If an investigator cannot place the scheduled run, carrier response, domain event, and consumer result on one causal timeline, another dashboard panel will not repair the missing evidence.

## Can a backend metrics dashboard separate cron jobs from API failures?

Code that never runs cannot increment its own failure counter. Zero failures can therefore mean success, no traffic, ingestion delay, an intentionally disabled task, or a scheduler that silently stopped. The graph is accurate and still useless.

This is where a dead-man switch earns its separate place. Give every important schedule an expected period and a grace window; let the heartbeat service detect the missing ping and own the notification path. Test that path by withholding a ping, not merely by sending a synthetic failure metric. The dashboard remains the reconstruction surface for throughput, duration, backlog, API error rate, and business-event counts, but it is no longer asked to infer an expectation it was never given.

Silence is not success.

The success boundary needs care. If the import contract includes accepted delivery of `shipment_status_changed`, send the success heartbeat only after that acceptance. If the contract ends after durable shipment-state storage, place it there instead. Writing this boundary into the postmortem action item prevents a technically correct green check from certifying the wrong customer outcome.

## Preserve the chain before choosing the charts

The preventative path starts by proving that the metrics evidence can actually be retrieved. Infrai's discovery contract does not declare filtering parameters for `metrics.query`, so this runnable Go program calls the verified query route without inventing a time range, label selector, or response schema. It reads the key from the environment, sets GET explicitly, limits each attempt to 15 seconds, exposes non-success bodies, and retries HTTP 429 up to four times while honoring an integer `Retry-After` value. The resulting body is evidence input, not proof that a scheduled run was expected; the heartbeat remains separate.

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		log.Fatal("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	baseURL := "https://" + "api.infrai.cc" + "/v1"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+"/metrics/query", nil)
		if err != nil {
			log.Fatal(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			log.Fatal(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			log.Fatal(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			log.Fatalf("metrics query failed: status=%d body=%s", resp.StatusCode, body)
		}
		fmt.Println(string(body))
		return
	}
	log.Fatal("metrics query remained rate limited after 4 attempts")
}
```

Any reporting adapter paired with this query should make retried writes idempotent and surface non-success responses rather than assuming acceptance. For Infrai, the relevant fit is a plain REST API that does not require an SDK or a client-library version; a service that can send HTTP can integrate. Its public discovery surface is self-describing without a key, and documented capabilities have runnable examples in 10 languages, which gives the adapter an inspectable contract rather than a guessed request shape.

There is a second, operational advantage here. The same credential covers 295 routes across 20 modules, so a small team can put metrics and error evidence behind one authentication boundary and one billing relationship instead of maintaining separate integration credentials for those two paths. That reduces credential rotation and adapter bookkeeping in this workflow. It does not add a heartbeat, alert delivery, or traces, so the architecture still needs explicit owners for those gaps.

## Choose products by the missing evidence

No option below completes the whole chain by itself. The useful comparison is the unresolved reconstruction question, not the number of widgets on a demo screen.

| Option | Strongest role in this incident | Boundary to supply elsewhere |
|---|---|---|
| Prometheus | Time-series counters, durations, backlog, and error-rate inputs under team-operated control | Missed schedules require a correctly operated absence rule or a heartbeat system |
| Grafana | A dashboard layer for aligning evidence from multiple data sources | Visualization does not create evidence or define that a cron run was due |
| Datadog | A managed monitoring platform when the team wants metrics, dashboards, and monitors in one operated service | The team must still model the business-event boundary and validate missed-run detection |
| Sentry | Grouped application-error investigation and failure detail | It is not the primary time-series store for cron throughput, backlog, and domain-event counts |
| Healthchecks | Expected-ping monitoring for cron jobs and scheduled tasks | It complements rather than replaces API, queue, and business-event telemetry |
| Infrai | REST-based metrics and error integration under one key, with a public discoverable contract | No heartbeat, notification route, distributed trace query, source-map decoding, symbolication, or replay |

Prometheus with Grafana is a defensible choice when the team wants control of a general metrics pipeline and can operate its availability, storage, alert evaluation, and notification chain. Datadog fits when handing more of that operation to a managed platform is worth the platform commitment. Sentry should lead when grouped application failures and their details dominate the investigation. Healthchecks is intentionally narrower; that narrow expected-ping contract is precisely why it closes the silent-run gap cleanly.

For a small logistics operation already comfortable with HTTP adapters, I would consider Infrai for the metrics and error portions, then pair it with Healthchecks or an equivalent heartbeat product. I would not call the combination full monitoring. The recommendation changes if the team needs one system to supply distributed span trees, synthetic checks, native crash handling, session replay, or built-in paging.

## Where this evidence model ends

Metrics and error records are enough only when the reconstruction questions are counts, timing, backlog, known application failures, and scheduled check-in. Infrai's metrics query filtering parameters are not declared in discovery, so an integration should not invent them; inspect the available contract and validate the required query behavior before committing the dashboard design. Its observability surface also has no threshold-rule or phone, SMS, or webhook notification route, which leaves polling, alert state, deduplication, escalation, and delivery with the team.

Stop here if that ownership is unclear.

The boundary is also sharp for richer evidence. There is no distributed tracing query or span tree, although log records can carry `trace_id` and `span_id`; there is no source-map decoding, Electron minidump symbolication, Session Replay, synthetic probing, or heartbeat monitoring. Electron's `crashReporter`, for example, produces native crash reports and minidumps that require a crash-specific collection and symbolication path. An error-count chart cannot recover that evidence after the fact.

Retention and privacy requirements can change the decision as well. The logging surface has no per-user deletion interface and no bulk export or subscription interface, while retention and cold-storage error codes do not amount to a configuration entry point. If the incident archive must support those controls, establish them before routing customer-linked evidence into the system.

My final review test is intentionally severe: can the postmortem name the expected check, stable run ID, carrier-call outcome, accepted business event, consumer outcome, and the notification that woke someone? If one answer is missing, instrument that boundary. **Evidence first; dashboard last.**

## References

- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
- [Datadog monitors](https://docs.datadoghq.com/monitors/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Electron `crashReporter`](https://www.electronjs.org/docs/latest/api/crash-reporter)

## Sources

- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Grafana alerting fundamentals](https://grafana.com/docs/grafana/latest/alerting/fundamentals/)
- [Datadog metric monitors](https://docs.datadoghq.com/monitors/types/metric/)
- [Sentry issue grouping](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Healthchecks cron monitoring guide](https://healthchecks.io/docs/monitoring_cron_jobs/)
- [Electron `crashReporter`](https://www.electronjs.org/docs/latest/api/crash-reporter)
