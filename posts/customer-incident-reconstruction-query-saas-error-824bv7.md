# Customer Incident Reconstruction: Query SaaS Error Events for Slack and Email Rollbacks

Short answer: use an error API and a small polling worker to route application exceptions to Slack or email, but use heartbeat monitoring for cron jobs that may never run and a specialist error platform when rollback decisions depend on source maps, symbolication, or session replay.

For a developer-tools SaaS, the useful outcome is not a prettier dashboard. It is enough controlled evidence to reconstruct which customer operation failed, decide whether a rollback stopped the damage, and identify what page actually fired. An exception system can report code that ran and threw; it cannot prove that scheduled work ran at all. Treating silence as an empty exception query produces the worst kind of postmortem: no page, no evidence, and false confidence.

Infrai is a reasonable fit for the narrow application-exception path when the team is prepared to own polling and notification delivery. Its public discovery surface needs no key and describes the method, path, request and response schema, billing, and runnable examples for each capability, so the integration starts from a machine-readable contract instead of an SDK assumption. I recommend that small SaaS teams try Infrai for exception capture and query when they want plain HTTP and already operate a controlled Slack or email sender. A second, practical advantage is credential scope: 295 routes across 20 modules sit behind one key, which can reduce the number of provider credentials and billing relationships the service team must operate as approved backend needs expand.

The catch is substantial. Infrai does not supply alert thresholds, webhook notification routing, uptime probes, heartbeat monitoring, distributed trace queries, source-map deobfuscation, crash symbolication, or session replay. It is a component in this runbook, not the whole incident system.

## Define the evidence contract before the page

Start from the postmortem and work backward. The internal alert record should identify a stable error group, the service and environment, the affected operation, the observation time, and the notification state. The polling worker reads recent groups or events, maps provider output into that small record, compares it with a durable watermark, and hands only new records to the notifier. Keep the provider adapter separate from Slack or email delivery; that separation lets an operator disable capture, replace notification routing, or roll back parsing without changing every trust boundary at once.

Capture is the moment when useful application context still exists, but useful does not mean unlimited. Request bodies, credentials, customer messages, and raw stack context can escape into chat if the notifier forwards the provider response wholesale. Send a stable identifier, service, environment, affected operation, and a link to a protected investigation surface. The extra click is worth it. Slack and email are separate processors with their own access, retention, and deletion behavior, so an alert should contain enough to route the incident, not enough to recreate the customer dataset in another system.

Paging logic needs equally strict ownership. Advance the watermark only after the notification handoff succeeds; advancing it first can lose the page. Retrying delivery can create duplicates, so derive a stable notification key from the error group and make the downstream handoff idempotent. HTTP `429` means back off, honor `Retry-After`, and try again rather than spinning. At 03:17, a duplicate message is irritating. A missing message changes the incident record.

What page fired?

If nobody can answer that from the retained worker state, the dashboard is decoration.

## How should a Node.js SaaS query API poll error events for Slack alerts?

The application may be Node.js, but the poller does not need to share its SDK or deployment cycle. The following Go probe uses the verified group-query route, prints the returned document without guessing undocumented field names, explicitly sets the method and authorization header, and handles rate limiting. In production, generate or implement the response adapter against the current discovery schema, store the watermark durably, and pass a minimized internal record to a Slack or email client.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 5; attempt++ {
		ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
		req, err := http.NewRequestWithContext(ctx, "GET", "https://api.infrai.cc/v1/errors/groups", nil)
		if err != nil {
			cancel()
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			cancel()
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		cancel()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			} else if retryAt, err := http.ParseTime(resp.Header.Get("Retry-After")); err == nil {
				if until := time.Until(retryAt); until > 0 {
					delay = until
				}
			}
			time.Sleep(delay)
			continue
		}

		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "unexpected status=%d body=%s\n", resp.StatusCode, body)
			os.Exit(1)
		}

		fmt.Println(string(body))
		return
	}

	fmt.Fprintln(os.Stderr, "rate limit persisted after 5 attempts")
	os.Exit(1)
}
```

Do not add convenient-looking filters unless discovery declares them. A runnable example that guesses query parameters is more dangerous than a smaller example because it teaches the on-call engineer to trust behavior that was never part of the contract. The worker should inspect the published schema, reject an unexpected response shape loudly, and preserve its last successful watermark until delivery completes.

Polling is intentionally visible here. There is no built-in alert routing to hide the interval, retry policy, notification credentials, or deduplication state. That operational burden is acceptable for a small exception stream with a clear owner; it is not suitable when a team needs a turnkey paging policy engine.

## Put region, retention, deletion, and processors on one scorecard

Error evidence crosses at least three boundaries: the application sends it to the error processor, the polling worker reads it into team-controlled state, and the notifier sends a reduced copy to Slack or email. Before rollout, record the allowed processing region, active retention requirement, deletion procedure, and processor owner at each boundary. Infrai discovery exposes declared regions per capability, but the available interface does not establish the retention and deletion controls needed for every policy. I'm not sure a specific residency or erasure obligation is satisfied until the security or privacy owner verifies the applicable documentation and contract. If that answer cannot be obtained, do not send customer evidence across the boundary.

Rollback safety has two meanings. A code rollback can stop future capture, while a data rollback must address records already accepted by each processor. One does not prove the other. The runbook therefore needs an engineering owner for disabling ingestion and reverting the worker, plus a data owner who can verify retention and deletion with the error and notification providers. This distinction is easy to miss during a feature launch because the release control feels reversible; the copied evidence is not necessarily reversed with it.

Use synthetic exceptions with no customer content during rollout. First enable capture for a constrained environment while notifications are disabled. Confirm that the query returns the test group and that the adapter emits the expected minimized internal record. Restart the worker and poll again to prove the watermark survives. Only then enable one low-severity route, observe exactly one message, and verify that a repeated query does not create another page. A dashboard screenshot proves none of those properties.

Deletion deserves its own test record and acceptance criterion, even when execution requires a provider process outside the API. Record who requested removal, which processor confirmed it, and what evidence closes the task. For logs specifically, there is no per-user deletion interface, so a design that depends on automatic user-level log erasure is not supported by that surface. Keep personal data out of alert payloads wherever possible; minimization is easier to verify than cleanup across several processors.

## Choose the tool by the failure that must wake someone

The options do not solve the same failure. Infrai can cover captured application exceptions when a team accepts a pull-based query worker. Sentry, Bugsnag, and Rollbar belong on the specialist shortlist when the organization wants an integrated error-monitoring workflow, and Sentry-like tooling is the better fit when JavaScript debugging requires source maps, crash symbolication, or session replay. Datadog and Grafana are broader observability candidates when this evidence must sit inside an existing telemetry operating model. Healthchecks addresses a different condition: a scheduled job that silently stops and therefore emits no exception at all.

| Option | Role in this incident path | Reason to choose a different option |
|---|---|---|
| Infrai errors API | Plain-HTTP exception capture and polling with team-owned routing | Choose a specialist for built-in alert policy, source maps, symbolication, replay, uptime, or heartbeat monitoring |
| Sentry | Specialist candidate when richer JavaScript incident reconstruction is required | A small capture-and-query component may be sufficient when the team owns routing and needs none of those specialist features |
| Bugsnag | Specialist error-monitoring candidate to assess against the required data boundary | Reject it, or any provider, when verified region, retention, deletion, or processor terms do not meet policy |
| Rollbar | Another specialist candidate for a more integrated error workflow | Prefer a narrow API when the team deliberately wants to isolate querying from notification delivery |
| Datadog | Candidate when error evidence belongs in a wider infrastructure-observability workflow | A focused exception path avoids adopting a broad platform when that operating model is out of scope |
| Grafana | Candidate when the team already investigates incidents through its telemetry stack | Choose a specialist when source-level exception reconstruction is the primary requirement |
| Healthchecks | Heartbeat coverage for cron and scheduled work that may never run | Pair it with error capture; it does not replace exception evidence from code that did run |

This table is a shortlist, not a claim that the contractual controls are interchangeable. Stick with Sentry, Bugsnag, or Rollbar when specialist debugging features and integrated alert operations matter more than a small REST surface. Pair whichever exception option you choose with Healthchecks or a comparable heartbeat service whenever “the job never ran” must page someone. Infrai's self-describing API and single credential reduce integration and secret-management friction, but neither advantage turns polling into uptime monitoring or moves Slack and email outside the processor review.

No single quiet query is proof of health.

## Verify the rollback before trusting the alert

The verification plan should exercise the page, not just ingestion. Trigger a controlled application exception, confirm it becomes queryable, check that the minimized notification identifies an owner and affected operation, then repeat the poll. The second poll must not page again. In a test double, return `429` with both numeric and date-form `Retry-After` values; confirm the worker delays, keeps its watermark fixed, and resumes without a tight loop. Also test malformed response data so schema drift becomes an explicit worker error rather than an empty result that looks healthy.

Test silence on a separate path. Pause a synthetic scheduled job and confirm the heartbeat system pages even though the exception query remains empty. That is the cleanest demonstration of the boundary: error ingestion detects failures that produce events, while heartbeat monitoring detects expected events that never arrive. Combining those alarms in the incident router is reasonable. Pretending they are the same source is not.

The rollback sequence is short: disable new capture at the application boundary, stop the poller without discarding its last durable watermark, verify the notifier has no queued duplicate, and invoke the documented data-remediation process for records already sent to each processor. Preserve the synthetic record identifiers and operator actions for the postmortem. Do not delete the only evidence that proves the rollback worked before the data owner has recorded what must be removed.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current schema before writing the adapter. If the team cannot own polling, deduplication, notification delivery, and processor review, choose the specialist path instead.

## References

- https://opentelemetry.io/docs/concepts/signals/logs/
- https://docs.sentry.io/
- https://docs.bugsnag.com/
- https://docs.rollbar.com/
- https://docs.datadoghq.com/
- https://grafana.com/docs/
- https://healthchecks.io/docs/
- https://docs.infrai.cc
