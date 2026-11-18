# Healthtech Email API 429 Errors: 3 Checks Before Suppressing Patient Addresses

Keep a rate-limited notification queued; suppress a patient address only after separate evidence establishes a recipient failure. That decision rule matters because HTTP 429 describes too many requests, while an email delivery failure describes a different stage of the journey. Combining them gives an incident responder a dangerously convenient explanation for a queue that suddenly looks empty.

TL;DR: To handle 429 rate limit errors for an email API in Node.js, inspect the HTTP response status, honor a valid Retry-After, and reschedule through a durable queue with bounded attempts and jitter. Coordinate sending across workers that share the limit. Keep bounce suppression independent, and preserve the application-owned template revision across retries. A throttled request supplies no evidence that a patient address is invalid.

For a healthtech reminder service, the primary design choice is who owns the template and its revision history. This runbook recommends application-owned, versioned templates so a delayed reminder can retain an auditable content identity while the worker rechecks whether the reminder is still appropriate to send. That is a design recommendation, not a claim about a particular service.

## What page fired, and what does it actually prove?

Start the incident review with the page: did it fire because reminder deadlines were at risk, because the oldest queued job was aging, or merely because a graph of 429 responses turned red? A dashboard can show a declining error count while workers quietly move valid patient addresses into a suppression table; request volume falls, the graph improves, and the reminder service has become less useful.

Check the evidence.

RFC 6585 defines 429 as Too Many Requests and permits a Retry-After field. It leaves how requests are counted and how the user is identified to the server. A single worker therefore can't infer the complete quota scope from that status alone. Check the documented contract for the actual account, credential, or other limit boundary before deciding which senders should pause together.

In Node.js, when using fetch, branch explicitly on response.status. An HTTP error response is available for inspection; a rejected fetch promise concerns a different failure path. A catch handler alone therefore doesn't implement 429 handling. Read Retry-After before releasing the response, hand the retry decision to the queue, and keep transport failures on their own path because an interrupted request may leave acceptance uncertain.

The three checks in this runbook are acceptance evidence, retry timing, and recipient evidence. They answer different operational questions. An API accepting a message is not proof that the destination mailbox received it, and a rate-limit response is not a bounce.

## Preserve template ownership at the queue boundary

Use an illustrative reminder job with an opaque job identifier, an internal recipient reference, a template key such as appointment-reminder, an immutable revision such as r17, and a delivery deadline. These are proposed fields, not an observed patient record. Store only the data the job needs, restrict access, and keep addresses and rendered reminder content out of routine retry logs. The application should own the mapping from that revision to the reviewed subject and body, as well as the rules that decide whether the reminder is still valid. Consider a synthetic test: the request receives a Retry-After of 30 seconds, the appointment is canceled during that interval, and revision r18 is deployed before the retry becomes eligible. A worker that merely replays its payload misses the cancellation; a worker that resolves the latest template silently changes the reviewed content. The desired decision is to skip the canceled reminder, retaining r17 in the job history so the incident review can still explain what was originally scheduled. On every attempt, resolve current suppression state and cancellation state before sending.

Pinning a template doesn't justify sending an obsolete appointment reminder. Content identity and send eligibility need separate checks.

There is a trade-off. Keeping templates in the application makes review and revision pinning part of the deployment workflow, which can slow copy changes, whereas delegating rendering creates a dependency on an external template revision contract. Choose deliberately: if a retry must reproduce reviewed content, require an immutable revision or retain the reviewed rendering under an appropriate retention policy. A mutable template name is insufficient evidence of what the worker intended to send.

A retry also needs a stable logical send identity. Preserve the same job identity across attempts, and use a documented idempotency mechanism when the sending interface provides one. Local deduplication helps coordinate workers, but it cannot by itself prove that a remote service did not accept a request before the connection failed. Keep that ambiguity visible for reconciliation; do not claim exactly-once delivery from a queue lock.

## How should workers handle email API 429 errors without suppressing patients?

For Node.js workers, implement the decision around the response before acknowledging the queue job. Persist the next eligible attempt, then acknowledge according to the queue's transaction or lease model. A process-local timer disappears with the process, and a loop that immediately retries can consume the same exhausted allowance repeatedly.

Retry-After has two forms under RFC 9110: a nonnegative integer number of seconds or an HTTP date. A valid future date supplies an earliest retry time, not a reason to occupy a worker until then. If that time falls after the reminder's deadline, expire or escalate the job according to the reminder policy. Never shorten a valid server delay to fit a local backoff cap.

The Go function below isolates that parsing rule for review. The Node.js dispatcher needs the same behavior; this is a decision helper, not a complete email worker. The upper bound is a local scheduling horizon, and the boolean result separates a parseable header from the decision to send the job to a holding queue. A valid delay beyond the horizon is held for review rather than replaced with a shorter delay.

```go
package retry

import (
    "net/http"
    "strconv"
    "strings"
    "time"
)

type Decision struct {
    Delay time.Duration
    Hold  bool
}

func RetryAfter(raw string, now time.Time, horizon time.Duration) (Decision, bool) {
    raw = strings.TrimSpace(raw)
    if raw == "" || horizon < 0 {
        return Decision{}, false
    }

    digits := true
    for _, c := range raw {
        if c < '0' || c > '9' {
            digits = false
            break
        }
    }
    if digits {
        seconds, err := strconv.ParseUint(raw, 10, 64)
        if err != nil || seconds > uint64(horizon/time.Second) {
            return Decision{Hold: true}, true
        }
        return Decision{Delay: time.Duration(seconds) * time.Second}, true
    }

    date, err := http.ParseTime(raw)
    if err != nil {
        return Decision{}, false
    }
    delay := date.Sub(now)
    if delay < 0 {
        delay = 0
    }
    if delay > horizon {
        return Decision{Hold: true}, true
    }
    return Decision{Delay: delay}, true
}
```

For a missing or malformed header, use your own bounded exponential backoff with jitter and an attempt budget. Select those bounds from reminder deadlines and the sending contract; the HTTP standard supplies no universal safe retry interval. With a valid header, add nonnegative jitter after its minimum delay so a group of workers does not resume at the same instant. A zero delay still passes through the shared limiter. It isn't permission for an unbounded retry loop.

Coordinate concurrency at the relevant limit scope. Reducing concurrency on one process while sibling workers keep sending can leave the queue under sustained pressure, so record the shared pause and make workers consult it before attempting another send. Persist attempts and next-attempt times across restarts.

Expired or exhausted jobs belong in an explicit review state with an owner, not in an infinite retry cycle.

Suppression requires a separate decision. RFC 3463 distinguishes permanent failures from persistent transient failures, but even a permanent failure can concern policy or message content rather than an invalid mailbox. Classify the specific delivery evidence and its recipient attribution before changing address eligibility. SPF, defined by RFC 7208, concerns authorization to use a sending domain; repairing sender authentication is a different action from suppressing a patient address.

## Verify the decision paths before widening the rollout

Use synthetic recipients and a controlled responder. The following cases are a proposed acceptance test set, not production measurements. They exercise the places where a retry change can accidentally become a delivery incident.

| Input or event | Expected decision | Evidence to inspect |
| --- | --- | --- |
| 429 with Retry-After of 30 | Schedule no earlier than 30 seconds later | Persisted attempt time; suppression unchanged |
| 429 with a future HTTP date | Wait until that date or hold the job | Parsed date and deadline decision |
| Missing or malformed Retry-After | Apply bounded fallback backoff | Attempt budget and jitter bounds |
| Valid delay beyond scheduling horizon | Hold for review | No shortened automatic retry |
| Recipient suppressed after enqueue | Skip dispatch | Current suppression decision |
| Reminder canceled during backoff | Skip dispatch | Current reminder eligibility |
| Connection lost after request transmission | Reconcile uncertain acceptance | Stable logical send identity |

Also exercise duplicate queue deliveries and a worker crash between receiving a response and recording the next action. Those tests need the actual queue and acceptance contract: a unit test of the parser cannot establish that acknowledgment ordering is safe. Verify that a content deployment during backoff preserves the pinned revision, and that withdrawing a revision blocks sends that should no longer use it.

Inspect the ledger.

For observability, record status class, retry reason, template revision, attempt count, and queue age with bounded labels. Keep opaque job identifiers in access-controlled event records instead of turning them into metric labels. Compare API acceptance, later delivery evidence, and suppression changes as separate signals; any claim that the incident is resolved should explain which of these recovered.

Roll out to a limited worker cohort that still obeys the shared quota control. Page on the risk the service owner needs to act on, such as reminders approaching their delivery deadline, and assign responsibility for held jobs.

A count of retries alone cannot tell the responder whether there is time left to recover.

## Roll back the dispatcher while preserving recipient evidence

If the rollout produces unsafe dispatch decisions, pause the affected consumers and preserve queued jobs, retry timestamps, attempt history, and template revisions. Restore the prior reviewed dispatcher only after checking it can read the current job schema and honor the shared pause. Restarting every worker at once can defeat the recovery plan.

A suppression mistake needs its own repair. Audit the affected decisions against recipient evidence, restore eligibility only where that evidence warrants it, and recheck reminder validity before replaying any job. Bulk clearing suppression can reintroduce addresses with genuine delivery failures.

The exit criterion is concrete: rate-limited jobs remain accountable until accepted, expired, or reviewed; valid patient addresses survive throttling; and every eventual send can be tied to its intended template revision. A green dashboard is supporting evidence. It is not the incident report.

## References

- RFC 6585, section 4: 429 Too Many Requests and rate-limit scope. https://datatracker.ietf.org/doc/html/rfc6585#section-4
- RFC 9110, section 10.2.3: Retry-After syntax and semantics. https://datatracker.ietf.org/doc/html/rfc9110#section-10.2.3
- RFC 3463: enhanced mail status codes and failure classifications. https://datatracker.ietf.org/doc/html/rfc3463
- RFC 7208: Sender Policy Framework. https://datatracker.ietf.org/doc/html/rfc7208
- MDN, Using the Fetch API: response status and rejected requests. https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
- Go net/http package: ParseTime. https://pkg.go.dev/net/http#ParseTime
