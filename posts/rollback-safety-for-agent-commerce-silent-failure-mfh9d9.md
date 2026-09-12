# Rollback Safety for Agent Commerce — Silent Failure, Cron Heartbeat, and Healthchecks

Rollback safety changes the monitoring design: error tracking, uptime monitoring, and a cron heartbeat answer different parts of a silent failure, while healthchecks alone cannot say whether reverting is the least dangerous action. A green storefront check cannot answer that when a scheduled AI agent is quietly accumulating latency, cost, or abandoned carts behind it.

**Short answer:** use error tracking for failed executions, uptime monitoring for externally reachable paths, a cron heartbeat for whether scheduled work arrived on time, and explicit healthchecks plus business-progress metrics for whether the agent loop completed useful work; rollback only when the deployment boundary and more than one independent signal point to the new release.

One page should name one failure. If the page can only say "the dashboard looks odd," it isn't ready to wake anyone.

## Read the agent loop backward from abandoned carts

Begin triage at the missing customer outcome and walk upstream. Consider a scheduled agent that reviews unresolved product questions, retrieves catalog data, calls a model, validates an answer, and queues the result for a human or the storefront. The HTTP edge can return `200`, the process can remain alive, and the scheduler can keep starting runs while the useful outcome falls to zero. No panic is required. A retry policy might keep consuming model calls; a malformed catalog response might be handled as an empty result; a queue consumer might accept work faster than the final publisher can drain it. From outside, the service is up. From the merchandiser's point of view, it has stopped.

That is the operational difference between **execution failure, reachability failure, arrival failure, and progress failure**. Error tracking observes exceptions, rejected operations, and captured error context. Uptime monitoring asks whether a chosen external request succeeds within its policy. A cron heartbeat proves that a scheduled producer checked in before a deadline. A healthcheck reports a narrowly defined dependency or internal readiness condition. None of those, alone, proves that a product question became an acceptable answer at a controlled latency and cost.

Silence is data.

The dangerous case is a handled path that produces no error event. Suppose validation rejects every generated answer and the loop records those rejections as normal outcomes. Error tracking can be perfectly quiet. An uptime probe can still pass. The cron heartbeat can arrive at the start of every run. Only a completion heartbeat, a growing oldest-item age, or a falling ratio of accepted answers to attempted answers exposes the stalled business flow. This is why a start heartbeat should never be treated as a completion signal — it proves scheduling, not useful completion.

The reverse trap matters too. One malformed item can create a rich error event while the batch continues and meets its service objective. Paging on that single event turns error tracking into a noise machine. Record it, correlate it, and investigate the pattern; reserve the page for user-visible risk or a burn rate that demands action. The pager question is not "did an error happen?" It is "what customer outcome is in danger now?"

## What can error tracking, uptime monitoring, cron heartbeats, and healthchecks actually prove?

Start with a failure-mode table, not a dashboard. Every row needs an owner, an observation point outside the component being judged, and a response that can actually change the outcome. The following matrix is deliberately small enough to use during a rollout:

| Signal | Question it answers | Typical blind spot | Rollback value |
| --- | --- | --- | --- |
| Error tracking | Which execution failed, where, and with what context? | Handled failures and work that never started | Strong when new error groups align with the release |
| Uptime monitoring | Can a user-like request reach a critical edge path? | Async work behind a healthy edge | Strong for a new edge regression; weak for an agent backlog |
| Cron start heartbeat | Did the scheduled run begin before its deadline? | A run that starts but stalls or produces nothing | Strong for scheduler or startup regressions |
| Cron completion heartbeat | Did the run finish within its allowed window? | A completed run with useless output | Strong when duration changes at the release boundary |
| Healthcheck | Is this process ready, and are selected dependencies usable? | End-to-end business correctness | Supporting evidence, not a business outcome |
| Progress and cost metrics | Did attempts become accepted answers within the operating budget? | Root cause and stack context | Strong for impact; pair with traces or errors for cause |

Keep liveness and readiness narrow. A liveness check that fails because a remote model or catalog dependency is briefly unavailable can cause restarts that add load without repairing the dependency. A readiness check may remove an instance from service while it cannot safely accept new work, but a scheduled worker often needs a separate admission rule: don't begin another agent loop when the previous run is still active or when the downstream queue has crossed its declared capacity policy.

The external probe should exercise a path that matters without mutating a real order, publishing an answer, or spending model budget on every poll. The internal checks can be more specific, but they shouldn't pretend to be end-to-end evidence. It's tempting to combine them into one giant `/health` verdict; don't. A single boolean erases the distinction between "cannot accept traffic," "cannot finish background work," and "business output has degraded," which leads directly to unsafe automated rollback.

For the agent itself, preserve correlation across schedule, run, item, model attempt, validation, and publish stages. Metrics answer how much and how long. Traces show where the loop waited. Error events preserve diagnostic context for exceptional paths. Logs provide discrete records when their fields and retention are controlled. A custom logging appender can send events to another destination, but logging transport is still not a substitute for a progress invariant; the Logback appender model is useful evidence of how emission and delivery are separate responsibilities, even though the example implementation below stays in Go.

## Make release identity part of every observation

The safest rule is boring enough to test. Mark each observation with a release identifier and run identifier, compare the candidate against an agreed baseline, and require corroboration. A latency regression plus missing completion heartbeats is actionable. A cost jump plus a sharp fall in accepted outputs is actionable. A lone readiness failure might call for traffic removal and investigation, not an immediate rollback.

This Go example keeps collection out of the decision function. The thresholds are policy inputs, not universal constants, and the sample doesn't claim that one set fits every store. It also refuses rollback when evidence is stale or split across releases — a common way automation converts unrelated noise into a second incident.

```go
package rollback

import "time"

type Snapshot struct {
	Release              string
	ObservedAt           time.Time
	CompletionLate       bool
	ErrorRateExceeded    bool
	P95LatencyExceeded   bool
	CostPerAnswerExceeded bool
	AcceptanceRateLow    bool
}

type Policy struct {
	MaxEvidenceAge time.Duration
	MinFailures    int
}

type Decision struct {
	Rollback bool
	Reasons  []string
}

func Evaluate(now time.Time, candidate string, s Snapshot, p Policy) Decision {
	if s.Release != candidate || now.Sub(s.ObservedAt) > p.MaxEvidenceAge {
		return Decision{}
	}

	reasons := make([]string, 0, 5)
	checks := []struct {
		failed bool
		name   string
	}{
		{s.CompletionLate, "scheduled completion is late"},
		{s.ErrorRateExceeded, "error-rate policy is exceeded"},
		{s.P95LatencyExceeded, "p95 latency policy is exceeded"},
		{s.CostPerAnswerExceeded, "cost-per-accepted-answer policy is exceeded"},
		{s.AcceptanceRateLow, "accepted-output rate is below policy"},
	}

	for _, check := range checks {
		if check.failed {
			reasons = append(reasons, check.name)
		}
	}

	return Decision{
		Rollback: len(reasons) >= p.MinFailures,
		Reasons:  reasons,
	}
}
```

Two details carry most of the safety. First, `CostPerAnswerExceeded` uses accepted answers as the denominator rather than raw calls, so retries and rejected output remain visible. Second, the release match prevents an old heartbeat miss from voting against a new deployment. Real systems also need a minimum sample policy and an explicit no-data state; zero observations must not be silently converted into a healthy zero rate.

I'm not sure a fixed two-signal rule is right for every traffic shape. It depends on event volume, batch frequency, and how costly a false rollback is. Resolve that uncertainty with replayed production-shaped events and staged rollout evidence, then encode the chosen minimum in version-controlled policy. Don't let an on-call engineer discover the rule by reading five panels at 3 a.m.

The limitation of automated rollback is compatibility: it is not suitable when schema changes are irreversible, a queue contains release-specific payloads, or reverting code would strand in-flight work. In those cases, stop admission, route to a compatible worker, or disable the affected capability with a pretested control while preserving evidence. Rollback is a response mechanism, not a moral verdict on the release.

## Rehearse stop, drain, reverse, and recover

Test the monitoring path as part of deployment. Inject one controlled condition at a time in a non-customer environment: suppress the scheduled start, allow a run to start without completing, reject every candidate answer, slow a model stub past the latency policy, and increase the test cost accounting input. Each condition should reach the expected recorder, alert, owner, and runbook. The test passes only if clearing the condition also clears the alert; a page that cannot resolve cleanly will be ignored during the next incident.

Then perform a canary rollback drill. Confirm that the candidate stops receiving new work, compatible in-flight items finish or return to a known queue state, the prior release resumes, and every deciding signal returns inside policy. Watch the negative space — duplicate publishes, abandoned leases, and cost continuing after admission closes. The desired result is not merely a green deployment controller. It is restored business progress without creating a second failure mode.

The catch is low volume: some stores may not produce enough accepted answers during a short canary to support rate comparisons. Use deterministic synthetic work that cannot reach customers, extend the observation window, or require a human decision. Don't lower a threshold until noise looks decisive. For infrequent daily jobs, a heartbeat deadline and a completion deadline can be more informative than a percentage based on one run, while error context remains essential for diagnosis after the page fires.

Write the rollback note as a miniature postmortem before anything fails: trigger, evidence, action, stop condition, and owner. If the trigger says "latency is high," it is incomplete. If it says "the candidate release breached the approved p95 policy while completion lateness also fired, and reverting preserves queue compatibility," someone carrying the pager can act without inventing architecture under pressure.

No mystery panel.

## References

- https://logback.qos.ch/manual/appenders.html
- https://opentelemetry.io/docs/specs/otel/metrics/
- https://prometheus.io/docs/practices/instrumentation/#batch-jobs
- https://sre.google/sre-book/monitoring-distributed-systems/
