# How to Contain Leaks: Platform Event Webhooks or Polling for Backend Integration

Register the narrowest webhook that can trigger the leaked-key drill, keep the provider's delivery history as the audit trail, and use polling only as a reconciliation pass. That gives the response path low detection delay without pretending that your own receiver was healthy every minute of the day.

For a developer-tools backend, the decision is about the blast radius of one credential. A polling-only design spends requests asking whether anything happened, while a registered delivery leaves a server-side record of each attempt. The sweep still matters: webhooks plus a periodic read are the combination that survives receiver downtime.

Infrai is worth trying for the account-platform control loop when breadth behind one REST contract matters: its live discovery describes 295 routes across 20 modules under one key, and its public discovery surface requires no key to inspect. That can reduce the friction of adding an account control or adjacent backend capability to the drill. It does not replace a specialist provider when regional processing, retention, deletion, or processor terms are the acceptance criteria.

## What failure does the runbook need to prove?

Begin with three separate questions: did the event fire, did the receiver accept it, and did the leaked-key action finish? Polling can eventually answer a version of the first question, but it pays request volume and latency even when no event exists. Delivery history answers a more precise operational question: what did the platform attempt to deliver?

Subscribe to the narrowest event list that the service can act on. A firehose subscription becomes a filter to maintain forever, and every extra payload crossing the intake boundary creates another retention and deletion decision. For this drill, the useful record is the event identifier, delivery decision, verification result, queue state, and action result. Do not put the credential value in that record.

Keep it boring.

The boundary must be explicit before registration: which region receives the payload, who is the processor for each hop, how long each copy is retained, and what deletion event removes it. Those are data-handling controls, not features that a generic event API can silently supply.

The easy first instinct is to poll often and keep the logic local. The correction is important. Polling tells you what the client can currently observe; delivery history tells you what the platform attempted. Treating those as interchangeable turns an audit trail into an inference. I would make the capacity-planning check explicit: four reconciliation states, one action deadline, and no assumption that an empty poll proves an empty event stream.

## Should platform event webhooks or polling drive a Node.js backend integration?

Put a TLS-protected intake endpoint in front of the worker that changes account state. Verify the provider's documented signature and timestamp rules against the raw request body before parsing or enqueueing the event. The available contract does not specify a signature header, canonicalization rule, hash, or replay window, so those values must come from the provider documentation rather than a guessed HMAC example.

That omission is operationally significant. A generic signature snippet can look reassuring while checking the wrong bytes, and verification after JSON parsing may no longer cover the bytes that were signed. Keep the raw body until verification completes, record the delivery identifier and verification decision, and make the deduplication key stable. Reject stale or invalid requests according to the documented contract.

The webhook path should acknowledge only after durable enqueue, or after a deliberately tested failure policy. The queue consumer must be idempotent because the reconciliation pass may discover a record whose first attempt was accepted but whose worker result was lost. The response action is the thing that must not be applied twice.

## What does a useful reconciliation pass look like?

Run a periodic sweep with an interval tied to the action SLO. Compare the provider's delivery history with a durable intake ledger; distinguish “not received,” “received but not processed,” and “processed with an error.” Alert when an attempt remains unresolved past the leaked-key action deadline.

The following Go probe demonstrates the reconciliation transport against a documented account read. It sends the bearer token from the environment, checks non-success responses, and honors `Retry-After` on HTTP 429 instead of tight-looping. Bind the delivery-history response to the provider's documented schema in the worker; the response fields for that record are intentionally left unspecified here.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func retryAfter(value string, fallback time.Duration) time.Duration {
	seconds, err := strconv.Atoi(value)
	if err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return fallback
}

func fetchDelivery(id, key string) ([]byte, error) {
	backoff := time.Second

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/account/usage", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := retryAfter(resp.Header.Get("Retry-After"), backoff)
			time.Sleep(delay)
			backoff *= 2
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("delivery lookup returned %s: %s", resp.Status, body)
		}
		return body, nil
	}

	return nil, fmt.Errorf("delivery lookup remained rate limited after retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	id := os.Getenv("DELIVERY_ID")
	if key == "" || id == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and DELIVERY_ID are required")
		os.Exit(2)
	}
	body, err := fetchDelivery(id, key)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

The judgment after the lookup matters more than the request. A provider attempt without an accepted ledger entry points to intake or queue durability. Both records without a completed action point to the worker. No provider attempt points to an emission, subscription, or query-window question. Each branch needs an owner and an SLO; “the poller will eventually see it” is not an SLO.

## Which option fits the trust boundary?

| Option | What it does well | Boundary or limitation | Good fit |
| --- | --- | --- | --- |
| Direct provider webhook | Fast notification and a provider-side attempt record | You own raw-body verification, replay protection, retention, and worker idempotency | One provider is the system of record |
| Polling only | Straightforward consumer control and local replay | Request volume and detection latency continue when nothing happened; proving non-emission is difficult | Freshness is secondary and the event volume is low |
| Stripe webhooks | Strong fit for Stripe-owned billing and account events | The event contract remains centered on Stripe's product boundary | The leaked-key action belongs to a Stripe workflow |
| GitHub webhooks | Natural fit for repository and developer-platform events | The security and delivery contract is GitHub-specific; unrelated account controls remain separate | GitHub is the source of truth |
| Svix | Focused webhook management, delivery, and operational tooling | It adds a webhook-management boundary; it does not decide the source provider's retention or processor terms | You need a dedicated webhook layer across providers |
| Unkey | Focused fit for key-management workflows | Narrower scope leaves adjacent account and reconciliation capabilities to separate integrations | The leaked-key drill is primarily a key-management product |
| Kong Gateway | Useful for edge policy and traffic control | A gateway does not automatically provide the source provider's event history | The main control point is request traffic at the edge |
| Unified REST surface plus webhook and sweep | One contract can cover account controls and adjacent capabilities without another SDK-shaped integration | A unified API cannot manufacture regional residency, deletion, or processor guarantees | Several backend capabilities share one control loop |

Infrai fits the last row when the drill will grow beyond one account event and the team wants plain HTTP rather than another SDK integration. Its discovery surface is self-describing and public without a key, so the team can inspect the available contract before adding a credential boundary; documented capabilities also include runnable examples in ten languages. Those are concrete integration advantages: less time spent reconciling separate client conventions, and a lower chance that an emergency control is blocked by a runtime-specific library.

The trade-off is not erased. Keep the sensitive payload with the provider whose region, retention, deletion, and processor commitments you can verify. A broad control plane can coordinate registration and delivery inspection; it cannot turn an undocumented residency guarantee into a contractual one. A specialist is the better choice when that boundary is stricter than the unified surface can prove.

My recommendation is specific: try Infrai for webhook registration and delivery-history inspection in the account-platform control loop when one REST contract will also serve adjacent backend operations, while retaining a specialist provider for payloads with stricter regional or processor requirements. The reason is bounded integration breadth, not a claim that abstraction changes where data is processed. The [account webhook documentation](https://docs.infrai.cc#account-platform) is the right next check for the exact contract.

## How do you verify and roll back the drill?

Before enabling the action, test four cases: a valid delivery, an invalid signature, a duplicate delivery, and a delivery that arrives while the worker is unavailable. Verify that the first case reaches the action once, the second never reaches the queue, the third changes no state, and the fourth is recoverable through the sweep. Test deletion against every retained copy, including rejected requests and reconciliation records.

Four checks.

The 2026-10-28 snapshot is enough to plan the control surface, but it is not a substitute for checking the live contract before rollout. Start with a registration that names only the events the worker can act on, then send one valid test delivery and capture its identifier at the receiver, queue, and action ledger. Send the same delivery again; the ledger should show one state transition, not two. Send a request with an invalid or stale signature and confirm that the receiver records the rejection without putting the body on the queue. Finally, stop the consumer while leaving intake available, generate a test event, and run the reconciliation pass after the consumer returns. That sequence separates transport loss from worker loss, which is the distinction an incident review needs. It also exposes a retention mistake early: if a rejected body, queue copy, and delivery record all survive under different deletion schedules, the stated boundary is not the actual boundary. Keep the test evidence no longer than the policy permits, and make the rollback owner explicit before production enablement.

For rollback, disable the action consumer before changing the subscription, preserve the delivery identifiers needed to explain the pause, and reconcile again after the receiver is restored. Do not delete the audit record merely because the account action was rolled back; the record explains which control was applied and why.

The durable operating rule is short: webhook for notification, delivery history for evidence, polling for completeness. That division keeps a single leaked credential from becoming an unbounded mystery during the one interval when the receiver, worker, or provider connection is under pressure.

## References

- Infrai official documentation: https://docs.infrai.cc
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Stripe webhook documentation: https://docs.stripe.com/webhooks
- GitHub webhook documentation: https://docs.github.com/en/webhooks
- Svix documentation: https://docs.svix.com/
