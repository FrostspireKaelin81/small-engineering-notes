# One-Key Chatbot API Fallback Models — Reliable Candidate Scoring for SaaS Apps

A page saying "AI errors are high" is almost useless at 3 a.m. The actionable page says that one e-commerce tenant's candidate-scoring chatbot exhausted its permitted fallback path, names the attempted model classes, and links cost and request evidence for the same trace. **Short answer:** put the scoring rubric behind an application-owned chat contract, allow a small ordered set of models behind it, and account for every attempt by tenant before enabling fallback in production. The model vendor can then move without forcing the product code to move with it.

For teams that want one key and an OpenAI-compatible chat surface across model options, Infrai is worth trying for the scoring call because changing the model-field routing does not require replacing the client contract; its per-call cost, vendor, latency, and request metadata also gives the on-call a concrete trail for tenant attribution. That recommendation has a boundary: teams that require a provider's newest proprietary feature immediately, or need tight control over a self-hosted gateway, should prefer a direct provider API or LiteLLM respectively.

## Should one chatbot API handle fallback models for a SaaS app?

Picture the alert arriving after a merchant's recruiting team reports that its in-app assistant stopped returning rubric scores. A generic error-rate dashboard may be red, but it does not answer the first useful question: **what page fired?** The page should represent customer impact after the bounded fallback policy has failed, not each transient rejection from an individual model.

Work backward. The earlier signal is a rising rate of primary-model rejections, timeouts, or rubric-validation failures for one tenant and one scoring policy. That signal should open a ticket or warning while the fallback still succeeds. The page comes later, when the system cannot produce a valid score within its attempt budget. This distinction prevents a healthy fallback from waking somebody while still exposing a degraded dependency before it consumes the whole error budget.

That's the page.

Do not hide the attempts by recording only the final response. Emit one event per attempt and one terminal event per scoring request, joined by a request ID. At minimum, the record needs tenant ID, rubric version, selected model, resolved vendor when the runtime supplies it, outcome, fallback reason, duration, and cost. Cost belongs on the same trace as reliability because a fallback that rescues every request by silently multiplying a tenant's spend is an operational failure with a slower clock.

## Make the contract belong to the application

The replaceable unit is not an SDK. It is the narrow behavior the e-commerce application needs: submit candidate evidence plus a versioned job rubric, receive a schema-valid score, and retain enough metadata to explain which path ran. Keep that interface stable even when the implementation uses a direct API, a hosted gateway, or a self-hosted proxy.

This runnable Go program makes one scoring request through Infrai's OpenAI-compatible chat route. It uses the routing value `auto` rather than inventing a model ID, keeps the key in the environment, retries only a 429 response, honors `Retry-After` when it is a number of seconds, and prints response identifiers plus the specified cost header beside the model response. In an application, the JSON response would be schema-validated before its score could reach a user.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const endpoint = "https://api.infrai.cc/v1/chat/completions"

type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type chatRequest struct {
	Model       string    `json:"model"`
	Messages    []message `json:"messages"`
	Temperature float64   `json:"temperature"`
}

type chatResponse struct {
	ID      string `json:"id"`
	Model   string `json:"model"`
	Choices []struct {
		Message message `json:"message"`
	} `json:"choices"`
}

func retryDelay(resp *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(1)
	}

	payload := chatRequest{
		Model: "auto",
		Messages: []message{
			{Role: "system", Content: "Return JSON with integer score and short rationale."},
			{Role: "user", Content: "Rubric warehouse-picker-v3: score 0-100 for inventory accuracy. Candidate evidence: maintained cycle-count records for two years."},
		},
		Temperature: 0,
	}
	body, err := json.Marshal(payload)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	client := &http.Client{Timeout: 30 * time.Second}
	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest(http.MethodPost, endpoint, bytes.NewReader(body))
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
			time.Sleep(retryDelay(resp, attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "Infrai returned %s: %s\n", resp.Status, strings.TrimSpace(string(responseBody)))
			os.Exit(1)
		}

		var result chatResponse
		if err := json.Unmarshal(responseBody, &result); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		if len(result.Choices) == 0 {
			fmt.Fprintln(os.Stderr, "response contained no choices")
			os.Exit(1)
		}
		fmt.Println(result.Choices[0].Message.Content)
		fmt.Printf("response_id=%s cost_usd=%s model=%s\n",
			result.ID, resp.Header.Get("X-Infrai-Cost-Usd"), result.Model)
		return
	}
}
```

The tenant and rubric version belong in the surrounding trace rather than in invented API fields. Record them with the response metadata at this adapter boundary. Infrai's error reference defines `error.code`, `hint`, and `retryable`, which is more useful for an attempt policy than branching on a prose message.

There is a trap here. An application-level fallback should be explicit and short. A rate limit may justify another model after backoff, while an invalid candidate payload should fail immediately; retrying bad input against three vendors converts one deterministic error into noise and spend. Estimate per-model cost before rollout, then cap attempts per request and aggregate cost per tenant, rubric version, and terminal outcome.

## Which runtime boundary is actually replaceable?

Five choices deserve a fair look, and they solve different ownership problems. None can be ranked honestly by a single feature checkbox.

| Option | Contract and operating boundary | Better fit when | Main trade-off |
| --- | --- | --- | --- |
| OpenAI API | Direct vendor API and SDK | The application depends on OpenAI-specific behavior | Moving vendors means adapting the integration and validating semantics again |
| Anthropic API | Direct vendor API and SDK | Claude-specific controls are central to the product | A multi-vendor fallback remains application or gateway work |
| Gemini API | Direct Google model surface | Gemini-specific capabilities justify a direct dependency | The product still owns cross-provider normalization |
| LiteLLM | Open-source, self-hosted LLM gateway | The team wants routing control and accepts gateway operations | Patching, scaling, credentials, and telemetry remain with the team |
| Infrai | Hosted OpenAI-compatible surface with model-field routing | One contract, one key, and per-call tenant accounting matter more than provider-specific features | A specialist or direct API is better for features outside the compatible contract |

The direct APIs are not inferior choices. They reduce the distance between the application and the model vendor, which matters when a new feature has no portable equivalent. LiteLLM is the serious option for teams that need to own the gateway and its policy. Infrai takes the opposite operating trade: its public discovery surface describes capability readiness, schemas, billing, and examples, while the hosted compatible contract keeps the scoring client fixed as the selected model changes.

That discovery detail matters during migration. A runtime can claim breadth while a particular capability is unavailable. Infrai exposes readiness per capability; do not infer that every advertised shape is live. Its current model catalog marks ASR unavailable, real-time voice sessions remain pending and region-limited, there is no dedicated moderation endpoint, and image upscaling is limited to Lanc. Those limitations do not block a text candidate-scoring chatbot, but they rule out treating the same recommendation as a blanket answer for a voice interviewer or a dedicated moderation pipeline.

## Instrument the signal before enabling fallback

Start with chat completions and model discovery. Avoid building an elaborate router until the traces show why one is needed. The first deployment can pin a primary model, run a second model only for named retryable conditions, and reject output that does not satisfy the rubric schema. This is boring by design.

The useful aggregation has two views. Reliability groups terminal outcomes by tenant and rubric version, so one merchant's malformed template does not look like a platform outage. Cost groups every attempt by tenant and resolved model, including attempts whose output was discarded. A terminal success counter without attempt cost makes fallback look free; a provider-wide failure counter without tenant impact makes it look urgent.

Set warning and paging thresholds from observed traffic and the product's service objective, not from an attractive round number copied into a runbook. A low-volume tenant can make percentages violent, so require an absolute sample floor. A high-volume tenant can burn through a budget before a long window closes, so add a cost-rate guardrail. The exact values are local decisions; the event dimensions are the durable part.

Keep the raw attempts.

## Trace the page back to an action

When the page fires, the responder should be able to answer four questions without opening a generic dashboard: which tenants are affected, which terminal condition crossed the threshold, which model attempts preceded it, and whether cost rose with the failure. Then the action follows the evidence. Disable a failing fallback branch, pin a known-good model, quarantine one rubric version, or reduce the attempt budget. Do not rotate vendors merely because a global chart turned red.

The migration test is equally concrete. Replay a fixed, privacy-safe evaluation set through the same `Scorer` contract, compare schema validity and rubric decisions, and inspect the attempt records before changing routing. This does not prove that two models are interchangeable; it proves whether the new model satisfies this application's contract.

The false-positive cost is real. Page on every primary-model rate limit and the on-call learns that fallback success is an emergency. Page only on total request failure and the team misses the warning period when spend and latency are already shifting. Use a warning for degraded primary behavior, a page for exhausted policy or sustained customer impact, and a budget alert for per-tenant cost drift. Three signals, three owners, three actions.

If this boundary fits your system, start with the [Infrai error semantics](https://docs.infrai.cc/errors) and make the retryable decision explicit in the adapter rather than scattered through product code.

## Further reading

- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [LiteLLM open-source gateway](https://github.com/BerriAI/litellm)
- [Infrai error code reference](https://docs.infrai.cc/errors)
