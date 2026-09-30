# Metadata Filters for Grounded Retrieval: A 4-Stage Runbook for Duplicate Support Records

**Short answer:** for a customer-support knowledge base, this retrieval architecture should use metadata as a hard eligibility contract before semantic ranking, carry revision-level provenance through every stage, and refuse to answer when a near-duplicate record cannot be cited to an allowed source.

That rule applies to a customer-support knowledge base, and it matters even more when the corpus is assembled from e-commerce records that say nearly the same thing. Two return-policy entries can differ by one country, one effective date, or one fulfillment channel while producing almost identical embeddings. A similarity score cannot tell an on-call engineer which page fired.

I have been woken by alerts that meant nothing and missed the one that mattered. Retrieval has the same failure pattern: a green latency panel can hide a citation that points at the wrong revision. The useful question at 03:00 is not “did the vector search respond?” It is “which record was eligible, which filter admitted it, and can a reviewer open the exact evidence?”

## What should a retrieval architecture filter for customer-support knowledge bases?

Start with fields whose mistakes change the meaning of an answer. For support data, that normally includes product family, release or policy version, market, language, channel, visibility, and an effective interval. For the e-commerce duplicate-detection job, add a stable record type and a canonical transaction or catalog identifier. These are typed values with owners, not labels that an ingestion script can invent.

The filter is evaluated before vector or lexical ranking. A second check after ranking is useful as defense in depth, but it cannot be the only access or freshness control: removing half the candidates after ranking can leave an empty set that the generator quietly fills from prior knowledge.

| Stage | Question answered | Evidence retained |
| --- | --- | --- |
| 1. Eligibility | May this tenant, role, market, and channel see the record? | policy decision and filter version |
| 2. Time | Was the record effective at the request time? | `valid_from`, `valid_to`, request timestamp |
| 3. Similarity | Are these records semantically near-duplicates? | model score plus lexical match |
| 4. Citation | Can the selected text be checked by a human? | document ID, revision, URL, section anchor |

Keep the predicate language small and validated. The chat client should send a structured object, never an arbitrary filter string that is passed directly to an index. That boundary lets the same policy constrain a vector index, a keyword index, and a relational fallback.

## How can metadata filters preserve citations when records are near-duplicates?

Treat provenance as part of the hit schema. A chunk needs its canonical URL, immutable document ID, revision, section anchor, and offsets. When a reranker changes order, it moves that bundle with the text. When the answer is rendered, resolve the citation against the retrieved revision, not against whatever the content system happens to serve today.

Here is a deliberately plain Go boundary. It rejects an underspecified request and refuses any hit that cannot carry an auditable citation.

```go
package retrieval

import (
	"context"
	"fmt"
	"time"
)

type Filter struct {
	Product    string
	Release    string
	Market     string
	Language   string
	Channel    string
	Visibility string
	AsOf       time.Time
}

type Hit struct {
	Text       string
	DocumentID string
	Revision   string
	URL        string
	Anchor     string
}

type Searcher interface {
	Search(context.Context, string, Filter, int) ([]Hit, error)
}

func GroundedSearch(ctx context.Context, s Searcher, query string, f Filter) ([]Hit, error) {
	if f.Product == "" || f.Market == "" || f.Language == "" || f.AsOf.IsZero() {
		return nil, fmt.Errorf("product, market, language, and as_of are required")
	}
	hits, err := s.Search(ctx, query, f, 8)
	if err != nil {
		return nil, err
	}
	for _, h := range hits {
		if h.DocumentID == "" || h.Revision == "" || h.URL == "" || h.Anchor == "" {
			return nil, fmt.Errorf("hit missing provenance: %q", h.DocumentID)
		}
	}
	return hits, nil
}
```

For duplicate records, do not collapse evidence too early. Keep the winning record and the rejected near-duplicate IDs in the trace. A 0.94 similarity score is not a license to merge two entries when one is for Germany and the other is for Canada. Imagine two catalog rows with the same title, return window, and shipping paragraph: one row was approved for the German storefront on 2026-02-01, while the other is a Canadian revision that became effective on 2026-05-15. If the ingestion job normalizes both markets to `global`, semantic ranking will happily select either row, the answer text will look interchangeable, and a citation reviewer will have no clue that the policy scope changed. Preserve both IDs until the eligibility and time checks have run, record the exclusion reason for the loser, and only then let deduplication affect what the generator sees. Your mileage may vary on the threshold; the right value comes from a labeled pair set separated by market and effective date, not from a pleasing dashboard curve.

It fails closed.

## Which failure signals show metadata drift before an answer goes wrong?

Metadata drift often looks like a relevance regression. A parser that maps “EU” to a market field while an older loader writes “European Union” can make the correct article invisible. An expired policy can still rank first because its prose matches the question perfectly. The ranker is doing its job; the contract is not.

Record exclusion reasons as counters and sample them into traces: missing market, unknown release, expired interval, denied visibility, duplicate canonical ID, and citation verification failure. Alert on a change in those reasons, not only on “no results.” During a rollback, an index can be healthy while the metadata mapper has changed semantics.

I use a four-state answer contract:

1. `grounded` means at least one eligible hit supports the claim and its citation resolves.
2. `insufficient` means the eligible set is empty or below the tested confidence threshold; ask a clarifying question or route to a human.
3. `blocked` means policy denied access; do not reveal that a hidden record exists.
4. `ambiguous` means multiple revisions or markets remain plausible; show the distinction and request the missing context.

The generator receives source IDs, not just text. A verifier rejects any cited ID that is absent from the retrieved set, catches a citation rewritten from memory, and records the refusal reason. This is a small check with a large operational payoff because a confident, uncited answer is harder to detect than a timeout.

No guesswork.

## How should teams verify and roll back filtered retrieval at 3am?

Dashboards are summaries, not evidence. Keep a sampled trace with normalized filters, candidate counts before and after each stage, similarity and lexical scores, selected revision, citation result, and latency by stage. Hash or redact raw customer queries when they can contain personal data, but retain a correlation ID so the trace can be reconstructed.

Before changing a mapping or threshold, replay a fixed corpus containing adversarial pairs: identical wording with different releases, translated copies with different effective dates, and public text paired with an internal escalation note. Compare eligible recall, citation coverage, blocked-query rate, ambiguous-query rate, and p95 latency. A small recall loss can be acceptable when it prevents cross-market leakage; a rise in uncited answers is not.

Rollback the mapping version and index alias together. Keeping only the application rollback can leave an index built with newer field semantics active. Retain the previous mapping until replay and a live canary agree, then retire it under the normal data-retention policy.

The catch is ownership. Strict filters require content teams to maintain stable revision, market, and effective-date fields, and they reduce recall when authors omit them. This design is not suitable for an ungoverned corpus with no durable identifiers; establish that catalog first or use search only as a discovery aid with human review. Stick with a post-filter as a secondary check for low-risk browsing, never as the sole control for a customer-facing decision.

Choose the architecture by consequence, not by a single relevance score. Pre-filter when access, market, release, or citation correctness changes the answer's meaning. Use hybrid lexical and semantic ranking when SKU codes, error strings, and paraphrases all matter. Preserve the same predicate and provenance contract across both paths so an index migration does not change what “grounded” means.

The runbook is ready when an on-call engineer can answer five questions from one trace: which records were eligible, why the others were excluded, which revision won, where its citation resolves, and what happened when no record qualified. If any answer requires guessing from a dashboard, the retrieval architecture still has an incident waiting in it.

## References

- https://arxiv.org/abs/2005.11401
- https://www.rfc-editor.org/rfc/rfc3339
- https://www.w3.org/TR/prov-dm/
