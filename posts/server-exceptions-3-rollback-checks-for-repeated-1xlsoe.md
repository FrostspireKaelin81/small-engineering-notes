# Server Exceptions: 3 Rollback Checks for Repeated Error Polling Alerts

A rollback is only as defensible as the evidence left behind it. TL;DR: capture Express and background-worker exceptions centrally, poll unresolved error groups over a recent time window, and alert on repeated failures while retaining the individual evidence needed to judge a rollback. A declining error count alone cannot tell you that a customer's scheduled media delivery recovered.

What page fired?

That is the opening question for a postmortem, before anybody shares a dashboard. A chart can show activity; the operational decision here is whether reverting a release is justified and whether the evidence will still support that decision afterward. The three checks are evidence preservation, a bounded repeat threshold, and a recovery observation tied to the affected job. They are design requirements, not measured outcomes.

## How should Express capture server exceptions before polling for repeated errors?

Consider a hypothetical production review: a media delivery worker reports three exceptions for the same error group during a five-minute window, shortly after a release, and a customer asks whether their scheduled delivery happened. This is an illustrative scenario, not an incident report. Walking through that review means asking what the alert establishes, what the individual exceptions establish, and what neither can establish.

Three exceptions establish repetition under the chosen grouping rule. They do not establish three affected customers, three distinct jobs, or a release regression. A retried job can generate repeated exceptions, and a group can contain failures from multiple attempts. Treating its count as a customer-impact count makes the rollback argument look stronger than its evidence.

Preserve the distinction.

For Express on Node.js, wire exceptions into central capture at the application's error boundary; include background workers in that capture plan because a scheduled delivery can fail outside an incoming request. Choose application evidence that can connect a captured exception to a delivery job and its attempted release, without copying customer content into the record. Those are requirements for the application's evidence model, not assertions about fields accepted by a particular capture API.

After a deployment is reverted, a responder should be able to reconstruct which delivery attempt failed, what evidence prompted the page, and what observation justified closing the incident. Capture first. A polling loop cannot reconstruct an exception that the application never recorded.

## Keep the evidence invariant separate from the alert threshold

The invariant is simple: changing the alert state must not erase the evidence that justified it. Resolving an error group, suppressing a duplicate notification, and deciding to roll back are different operations; design the record so a reviewer can understand each decision later.

A small Node worker can query unresolved groups every few minutes and send a Slack or email notification when a threshold is crossed. For this illustrative policy, use three matching exceptions in five minutes. Neither number is a universal recommendation. A low-volume delivery pipeline may need investigation after a single failure, while a noisy retry loop may require a separate impact signal before paging.

The window matters.

Before implementing the predicate, establish what the returned group count means. A lifetime total cannot stand in for occurrences during the last five minutes, and a poller's arrival time cannot establish when an exception occurred. Use occurrence evidence whose time semantics you have checked; retain enough information to explain the result. A polling interval also adds detection delay, so decide whether waiting a few minutes is compatible with the delivery deadline before calling the setup sufficient.

Group-based notification helps reduce per-event noise and gives a junior responder a smaller set of incidents to inspect. The trade-off is that grouping can hide variation within a group. Keep individual evidence available, and record why a notification was suppressed, particularly if suppression spans a deployment change that deserves fresh review. Re-reading the same events on consecutive polls must not turn one incident into a new page every time.

Rollback safety adds another requirement: stopping the exceptions is not proof of delivery. Define a recovery observation for the affected job before acting. A worker that stops running may produce a beautifully quiet graph.

## Compare the evidence model before choosing the tool

The alternatives serve different parts of this decision. Evaluate them against a concrete reconstruction exercise: retrieve the evidence for a failed delivery attempt, explain the page, and identify what would demonstrate recovery.

| Option | Relevant emphasis | Decision boundary for this workflow |
| --- | --- | --- |
| Prometheus | Metric naming practices for consistent measurements | Use metrics to describe repeat rates and operational signals; require separate event evidence before treating an aggregate as a reconstruction of a customer incident. |
| OpenTelemetry | Sampling concepts, including head and tail sampling | Inspect the sampling policy when deciding whether retained telemetry can support a reconstruction; do not assume a sampled view contains every failed attempt. |
| Healthchecks | Check-ins for scheduled work | Evaluate it for the silent-failure question: did the scheduled work report completion? Pair that question with exception evidence. |
| Sentry | Dedicated error investigation | Put it on the shortlist when error investigation is the primary workflow; test the required evidence retrieval and release comparison rather than assuming that exception counts establish delivery recovery. |

These are complementary choices. The table does not imply that each implements the same grouped-error capture interface, or that adopting every option is necessary.

Infrai combines a self-describing REST API with a single key and unified billing: one key, one bill across 295 routes in 20 modules. Public, unauthenticated discovery provides request and response schemas plus runnable examples, so an engineer can inspect a capability before wiring it into the worker. Separately, a media worker using capture alongside other backend capabilities can avoid maintaining separate credentials and reconciling multiple service invoices. That breadth is a coordination advantage, not evidence that every capability supplies the same incident-investigation depth.

For a capture-and-query design, keep the threshold evaluator and notification delivery explicit in the architecture. Select a dedicated error-monitoring workflow when your requirement includes source-map unminifying, crash symbolication, Electron minidump analysis, or session replay; a basic server-exception grouping workflow is too narrow for that investigation. Likewise, log correlation identifiers should not be treated as a substitute for a queryable distributed span tree.

A screenshot of a falling line isn't sufficient. The tool must help answer what failed and why the proposed recovery observation is sufficient.

## Preserve the evidence before evaluating the page

Start with this read-only grouped-error API probe. Set INFRAI_BASE_URL to the service's HTTPS origin, without a trailing path, and INFRAI_API_KEY to your secret key. Save the program in its own Go source file. It prints the actual response rather than guessing fields; inspect that response against discovery before connecting the window evaluator. The 20-second client timeout and three-attempt retry budget below are example choices, not service limits.

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

func run() error {
	base := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	key := os.Getenv("INFRAI_API_KEY")
	u, err := url.Parse(base)
	if err != nil || u.Scheme != "https" || u.Host == "" ||
		u.Path != "" || u.RawQuery != "" || u.Fragment != "" ||
		u.User != nil || key == "" {
		return fmt.Errorf("set an HTTPS service origin and API key")
	}
	client := &http.Client{
		Timeout: 20 * time.Second,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest(http.MethodGet, base+"/v1/errors/groups", nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
			delay := time.Second * time.Duration(1<<attempt)
			retry := resp.Header.Get("Retry-After")
			if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
				if seconds > 60 {
					return fmt.Errorf("rate limited; retry after %s", retry)
				}
				delay = time.Duration(seconds) * time.Second
			} else if at, err := http.ParseTime(retry); err == nil {
				delay = time.Until(at)
			}
			if delay < 0 {
				delay = 0
			}
			if delay > time.Minute {
				return fmt.Errorf("rate limited; retry after %s", retry)
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("HTTP %d: %s", resp.StatusCode, body)
		}
		_, err = os.Stdout.Write(body)
		return err
	}
	return fmt.Errorf("retry budget exhausted")
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

This probe retries reads only. It doesn't capture an exception, send a page, or assume that the response contains a recent-window count. A successful query supplies observations; the application still owns their interpretation.

The following standalone Go program illustrates the preventative ordering with a local fixture: write three example records to a new evidence file, sync it, and only then evaluate a five-minute repeat threshold. The local struct is an application-owned evidence format. It is not a vendor request or response schema, and the identifiers are invented fixture values.

Save it as a Go source file and run it with Go in a directory where creating the evidence file is appropriate. The program refuses to overwrite an existing file; use a fresh directory for another run. That deliberate constraint keeps a demonstration from silently replacing its previous evidence.

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"time"
)

type Evidence struct {
	EventID string    `json:"event_id"`
	GroupID string    `json:"group_id"`
	JobID   string    `json:"job_id"`
	Release string    `json:"release"`
	At      time.Time `json:"at"`
}

func run() error {
	now := time.Now().UTC()
	records := []Evidence{
		{"fixture-1", "delivery-error", "delivery-42", "candidate", now.Add(-4 * time.Minute)},
		{"fixture-2", "delivery-error", "delivery-42", "candidate", now.Add(-2 * time.Minute)},
		{"fixture-3", "delivery-error", "delivery-42", "candidate", now.Add(-time.Minute)},
	}

	file, err := os.OpenFile(
		"delivery-evidence.jsonl",
		os.O_WRONLY|os.O_CREATE|os.O_EXCL,
		0600,
	)
	if err != nil {
		return err
	}
	encoder := json.NewEncoder(file)
	for _, record := range records {
		if err := encoder.Encode(record); err != nil {
			file.Close()
			return err
		}
	}
	if err := file.Sync(); err != nil {
		file.Close()
		return err
	}
	if err := file.Close(); err != nil {
		return err
	}

	cutoff := now.Add(-5 * time.Minute)
	seen := make(map[string]bool)
	counts := make(map[string]int)
	for _, record := range records {
		if seen[record.EventID] {
			continue
		}
		seen[record.EventID] = true
		if !record.At.Before(cutoff) && !record.At.After(now) {
			counts[record.GroupID]++
		}
	}
	for group, count := range counts {
		if count >= 3 {
			fmt.Printf("review group=%s occurrences=%d evidence=delivery-evidence.jsonl\n", group, count)
		}
	}
	return nil
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

This example prints a review candidate after preserving the fixture. It does not send a notification or authorize a rollback. Its event-ID deduplication applies within this one batch; a deployed poller needs persistent state across polls, plus a durable notification identity so a delivery retry does not produce duplicate pages. Keep that notification identity separate from the captured-event identity: one explains delivery of the page, while the other explains which exceptions justified it.

For the real integration, inspect the capture and grouped-query schemas through discovery, then adapt verified occurrence records into the application model before evaluating the predicate. Do not guess payload fields or recent-window filters from their names. On rate limiting, honor Retry-After and use bounded backoff; on capture or notification writes, use a supported idempotency mechanism and check the response before recording success. A failed polling request must remain an unknown observation rather than becoming a zero-error sample.

The extra state is a trade-off. It makes a simple polling setup less disposable, but it lets the postmortem explain why a page was sent once, which records it covered, and why retrying the poll did not manufacture another incident.

## When the rollback rule should stop

A threshold crossing should request investigation, not automatically revert a release. The example has no evidence that the candidate release caused the errors, no successful-delivery observation, and no analysis of whether replaying an interrupted media job would double-deliver content. Require those answers before treating rollback as safe.

Silence proves little.

If the customer incident is a job that never started, add a scheduled-work check-in rather than expecting exception capture to detect an event that did not occur. If reconstruction depends on browser replay, native crash decoding, or a distributed span tree, choose an investigation workflow built for that evidence. For server exceptions that can tolerate polling delay, retain the occurrence records, page on a bounded repetition rule, and close the incident only after observing the affected delivery recover.

## References

These references distinguish aggregate measurements, sampled telemetry, scheduled-work checks, and error investigation. They support different parts of the reconstruction exercise; none establishes that the hypothetical delivery incident occurred.

## Sources

- Prometheus metric naming practices: https://prometheus.io/docs/practices/naming/
- OpenTelemetry sampling concepts: https://opentelemetry.io/docs/concepts/sampling/
- Healthchecks documentation: https://healthchecks.io/docs/
- Sentry documentation: https://docs.sentry.io/
