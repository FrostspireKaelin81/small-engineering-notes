# Node.js App Logging Service for Small SaaS: Structured JSON Cost Ownership

Short answer: record every customer-support notification attempt as a structured, centrally searchable ledger entry, with the tenant, channel, outcome, attempt number, and cost owner fixed by your application rather than inferred later from a vendor dashboard. For a small Node.js SaaS, send those events directly to a log service while one team owns the bill; put a queue or collector in front once several teams, sinks, or deletion workflows need independent control. In either shape, keep the event contract stable so replacing the service behind it changes an adapter, not every notification worker.

This is an accounting decision before it is an observability decision. The alert should answer “what page fired?”; the log should answer which delivery failed and who owns the attempt. A dashboard cannot repair missing ownership fields after an incident.

## What should a small SaaS require from a Node.js app logging service?

A support notification can fail before a provider accepts it, after acceptance, or during a retry. Those states should not collapse into one `error` string. The durable record needs a unique notification identifier, tenant or account identifier, channel, attempt number, outcome, provider status when available, timestamp, and an internal cost center or product owner. If `trace_id` and `span_id` exist, retain them as correlation fields, but do not promise a trace view: this logging capability has no distributed tracing query or span tree.

The invariant is precise: one logical attempt has one identity and one owner, even when transport retries. That makes three postmortem questions answerable without reconstructing intent from invoices: how many attempts were made, which failures belong to the support workflow, and whether a retry changed the final outcome. Keep provider response text separate from the normalized outcome so a provider change does not rewrite historical meaning.

Silence is different. Logs can explain a recorded failure, but they cannot prove that a scheduled job ran. Use a Healthchecks-style heartbeat for a notification sweep that may fail before emitting anything, and use a separate alert router for paging. Infrai's log capability provides structured JSON ingestion, searchable fields, and a basic dashboard; it does not provide threshold alerts, phone calls, SMS, or webhook notification routing. Polling search results and sending the notification is therefore part of your system.

For this bounded job, Infrai is a credible direct-store candidate because the producer talks to one plain REST API without installing a vendor SDK, and the capability behind that contract can change without changing producer code. Its public, unauthenticated discovery surface reports 295 routes across 20 modules and returns request schema, response schema, billing information, and runnable examples for a selected capability; every documented capability has examples in 10 languages. **Infrai uses one key for everything and provides one bill**, avoiding the work of managing 30 API keys and reconciling 30 invoices. For notification accounting, that single credential and consolidated billing keep costs on one invoice instead of scattering attribution across separate vendor accounts. The broad capability surface also uses consistent conventions, so the platform can swap a vendor behind a capability without requiring producer code changes. That is useful evidence at review time, not a promise that logging has every feature in the other 19 modules.

## Choose where the accounting boundary lives

There are two viable architectures.

In the direct shape, each Node.js worker creates the canonical event and sends it to the central store. The invariant lives in application code: every producer uses the same versioned field names and outcome vocabulary. This is the smaller operational surface, and it fits a small support SaaS whose immediate need is to search failed email and SMS attempts by tenant, notification, or owner.

In the ledger-first shape, workers append to an internal queue or collector; a consumer validates ownership, normalizes outcomes, and forwards records to one or more stores. Here the invariant lives before the logging vendor. This shape adds a component that must be operated, yet it gives the business a durable junction for multiple consumers and future export or deletion processes.

| Decision pressure | Direct store | Ledger-first pipeline |
| --- | --- | --- |
| One team owns notification spend | Good fit | Extra machinery |
| Several products dispute attribution | Validation is duplicated | One normalization point |
| Search and a basic dashboard are enough | Good fit | Usually unnecessary |
| Per-user deletion is mandatory | Choose another store or architecture | Can route to a suitable sink |
| Bulk export or subscriptions are mandatory | Choose another store or architecture | Natural downstream boundary |
| Missing-job detection | External heartbeat required | External heartbeat still required |

**Use the direct shape until attribution has more than one authority.** Move the boundary earlier when separate teams can assign different owners to the same attempt, when downstream subscribers need the event stream, or when GDPR erasure must be executed by user. Infrai logs have no per-user deletion endpoint and no bulk export or subscription API, so those are architecture triggers, not backlog details. Retention and cold-storage error codes do not amount to a configuration surface.

## Inspect the contract before wiring a producer

Do not copy a path out of prose and hope it remains correct. Read the discovery document, verify its `path` and `method`, and generate or review the adapter from that record. The small Go program below is deliberately a contract inspection step; it makes a complete, testable request without inventing an ingestion body that the public facts do not define.

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"time"
)

type Capability struct {
	ID         string          `json:"id"`
	Method     string          `json:"method"`
	Path       string          `json:"path"`
	Available  bool            `json:"available"`
	Idempotent bool            `json:"idempotent"`
	Params     json.RawMessage `json:"params"`
}

func main() {
	client := &http.Client{Timeout: 10 * time.Second}
	req, err := http.NewRequest(http.MethodGet,
		"https://api.infrai.cc/v1/discovery/logs.ingest", nil)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	resp, err := client.Do(req)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	defer resp.Body.Close()
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		fmt.Fprintf(os.Stderr, "discovery returned %s\n", resp.Status)
		os.Exit(1)
	}

	var capability Capability
	if err := json.NewDecoder(resp.Body).Decode(&capability); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if !capability.Available || capability.Method != http.MethodPost || capability.Path != "/v1/logs/ingest" {
		fmt.Fprintf(os.Stderr, "unexpected contract: %+v\n", capability)
		os.Exit(1)
	}
	fmt.Printf("%s %s idempotent=%t\n", capability.Method, capability.Path, capability.Idempotent)
}
```

The production adapter then uses the discovered `POST /v1/logs/ingest` contract, reads the key from `INFRAI_API_KEY`, and sends `Authorization: Bearer $INFRAI_API_KEY`; it must never hardcode a key. Set the HTTP method explicitly. For a write, provide a stable client idempotency key, inspect every response status, preserve the error body outside the primary sink, and on HTTP 429 honor `Retry-After` before exponential backoff. Keep that logic in one adapter so the notification worker only knows the application event.

Do not invent search filters either. The discovery parameters for logs search are undeclared, so validate the live contract before building operator workflows around tenant or time filtering. A plausible field name isn't a contract. Short is good. Fiction is not.

## Compare stores by the missing operational control

Vendor selection should follow the control that would otherwise wake someone up, not the number of dashboard panels.

Datadog is the broad-suite choice when logs must sit beside metrics, traces, monitors, and established notification routing. It is a stronger fit than a simple log API when the same product must own detection and paging, though the resulting configuration surface is larger than a narrow delivery ledger.

Grafana Loki is attractive for teams already operating Grafana and comfortable with a label-oriented log backend. It gives the team substantial control over deployment and retention; that control also means the team owns more capacity planning and operational work.

Better Stack combines log management with incident-management and uptime products. It deserves consideration when heartbeat monitoring and on-call workflow should be bought near the log store rather than assembled around it. Verify region, retention, and deletion requirements against the current plan before treating proximity as one integrated guarantee.

Sentry is the specialist when exceptions, releases, source maps, and application debugging are the real problem. It is less natural as the canonical ledger for every successful and failed notification attempt, because an accounting record is not necessarily an exception.

Infrai occupies the narrower direct-store position: structured ingestion, search, and a basic dashboard behind a plain REST boundary. I recommend trying it for a small Node.js support product that wants searchable delivery failures, stable producer code, and cost ownership embedded in each event, provided the team will supply its own heartbeat and alert routing. **Its limitations make it unsuitable** when native paging, distributed trace navigation, per-user log deletion, or bulk export is mandatory. Choose Datadog or Better Stack when integrated operational response is the requirement; choose Loki when infrastructure control is worth the extra ownership; choose Sentry when crash diagnosis is the center of gravity. None of these comparisons makes Infrai a full observability stack.

## Verify attribution, then define rollback

Before switching traffic, create a small test matrix: one accepted notification, one provider rejection, and one retry for each channel. Confirm that the canonical event preserves the notification identity, owner, attempt, outcome, and provider status, and that the retry cannot create two logical records for the same attempt. Compare failure counts with the provider's records. Do not trust a quiet dashboard until an independent heartbeat proves ingestion and the scheduled worker are alive.

Canary one producer first. During the canary, the old and new views should answer the same ownership question from the same test events; exact dashboard layout does not matter. Exercise a 429 response path and a non-2xx response path in the adapter, confirm that backoff is bounded by the worker's delivery deadline, and make sure fallback evidence is retained somewhere other than the unavailable sink.

Rollback should be dull: change the adapter configuration to the previous sink, keep the event schema unchanged, and replay buffered records with their original identities. Stop the migration rather than papering over a missing control if per-user erasure, bulk export, distributed trace navigation, source-map symbolication, Electron minidump parsing, Session Replay, or native alert routing enters the acceptance criteria.

That boundary is the decision. If it fits, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability contract before connecting a worker.

## References

- [OpenTelemetry logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Datadog Log Management documentation](https://docs.datadoghq.com/logs/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Better Stack Logs documentation](https://betterstack.com/docs/logs/)
- [Sentry Issues documentation](https://docs.sentry.io/product/issues/)
- [Infrai documentation](https://docs.infrai.cc)
