# Next.js SaaS Structured Logs: 6 Rollback Tests for a Budget Platform

Short answer: for a budget Next.js SaaS, choose a structured logging platform only after it passes a shadow-write trial against one complete nightly media run, and keep the old sink authoritative until the new one can reconstruct a failed asset from ingestion through publication. The deciding constraint isn't the prettiest dashboard. It's whether an operator can answer what page fired, find the affected `asset_id`, and reverse the migration without losing the evidence needed for the next postmortem.

For a budget Next.js SaaS, that trial is more useful than comparing feature grids for Sentry Logs, Axiom, Better Stack Logtail, and a generic hosted logs API. Each belongs in the candidate set if it meets the team's written requirements; none gets a pass on rollback mechanics. Run the same six tests, record the same evidence, and decide from the resulting incident transcript.

No vibes.

## Start from the incident page that must fire

Start with the failure the system must explain. In this media workflow, a scheduler opens a nightly run, workers fetch source files, a transcoder creates renditions, metadata is indexed, and a publisher makes each asset visible. A page such as “publication success ratio fell below the service objective” is actionable. “Log volume changed” usually isn't. The first page names a user-visible condition; the second announces that instrumentation exists.

The comparison should therefore use one fixed fixture and six pass/fail tests, not six loosely worded impressions:

| Test | Injected condition | Evidence required before cutover |
|---|---|---|
| Correlation | One asset fails after metadata indexing | One query follows `run_id`, `asset_id`, and `attempt` across services |
| Duplicate delivery | The same event is sent twice | Results expose the stable `event_id`; counts can be deduplicated |
| Backpressure | The receiver throttles or times out | The app path stays bounded and the local queue preserves order |
| Schema drift | One canary adds `rendition` | Old queries still work; the new field is discoverable |
| Erasure | An asset is tied to an erasure request | The team can locate and remove the relevant records under its policy |
| Rollback | The new sink is disabled mid-run | The previous sink still contains a complete incident timeline |

This matrix produces objective differences without pretending a universal winner exists. For every candidate, capture query text, result count, elapsed query time, export format, retention configuration, throttle behavior, and the operator steps needed to disable delivery. I'm not sure which candidate will win in a given environment, because event shape, daily volume, retention, and the team's tolerance for operating a buffer change the answer. A representative replay resolves that uncertainty; a marketing comparison does not.

Budget belongs in the worksheet, but not as a headline score. Estimate monthly ingest from a sampled nightly run, apply the retention period, add query and export assumptions, then compare that estimate with an actual invoice after the trial. Cardinality deserves its own row: Prometheus warns against labels with unbounded values, and the same operational instinct helps here even though logs and metrics are different data types. Keep identifiers in structured log fields for investigation, but don't casually promote `asset_id`, raw URLs, or request IDs into metric labels.

## Implement the event contract before choosing the sink

A rollback-safe migration begins at the event boundary. The application should create one canonical record before any transport-specific encoding, so both sinks receive the same timestamp, severity, schema version, and correlation keys. If each transport assembles its own payload, a dual-write trial compares two different instruments and tells you very little.

For the nightly job, the minimum useful record is deliberately boring: `event_id`, `occurred_at`, `service`, `environment`, `schema_version`, `run_id`, `asset_id`, `stage`, `attempt`, `outcome`, and `duration_ms`. Error details need a bounded machine-readable code plus a scrubbed message. Secrets, access tokens, full source URLs, and unreviewed user text don't belong there. A log search system is still a data store, and GDPR Article 17 makes erasure an operational concern rather than a future paperwork exercise.

Use an allowlist.

The following Go type makes the contract visible. A Next.js producer can emit the same JSON shape; Go is used here because a small relay keeps transport retries out of request handlers and pipeline workers.

```go
package eventlog

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"time"
)

type PipelineEvent struct {
	EventID      string    `json:"event_id"`
	OccurredAt   time.Time `json:"occurred_at"`
	Service      string    `json:"service"`
	Environment  string    `json:"environment"`
	SchemaVersion int      `json:"schema_version"`
	RunID        string    `json:"run_id"`
	AssetID      string    `json:"asset_id"`
	Stage        string    `json:"stage"`
	Attempt      int       `json:"attempt"`
	Outcome      string    `json:"outcome"`
	DurationMS   int64     `json:"duration_ms"`
	ErrorCode    string    `json:"error_code,omitempty"`
}

func NewEvent(runID, assetID, stage, outcome string, attempt int) PipelineEvent {
	key := fmt.Sprintf("%s:%s:%s:%d:%s", runID, assetID, stage, attempt, outcome)
	sum := sha256.Sum256([]byte(key))
	return PipelineEvent{
		EventID:       hex.EncodeToString(sum[:16]),
		OccurredAt:    time.Now().UTC(),
		Service:       "nightly-publisher",
		Environment:   "production",
		SchemaVersion: 1,
		RunID:         runID,
		AssetID:       assetID,
		Stage:         stage,
		Attempt:       attempt,
		Outcome:       outcome,
	}
}

func Encode(event PipelineEvent) ([]byte, error) {
	return json.Marshal(event)
}
```

The deterministic ID is a deduplication key, not proof that a delivery happened exactly once. Retries can produce duplicates; network failure can leave the sender uncertain whether the receiver accepted a request. Preserve that ambiguity in the design. Query counts should either deduplicate by `event_id` or state plainly that they count deliveries, while the pipeline's business state remains elsewhere.

## Roll out a reversible migration at 3 a.m.

Treat the transport switch as a deployment with a rollback flag. The existing destination remains primary, the candidate receives a shadow copy, and pipeline success never depends on both remote writes completing inline. A bounded local queue or relay absorbs short receiver delays; when that queue reaches its stated limit, the system emits a metric and pages on threatened evidence loss rather than hiding the condition in another log line.

This is where a lot of “simple logging” designs become an incident amplifier. A worker that waits indefinitely for two remote acknowledgements can stall the media run. A worker that fires two goroutines and forgets them can lose the only record explaining why asset `ast_18427` never published. The relay below puts a deadline on each attempt and treats any non-2xx response as a failed delivery. Production code also needs a durable bounded queue, jittered backoff, explicit drop policy, and queue-depth metrics; those aren't included because pretending an in-memory sample is durable would be dangerous.

```go
package eventlog

import (
	"bytes"
	"context"
	"fmt"
	"net/http"
	"time"
)

type Sink struct {
	Name     string
	Endpoint string
	Token    string
	Client   *http.Client
}

func (s Sink) Send(ctx context.Context, payload []byte) error {
	requestCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	req, err := http.NewRequestWithContext(requestCtx, http.MethodPost, s.Endpoint, bytes.NewReader(payload))
	if err != nil {
		return err
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+s.Token)

	resp, err := s.Client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return fmt.Errorf("%s delivery returned status %d", s.Name, resp.StatusCode)
	}
	return nil
}
```

Keep endpoint paths in configuration and use only the ingestion contract documented by the selected receiver. Don't infer a REST path from a product's object model. During shadow mode, transport failures go to the relay's retry and dead-letter path, while the old sink remains authoritative; after cutover, reverse those roles only after the rollback window has closed by an explicit decision.

The catch is operational weight. A durable relay adds disk sizing, saturation alarms, replay controls, and another component to patch. It is not suitable when the team cannot own that machinery. In that case, stick with a single sink and its supported client or collector until a managed buffering path has been tested. Dual writing from every Next.js request handler is not the cheaper substitute it appears to be — it spreads timeout, retry, and credential logic across the application.

## How should a budget Next.js SaaS verify hosted structured logging platforms?

Verification starts from the page and walks backward. Trigger the synthetic failed asset, confirm the expected service-level alert fires once, open the runbook, and execute its saved query using only the fields named in the event contract. The result must show the last successful stage, the first failed stage, every attempt, and whether publication occurred. Then repeat while the candidate sink is disabled at the relay. The nightly pipeline should continue, the bounded queue should become visible through a metric, and restoring delivery should replay events without changing their IDs.

Now test the part teams postpone: deletion and export. Select one synthetic `asset_id`, locate every matching record inside the declared scope, exercise the documented deletion process, and verify absence with the same search. Record who can request it, who approves it, what audit evidence remains, and how backups fit the policy. Article 17 contains conditions and exceptions, so legal counsel and the organization's data policy determine the final procedure; a search box alone doesn't establish compliance.

The rollback drill is short but strict. Disable shadow delivery, drain or preserve the candidate queue according to the written policy, verify that the old sink still reconstructs the full run, and confirm that no application deployment is required to change destinations. Rotate the trial credential after the exercise. If rollback needs edits in five services, it isn't a rollback plan yet.

Only promote the candidate after two representative nightly runs pass all six tests and an operator other than the implementer can follow the runbook. The number two is a release criterion here, not a claim of statistical confidence: one run proves the happy path once; a second catches state accidentally left behind by the first. Your mileage may vary for a pipeline that runs weekly or has large seasonal swings, so define a window that includes the operational states the team actually needs to recover.

## Evaluate the incident transcript before signing the contract

The final comparison artifact should look like a pre-mortem: “At 03:12, the publication page fired; at 03:16, the operator found run `run_20260816_01`; at 03:20, shadow delivery was disabled; at 03:27, the old destination confirmed the complete asset timeline.” Fill those timestamps from the drill, not from aspiration. Beside them, record query duration, missing fields, duplicate count, queue high-water mark, deletion result, estimated monthly usage, and rollback steps for every candidate.

Then choose the option whose failure mode the on-call team can detect and reverse. A hosted platform may be wrong when policy requires storage under direct organizational control; a self-hosted stack may be wrong when nobody owns upgrades, capacity, and recovery; a general hosted API may be wrong when the team needs an integrated incident workflow; an integrated suite may be wrong when export and independent replay are hard requirements. Those are constraints, not rankings.

The dashboard can wait. The page, evidence chain, and exit path cannot.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://gdpr-info.eu/art-17-gdpr/
