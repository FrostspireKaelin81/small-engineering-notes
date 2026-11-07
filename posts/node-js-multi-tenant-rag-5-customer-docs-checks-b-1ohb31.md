# Node.js Multi-Tenant RAG: 5 Customer Docs Checks Before Swapping Embeddings

Keep tenant and document-permission filters ahead of retrieval when a Node.js SaaS scores customer-support candidates against a job rubric, and keep that authorization contract fixed when changing model providers.

Short answer: attach `tenant_id` and document permissions to every embedded chunk; derive the allowed scope from the authenticated customer, filter before or during retrieval, then rerank the permitted passages and generate a cited answer. A namespace can organize customer data, but the runbook must still account for document permissions. Provider portability ends at the point where a replacement changes who can see a passage.

Contain the evidence.

For this workflow, the useful incident question is not whether the model returned a plausible score. It is whether the score used only the requesting customer's authorized candidate documents and rubric. A polished answer can conceal the wrong evidence just as effectively as a dashboard can conceal the wrong alert.

## 1. Define the containment boundary before changing providers

Treat candidate scoring as an ask-your-docs request: retrieve evidence from a candidate's application and the customer's job rubric, then produce a score that cites the supporting passages. The application owns the customer boundary. A model selection field must not get to redefine it.

Consider an illustrative fixture with 2 customers, Cedar and Harbor. Each has a support-role rubric and a candidate document, and both use the local document identifier `candidate-17`. Cedar's rubric asks for evidence of escalation judgment; Harbor's asks for evidence of written troubleshooting. These are example records, not observed hiring results. Their overlapping identifiers make a useful containment check: a lookup keyed only by that local identifier is ambiguous.

The metadata needs the tenant, document identity, and document permissions together. Candidate scope also belongs in the application's request policy; tenant isolation alone does not authorize every recruiter to inspect every candidate.

An embedding is not an authorization decision.

Keep the logical contract stable across adapters: authenticated scope goes in, permitted passages with document identities come out. If a provider switch forces the Node.js handler to reconstruct permissions from generated text, the adapter boundary is in the wrong place. This is a proposed application design, not a claim that every retrieval service accepts the same filter syntax.

## 2. How should a multi-tenant Node.js SaaS filter customer docs before RAG?

Define the page around an authorization violation: a passage outside the authenticated tenant or document permissions reaches the reranker or answer generator. The corresponding containment action is to stop that request before generation. A later citation check is useful evidence, but it cannot undo disclosure to a downstream component.

Ask what page fired.

A success-rate chart will not establish isolation. Nor will a fluent answer with a familiar filename. In the fixture, both customers have a candidate document with the same local identifier, so the verification record must distinguish the customer and authorized document, while avoiding copying candidate text into routine alerts. Record the retrieval decision and enough identifiers to investigate it under the application's access policy.

One specific trap deserves attention: retrieving a global shortlist and filtering it afterward puts the authorization boundary too late. It also leaves the shortlist dependent on documents the caller could never read. Apply the restriction before or during retrieval; rerank only what survives that boundary.

A namespace and a metadata filter answer different questions. Use the namespace to select the intended customer partition where the retrieval system supports that organization, then ensure document permissions remain part of the enforced retrieval scope. Do not assume that choosing the correct partition settles access to every document within it.

## 3. Make the scope check executable

The following Go program exercises the authorization contract and then makes one answer-generation request using authorized fixture evidence. It performs no semantic search. Its job is smaller: reject an unscoped principal, admit the permitted Cedar record, and reject records from Harbor or a different permission group before sending evidence downstream.

The principal is a test input. In the deployed service, populate it from verified authentication and server-side authorization, never from a customer-supplied tenant field alone. Translate the resulting scope into the retrieval backend's supported restriction, and retain a defensive check on the returned passages before reranking. For this standard-library Go example, set `INFRAI_API_KEY` and `INFRAI_BASE_URL` in the environment, using the deployment's OpenAI-compatible v1 base URL without a trailing slash, then run the saved file with `go run main.go`. An SDK is preferable in a larger client; the explicit HTTP boundary here makes status handling and retry behavior visible.

```go
package main

import (
	"bytes"
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Principal struct{ Tenant, Group string }
type Chunk struct{ Tenant, Document, Permission, Text string }

func allowed(p Principal, c Chunk) bool {
	return p.Tenant != "" && p.Group != "" &&
		c.Tenant == p.Tenant && c.Permission == p.Group &&
		c.Document != ""
}

func run() error {
	p := Principal{"cedar", "support-hiring"}
	evidence := Chunk{"cedar", "candidate-17", "support-hiring",
		"Explains when to escalate a support ticket."}
	if !allowed(p, evidence) ||
		allowed(p, Chunk{"harbor", "candidate-17", "support-hiring", "Private"}) ||
		allowed(p, Chunk{"cedar", "candidate-18", "restricted-hiring", "Private"}) ||
		allowed(Principal{}, evidence) {
		return fmt.Errorf("authorization fixture failed")
	}

	key := os.Getenv("INFRAI_API_KEY")
	base := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	if key == "" || !strings.HasPrefix(base, "https://") {
		return fmt.Errorf("configure an API key and an HTTPS v1 base URL")
	}
	prompt := "Fixture rubric: score escalation evidence from 0 to 2; " +
		"0 means absent, 1 mentions escalation, 2 explains a threshold. " +
		"Use only the following evidence, cite its document ID, " +
		"and identify missing evidence. Document " +
		evidence.Document + ": " + evidence.Text
	payload, err := json.Marshal(map[string]any{
		"model": "auto",
		"messages": []map[string]string{
			{"role": "user", "content": prompt},
		},
	})
	if err != nil {
		return err
	}
	var nonce [16]byte
	if _, err := rand.Read(nonce[:]); err != nil {
		return err
	}
	idempotencyKey := hex.EncodeToString(nonce[:])
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{
		Timeout: 30 * time.Second,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			base+"/chat/completions", bytes.NewReader(payload))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)
		res, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(io.LimitReader(res.Body, 1<<20))
		res.Body.Close()
		if readErr != nil {
			return readErr
		}
		if res.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Second * time.Duration(1<<attempt)
			retry := res.Header.Get("Retry-After")
			if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			} else if date, err := http.ParseTime(retry); err == nil {
				if remaining := time.Until(date); remaining > 0 {
					delay = remaining
				}
			}
			timer := time.NewTimer(delay)
			select {
			case <-ctx.Done():
				timer.Stop()
				return ctx.Err()
			case <-timer.C:
				continue
			}
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return fmt.Errorf("HTTP %d: %s", res.StatusCode, body)
		}
		fmt.Println(string(body))
		return nil
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

The 30-second request budget and 4-attempt limit are example settings, not measured service guarantees. Rate limits back off, a supplied Retry-After controls the delay, and every retry reuses the same idempotency key. The program prints the response for inspection; it does not publish a hiring decision. Validate the returned citation and rubric interpretation before using any score.

This deliberately small fixture has an explicit limitation: it models one required permission group per chunk. If the application supports several groups, individual grants, or revoked permissions, extend the authorization model and its fixtures before relying on it. The local check does not prove that a remote vector query applied its restriction correctly.

That distinction matters under pressure. A passing unit fixture isn't permission to skip adapter verification. It is permission to test the adapter, not permission to declare the retrieval path safe.

The safe sequence remains scoped retrieval, reranking, and cited generation. Preserve the source-document identity through all three steps so that the final score can be checked against the passages actually admitted. If there is no authorized evidence for a rubric criterion, return an explicit evidence gap rather than filling it with another customer's material.

## 4. Compare replacement boundaries, not vendor logos

Provider portability has two separate surfaces here: model calls and the retrieval store's representation of tenant scope. Replacing either one requires checking the authorization invariant. A familiar request shape can reduce integration changes, but it does not establish equivalent retrieval or scoring behavior.

| Option | Relevant boundary to evaluate | Acceptance condition for this workflow |
| --- | --- | --- |
| OpenAI | Embeddings and answer-generation integration | Keep customer scope in the application and retrieval path; verify cited scoring after a model change. |
| Anthropic Claude | Answer-generation integration with its own provider contract | Choose it when an explicit generation adapter is acceptable; keep evidence retrieval outside that adapter and recheck rubric scores. |
| Google Gemini | An alternative model integration to evaluate independently | Choose it when the team can validate a separate model contract; do not infer compatibility from an equivalent-looking answer. |
| Pinecone | Namespace and metadata-filter representation in retrieval | Demonstrate the customer restriction and document-permission restriction together on the overlapping-ID fixture. |
| Weaviate | Tenant selection and filtered retrieval representation | Demonstrate that tenant selection and document access rules survive the adapter mapping. |

These are different roles in a system, so the table is not a claim that the products are interchangeable. OpenAI is relevant when embeddings and generation are being evaluated in the same integration; Claude is relevant to the generation boundary in this comparison; Gemini is a separate model choice whose scoring behavior needs its own checks. This does not claim identical feature sets. None of those model choices replaces the authorization work in a retrieval store, and a tenant-aware store does not decide whether a generated score follows a rubric. Consult the linked product documentation for each integration's concrete contract; this guide does not assert identical filter operators across them.

Infrai fits when a team wants one REST API and one key across backend capabilities, with an OpenAI-compatible model interface that lets an existing client keep its contract while model-field routing changes the selected vendor; its per-call vendor and request metadata also supports attribution during a scoring investigation.

A model adapter offers a concrete portability benefit: the application's capability contract can stay in place while the selected vendor changes behind it. Capture the selected provider and request correlation identifier in the adapter's operational record so an investigation can identify which configuration produced a score. This is an instrumentation requirement for the application, not an assertion that every provider returns identical metadata.

Keep the promise narrow. It does not imply interchangeable embedding spaces, identical ranking, or equivalent rubric scores, and it does not move tenant authorization out of the application. Switching embedding providers should therefore trigger compatibility verification before existing document vectors and new query vectors are used together.

The trade-off is operational ownership. An adapter reduces changes in the handler, but somebody still has to prove that each replacement preserves the security boundary and produces usable evidence for the scoring task. Direct integrations expose more provider-specific detail; an abstraction gives the application a smaller contract to maintain. Choose the boundary the team can actually verify.

## 5. Verify containment, then rehearse rollback

Before allowing a provider change into the scoring path, run the Cedar and Harbor fixtures through the real retrieval adapter. Check the passages delivered to reranking, then the passages delivered to generation, then the citations attached to the score. The test should establish that cross-customer and permission-denied records never enter either downstream input.

Test an empty scope too.

Add an authorized document with no relevant rubric evidence, and require an evidence-gap result. Repeat the checks after changing the model adapter, retrieval adapter, or permission mapping. These are release criteria, not claims about measured accuracy or production reliability.

Rollback should restore the previous verified adapter configuration and its compatible retrieval configuration together. If a replacement changes how vectors are produced, do not treat changing the answer model back as a complete recovery plan; verify the query and document representations before resuming retrieval. Stop candidate-score publication while the boundary is unverified.

Keep the rollback trigger concrete: unauthorized evidence, an untraceable citation, or a failed scope check blocks the rollout. A plausible score does not override it. After restoration, rerun the same fixture through the same downstream boundaries, and release only when the authorized evidence path is demonstrated again.

The useful postmortem artifact is the failed invariant and the check that now catches it. A screenshot of a green dashboard is less persuasive.

## References

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI text generation guide](https://platform.openai.com/docs/guides/text)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/overview)
- [Google Gemini documentation](https://ai.google.dev/gemini-api/docs)
- [Pinecone multitenancy guide](https://docs.pinecone.io/guides/index-data/implement-multitenancy)
- [Pinecone metadata filtering guide](https://docs.pinecone.io/guides/search/filter-by-metadata)
- [Weaviate multitenancy guide](https://docs.weaviate.io/weaviate/manage-collections/multi-tenancy)
- [Weaviate search filters](https://docs.weaviate.io/weaviate/search/filters)
