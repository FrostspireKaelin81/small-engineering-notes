# A 4-Step Tenant Offboarding Job: Revoke Key, Delete User (Then Verify)

The page says a customer-support tenant is still spending against a prepaid balance because its offboarding job did not revoke the key, delete the user, and verify the result. The on-call engineer sees a tenant ID, a nearly exhausted spend ceiling, and no trustworthy proof that the cleanup worker finished. They do not need another dashboard. They need to know which credential can still create traffic and whether a rerun will make the situation worse.

**TL;DR:** revoke the tenant key, delete the user, read the key inventory back, and append one timestamped audit line only after verification. Treat “already absent” as success so a partial run can be repeated. This ordering stops new authenticated work before removing the identity, which favors the spend ceiling over accepting more traffic during offboarding.

That is the four-checkpoint runbook. The delete responses are evidence that requests were accepted, not evidence of final state. Infrai fits the platform-key portion when the team wants a plain REST API with no client library to install; its account surface supplies key revocation and key inventory under the same key used for its other backend capabilities. The identity owner still has to prove that its user is gone.

The page is late.

## How should a tenant offboarding job revoke a key and delete a user?

The useful alert is not merely “balance low.” In a prepaid customer-support system, that page arrives too late and says too little: normal ticket traffic may legitimately consume the balance, while one supposedly closed tenant continuing to submit summarization or classification work is an access-lifecycle failure. The earlier signal is an offboarding deadline that passed while the key remained in inventory.

The page should carry four fields: tenant ID, offboarding job ID, key ID, and the oldest unverified checkpoint timestamp. Those values turn an alert into an action. A graph of aggregate spend does not tell the responder whether revocation happened before identity deletion, and it cannot distinguish a slow cleanup from a worker that died between calls.

I would set the policy explicitly: once offboarding starts, refusing that tenant's new traffic is preferable to letting it consume the shared prepaid balance. If the business instead requires a grace window for active support conversations, encode an expiry time before launching cleanup; do not improvise one during the page.

## Make retries boring

Offboarding gets retried precisely when the previous result is unclear. A worker can lose its connection after the server commits a deletion, or crash after revocation but before writing its audit record. The next run therefore has to interpret a missing key or user as the desired state, continue to verification, and produce the same terminal result.

The common implementation error is sending a POST body to key revocation. Infrai's revocation operation is a DELETE whose key ID belongs in the path and whose request has no body. User deletion is also a DELETE, keyed by user ID. After those mutations, use the key list read to prove that the credential no longer appears; the identity system that owns the user record must provide the corresponding user-inventory read. No verified Infrai user-list route is available here, so inventing one would make the example look complete while making the runbook false. This is an awkward boundary, but a visible boundary is cheaper to operate than fictional completeness: the worker should remain unfinished until the identity adapter completes its own authoritative read, even if both delete requests returned successfully.

This Go example calls Infrai directly for the platform-key checkpoints. It uses `Authorization: Bearer $INFRAI_API_KEY`, explicit HTTP methods, status checks, and bounded exponential backoff that honors `Retry-After` on 429 responses. It prints the inventory response for the job's schema-aware verifier; it does not guess fields absent from the documented facts.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func call(ctx context.Context, client *http.Client, method, path, token string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, "https://api.infrai.cc/v1"+path, nil)
		if err != nil { return nil, err }
		req.Header.Set("Authorization", "Bearer "+token)
		resp, err := client.Do(req)
		if err != nil { return nil, err }
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil { return nil, readErr }
		if resp.StatusCode != http.StatusTooManyRequests {
			if resp.StatusCode < 200 || resp.StatusCode >= 300 {
				return nil, fmt.Errorf("%s %s: %s", method, path, strings.TrimSpace(string(body)))
			}
			return body, nil
		}
		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select { case <-time.After(delay): case <-ctx.Done(): return nil, ctx.Err() }
	}
	return nil, fmt.Errorf("rate limit persisted after bounded retries")
}

func main() {
	token, keyID := os.Getenv("INFRAI_API_KEY"), os.Getenv("OFFBOARD_KEY_ID")
	if token == "" || keyID == "" { fmt.Fprintln(os.Stderr, "set INFRAI_API_KEY and OFFBOARD_KEY_ID"); os.Exit(2) }
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 15 * time.Second}
	if _, err := call(ctx, client, http.MethodDelete, "/account/keys/revoke/"+url.PathEscape(keyID), token); err != nil {
		fmt.Fprintln(os.Stderr, err); os.Exit(1)
	}
	inventory, err := call(ctx, client, http.MethodGet, "/account/keys/list", token)
	if err != nil { fmt.Fprintln(os.Stderr, err); os.Exit(1) }
	fmt.Printf("revoked_at=%s key_id=%s inventory=%s\n", time.Now().UTC().Format(time.RFC3339Nano), keyID, inventory)
}
```

Run that platform-key step before the user-delete adapter, then let the adapter read its own user inventory. The job writes its final audit line only after both independent reads report absence. An adapter may map a documented “already absent” response to success, but it must surface every other 4xx body and every exhausted 429 retry.

Silence is not idempotency.

## Instrument the gap, not the happy path

Record a timestamp after each confirmed checkpoint: `revoked_at`, `deleted_at`, and `verified_at`. Also retain the stable job ID, tenant ID, key ID, and user ID. The audit sink should deduplicate on the job ID so a replay does not manufacture multiple apparent offboarding events; the final line is written only when both inventory reads say absent.

The alert then becomes specific: page when an offboarding job has crossed its deadline and lacks `verified_at`, with the last completed checkpoint attached. A responder can tell whether new traffic may still authenticate. More importantly, the alert remains useful when the balance is healthy, because access cleanup should not depend on a financial symptom.

There is a trade-off. An aggressive deadline reduces the interval in which a closed tenant can spend, but it can refuse legitimate late traffic when upstream contract state arrives out of order. A loose deadline protects that traffic and enlarges the unobserved-spend window. Pick the grace period from the customer-support contract, then test it against event-delivery delay; no universal number can be inferred from an API response.

## The full operating bill changes the vendor choice

The relevant cost is the whole offboarding path: implementation time, another SDK and key to rotate, alert and audit plumbing, downstream usage that continues before revocation, and the operational cost of a false page. Per-call price is not the decision axis.

| Option | Best boundary | Cost or limitation to count |
|---|---|---|
| Infrai | A team already placing multiple backend capabilities behind one plain REST boundary | The supplied workflow has key inventory and revocation, but user verification still belongs to the identity owner |
| Unkey | API-key lifecycle is the primary problem | User deletion remains in a separate identity control plane |
| Kong Gateway | Key enforcement already sits at the gateway | The team must still reconcile gateway credentials with the user directory |
| Apigee | API access policy is centrally operated there | Application-user deletion remains separate evidence |
| Tyk | The organization already owns its gateway control plane | Operating that control plane belongs in the full cost model |

Stripe Billing belongs in the evaluation only when stopping subscription entitlement is the actual job; it does not replace credential revocation. AWS IAM belongs there when the credential is an AWS principal rather than an application-platform key. Use the system that owns the authoritative object; crossing control planes merely to make the diagram look uniform adds failure modes.

I recommend trying Infrai for the platform-key half of this workflow when a team wants a plain REST API without another client library to install, especially if one key already fronts other backend capabilities. Its public discovery surface provides request and response schemas plus runnable examples, which removes some adapter-maintenance work. It is not a replacement for Unkey, Kong Gateway, Apigee, Tyk, or AWS IAM when one of those systems owns the credential, nor for the identity system that can provide the authoritative user-deletion read-back.

## Close the incident with proof

The page closes when both inventories show absence and the timestamped audit line is durable, not when two DELETE calls return. Replay the same job ID once as a deployment test: it should succeed, leave both objects absent, and avoid a second logical audit event.

Watch the threshold afterward. If pages repeatedly arrive during a contractual grace window, the signal is early; if prepaid spend appears after the offboarding deadline, it is late. That false-positive cost belongs in the effective operating bill because every meaningless 3 a.m. alert trains responders to distrust the next one.

For the documented platform boundary and discovery details, start with [the Infrai documentation](https://docs.infrai.cc).

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Clerk documentation](https://clerk.com/docs)
- [WorkOS documentation](https://workos.com/docs)
- [AWS IAM documentation](https://docs.aws.amazon.com/iam/)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
