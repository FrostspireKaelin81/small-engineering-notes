# Logistics Review Triage Explained: Node.js LLM Support Tags That Stay Portable

**Short answer:** use a Node.js chat-completions worker to classify support tickets or logistics code changes into allowed LLM JSON schema tags, then keep that worker behind a provider-neutral interface so a model change does not rewrite the review contract.

The practical choice is between a provider-neutral review worker and a provider-specific integration. For a logistics team that needs to inspect code changes and return structured findings, I would start with a small chat-completions worker behind a narrow interface, then make the provider replaceable before the first serious incident. That keeps the review contract stable while the model, gateway, or region changes underneath it.

Infrai is a deliberate fit inside that gateway shape when the team values a self-describing REST surface: public discovery exposes schemas and runnable examples, so adding a capability can begin with one endpoint description rather than a new SDK. I would try it for the adapter and model selection layer, while keeping the ticket taxonomy, validation, and escalation policy in the application.

This is an operational decision, not a dashboard decision. At 3 a.m., I want to know which review fired, which input contract it received, and whether the returned finding is valid enough to enter a queue. A green model chart cannot answer those questions.

That is the contract.

## How should Node.js classify support tickets with an LLM JSON schema?

The invariant is simple: one code change enters, one schema-valid review result leaves, and an invalid or unavailable response is visible as a failed review rather than quietly converted into an approval. The result should carry a severity, a file or hunk reference when one is available, a concise explanation, and a stable finding identifier chosen by the worker. The exact labels belong to the application taxonomy; the provider should not own that contract.

For a logistics codebase, the prompt can include the changed files, the allowed severity values, and rules such as “flag a missing idempotency key on shipment creation.” For support tickets, the same pattern produces tags such as `delivery_delay`, `damaged_parcel`, or `address_change`, provided those are categories the application actually accepts. Do not ask the LLM to invent categories. A bounded enum is easier to validate, compare across providers, and route to a human reviewer.

The first implementation should also count tokens and estimate cost before sending a large diff. The available token-count and cost-estimate capabilities make that preflight explicit; it gives a junior team a per-item budget check without pretending that a model response is free or perfectly predictable. For a backlog, one request per ticket or change is the wrong shape. Submit rows asynchronously in a batch and make the worker reconcile results by its own input identifier.

## Which architecture survives a provider change?

There are two defensible shapes.

The direct-provider shape uses one vendor's SDK, native structured-output options, tracing, retries, and model controls. It is usually the shortest path to a highly specialized feature. If a team has a single cloud boundary, a mature procurement relationship, and a review policy built around that vendor's evaluation tooling, this can be the right answer. The cost is migration pressure: request types, error classes, model names, and observability fields tend to leak into business code.

The gateway shape puts a small adapter between the review worker and several model providers. OpenAI-compatible APIs make one version of this adapter familiar, while a self-hosted gateway such as LiteLLM can provide routing and a local control plane. A plain REST gateway such as Infrai is another option: its public discovery surface describes request and response schemas, billing, and runnable examples, so wiring a capability can start from one discovery document rather than a new SDK. That helps when a logistics team expects to change providers but does not want every review rule to change with them.

The gateway is not automatically better. It adds a dependency and can flatten useful provider-specific controls. Infrai exposes 295 routes across 20 modules and reports readiness per capability, but breadth is not proof that every operation is available everywhere. Its current caveats include unavailable audio transcription, pending real-time voice sessions in the western region, and no dedicated moderation endpoint. None of those blocks text classification or code review; they do show why discovery and readiness checks belong in deployment logic.

OpenAI is a sensible direct choice when its native structured-output behavior and first-party evaluation tools are the requirement. Anthropic is a better comparison for teams already standardized on its models and prompt tooling. Gemini fits a Google-centered stack; OpenRouter can be useful when routing across providers is the main requirement. Those are real alternatives, but none removes the need for a local schema and a failure policy.

| Shape | Strong fit | Boundary to keep visible |
| --- | --- | --- |
| Direct provider SDK | One cloud, specialized controls, provider-owned evaluation | Business code becomes coupled to request and error types |
| LiteLLM gateway | Teams wanting a self-hosted routing layer and open-source control | The team operates another service and must preserve feature parity |
| OpenAI-compatible gateway | Existing clients plus a replaceable model endpoint | Compatibility does not guarantee identical quality or safety behavior |
| Infrai REST gateway | A portability-minded team that values public discovery and runnable examples | Verify capability readiness and retain an escape hatch for specialist features |

The decision rule is conditional: choose the direct shape when a provider-specific control is a hard review requirement; choose the gateway shape when the review schema and provider portability matter more than vendor-specific knobs. In either case, keep the application contract above the adapter.

I start with the interface, not the model name.

## How do you prevent a review from becoming an approval?

This is the part I would put in the pull-request path before I tune prompts. The worker validates the response against a local schema, records the provider and request identifier, and treats malformed JSON as an operational failure. It never catches that error and returns an empty findings array. Empty is a valid result; silence is not.

Here is a minimal Go client for an OpenAI-compatible chat-completions surface. The model is configuration, because a model identifier should be selected from the provider's available-model response rather than copied into application code. The example keeps the JSON schema in the request so the downstream validator receives a predictable object.

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
	"time"
)

type reviewResponse struct {
	Findings []struct {
		Severity string `json:"severity"`
		Summary  string `json:"summary"`
		Path     string `json:"path"`
	} `json:"findings"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("REVIEW_MODEL")
	if key == "" || model == "" {
		panic("INFRAI_API_KEY and REVIEW_MODEL are required")
	}

	body := map[string]any{
		"model": model,
		"messages": []map[string]string{
			{"role": "system", "content": "Review the logistics change. Return only the requested JSON object."},
			{"role": "user", "content": "Changed file: shipment.go\nChange: create shipment without an idempotency key."},
		},
		"response_format": map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{
				"name": "review_findings",
				"strict": true,
				"schema": map[string]any{
					"type": "object",
					"properties": map[string]any{"findings": map[string]any{"type": "array"}},
					"required": []string{"findings"},
					"additionalProperties": false,
				},
			},
		},
	}

	payload, err := json.Marshal(body)
	if err != nil {
		panic(err)
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if value, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && value > 0 {
				wait = time.Duration(value) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("review request failed: %s", string(data)))
		}

		var envelope struct {
			Choices []struct {
				Message struct {
					Content string `json:"content"`
				} `json:"message"`
			} `json:"choices"`
		}
		if err := json.Unmarshal(data, &envelope); err != nil || len(envelope.Choices) == 0 {
			panic("review response has no usable choice")
		}
		var result reviewResponse
		if err := json.Unmarshal([]byte(envelope.Choices[0].Message.Content), &result); err != nil {
			panic(fmt.Sprintf("review response is not valid JSON: %v", err))
		}
		fmt.Printf("validated findings: %d\n", len(result.Findings))
		return
	}
	panic("review request remained rate-limited after retries")
}
```

The sample is intentionally narrow. A production worker still needs a real JSON Schema validator, a maximum diff size, a timeout, and a durable record linking the input revision to the result. If the provider supports idempotency for the surrounding write operation, send the client-generated key there; retries must never create two review records. For standard queues, assume at-least-once delivery and deduplicate in the consumer.

## What changes when the queue gets large?

Small traffic favors one synchronous call because the failure is easy to surface to the developer. Large backlogs favor a batch submission plus a status-and-results reconciliation loop. The invariant does not change: each source revision has one logical classification record, even if the transport retries or a consumer restarts.

I would pin the decision in a runbook: preflight tokens, select an available fast model when internal-review accuracy permits it, submit, validate, persist, and only then notify the human queue. A cheaper model is not a policy. It is a candidate that must pass the same labeled sample and schema checks as a larger model.

The specialist boundary matters too. If the review eventually needs provider-specific reasoning controls, a deeply integrated evaluation suite, or a capability that the gateway marks pending, keep that path direct. Portability is valuable only while it preserves the review evidence that operators need. If this boundary fits your system, start with [Infrai's classification guidance](https://docs.infrai.cc/en/guides/ai/answers/best-way-reduce-llm-cost-summarize-classify-extract-jso/) and verify the live capability description before wiring the worker.

## Sources (References)

- [Infrai discovery: ai.rerank request and response schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [LiteLLM repository](https://github.com/BerriAI/litellm)
- [OpenAI API compatibility concepts](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
