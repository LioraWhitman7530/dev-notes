# Practical Limits of Free Error Tracking: Rules, Webhooks, Notifications, and Polling

Short answer: use threshold rules and webhooks for timely handoff, notifications for human attention, and API polling for reconciliation; a free error-tracking plan is operationally adequate only when its tested limits still leave enough time and capacity to protect the service SLO.

Error tracking is diagnostic evidence, not a complete availability model. An exception count can say that something failed, but without a denominator it cannot say what fraction of user work failed, and without a delivery objective it cannot say how quickly anyone must react. The practical design therefore starts with the action and its deadline, then assigns each mechanism one job. Don't make a chat message, a webhook receiver, or a scheduled poller carry the entire incident-response contract.

Keep those boundaries boring.

## How should free error tracking combine threshold rules, notifications, webhooks, and API polling?

A threshold rule evaluates a condition over a window. A notification puts the result in front of a person. A webhook transfers an event to another system. An API poll asks for state on the caller's schedule. They may sit beside one another in a configuration screen, but they have different failure modes, timing guarantees, owners, and capacity constraints.

Use an error-tracking rule to detect diagnostic changes such as a new fingerprint or a recurrence after a release. Use an SLO signal with a meaningful denominator to establish urgency. Ten exceptions could mean ten failed checkouts, ten successful retries, or one client stuck in a loop; grouping helps investigation, while a request- or journey-based SLI supplies the impact context needed for paging. If the tracker cannot evaluate that denominator, keep the page in the system that can and attach the error evidence to the incident.

Webhooks are the push path. They suit near-real-time automation because the sender initiates delivery, but the receiver still needs authentication, deduplication, bounded work, and a durable queue. Notifications are the presentation path. Their useful payload is small: service, environment, severity, time window, likely owner, and a stable link or identifier for the diagnostic record. Dumping an entire event into chat creates another loosely governed telemetry store and usually makes the message harder to act on.

Polling is the repair path — and sometimes the reporting path. A poller can reconcile unresolved items, recover state after a receiver deployment, or build a daily ownership report. It is a poor substitute for push when a short error-budget window governs response: detection latency includes the polling interval, request time, processing time, and any backlog, while frequent polling consumes API allowance even when nothing changed.

“Free” changes the capacity envelope, not the architecture. Before depending on any plan, record its current ingestion allowance, retention, rule count, outbound integration limits, API quota, and data-access boundaries, then test the credible release burst rather than the quiet-hour average. I'm not sure any published quota can answer the harder question — whether a particular workload's fingerprints and retries will amplify during its worst deployment — so a synthetic load test and observed queue behavior have to settle that locally.

## Separate diagnostic evidence from the paging decision

The threshold should name a population, window, action, and owner. “More than 20 errors” is incomplete. Twenty failures among 25 attempts and twenty among 25 million imply different impact, while a five-minute response objective and a next-business-day review require different routes. A useful rule definition connects the error evidence to a service objective and says what happens when the condition fires.

This is the buy-versus-build review I would put in front of a platform team:

| Approach | Best use | Capacity and on-call cost | Practical limitation |
| --- | --- | --- | --- |
| Managed error tracking | Grouping diagnostic events and routing issue signals | Quotas, retention, rule ownership, and integration checks | Issue counts may not represent affected requests |
| Self-hosted error pipeline | Controlling storage, routing, and retention | Upgrades, backups, storage growth, and a named rotation | Platform maintenance competes with product roadmap work |
| Metric-based SLO evaluation | Measuring rates, latency, and error-budget burn | Instrumentation quality and cardinality control | Diagnostic detail usually lives elsewhere |
| Scheduled API polling | Reconciliation, exports, and delayed summaries | Cursor state, API allowance, and credential rotation | Detection delay makes it unsuitable for urgent response |

No row wins universally. A managed path is not suitable when data residency, retention control, or a required delivery contract falls outside its documented boundary; self-hosting is not suitable when nobody owns upgrades, restores, capacity, and after-hours failures. Stick with polling for delayed inventories and reconciliation, but keep it out of the primary paging path when one missed interval can consume a material part of the error budget.

Capacity planning needs one ugly scenario: a release produces multiple fingerprints at once, retries multiply events, and every matching rule emits outbound work. Estimate event volume, webhook requests, queue depth, downstream notification rate, and replay volume together. Average events per minute conceal this shape. If the tested burst exhausts a free allowance or fixed queue before the response objective, reduce per-event fan-out, reserve more capacity, or move the authoritative page to an independently measured SLI.

Fast is not enough.

## Build a narrow and authenticated push handoff

The synchronous receiver should cap the body, authenticate the exact bytes, validate a small schema, enqueue idempotently, and acknowledge. Everything slower belongs behind the queue: enrichment, ownership lookup, correlation, ticket creation, and human notification. This keeps sender retries cheap and prevents a slow downstream destination from consuming all receiver capacity.

The following Go handler illustrates an application-owned contract, not a claim about a vendor's payload. The sender places a hexadecimal HMAC-SHA256 value in `X-Event-Signature`; both sides must agree on that contract. `EnqueueOnce` is intentionally one operation because separate “seen?” and “enqueue” calls create a race under concurrent delivery.

```go
package eventhook

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"net/http"
)

const maxBodyBytes = 1 << 20

type Event struct {
	ID          string `json:"id"`
	Service     string `json:"service"`
	Environment string `json:"environment"`
	Fingerprint string `json:"fingerprint"`
}

type Queue interface {
	// EnqueueOnce must atomically suppress a repeated event ID.
	EnqueueOnce(Event) error
}

func Handler(secret []byte, queue Queue) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}

		body, err := io.ReadAll(http.MaxBytesReader(w, r.Body, maxBodyBytes))
		if err != nil {
			http.Error(w, "invalid body", http.StatusBadRequest)
			return
		}

		supplied, err := hex.DecodeString(r.Header.Get("X-Event-Signature"))
		mac := hmac.New(sha256.New, secret)
		_, _ = mac.Write(body)
		if err != nil || !hmac.Equal(mac.Sum(nil), supplied) {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		var event Event
		if err := json.Unmarshal(body, &event); err != nil || event.ID == "" || event.Service == "" {
			http.Error(w, "invalid event", http.StatusBadRequest)
			return
		}

		if err := queue.EnqueueOnce(event); err != nil {
			http.Error(w, "queue unavailable", http.StatusServiceUnavailable)
			return
		}

		w.WriteHeader(http.StatusAccepted)
	}
}
```

Secret rotation deserves an explicit two-key overlap rather than an undocumented exception in the receiver. Size the overlap against the sender's documented retry horizon, remove the old key after the overlap, and observe verification outcomes by reason without logging the signature or raw body. A `401` after rotation is an authentication signal; repeating the request blindly won't make the contract safer. Likewise, treat `413` as evidence that the schema or size limit changed, not as permission to accept an unbounded body.

The capacity constraint is synchronous work per request. If authentication, decoding, and durable enqueue take 40 milliseconds at the slow end, multiplying an optimistic average throughput by queue depth will mislead; test concurrency, storage latency, and duplicate delivery together, then preserve headroom for release bursts. Your mileage may vary, especially when the durable queue shares storage with another workload.

## Verify the release path, then make rollback dull

Deploy first to an isolated environment and emit a synthetic event with known service, release, environment, and fingerprint fields. Capture the creation time, rule-evaluation time, receiver acceptance time, durable enqueue time, and final notification time. The path passes only when the intended owner receives one actionable notification within the declared objective and the audit record can connect every hop without retaining unnecessary event contents.

Test duplicates. Test an invalid signature, a body over the configured limit, slow queue storage, a notification destination that cannot keep up, and a receiver restart after acceptance. For the poller, persist the documented cursor or high-water mark only after durable processing, restart from the last committed checkpoint, and verify that the same records don't create repeated work. Also decide how the client handles an empty page, out-of-order records, credential rotation, and an API budget nearing its limit. A `403` should trigger credential and scope inspection; it should not become a tight retry loop that spends the remaining request budget.

CLI telemetry belongs in this review because operational tools often run during sensitive tests. The `DO_NOT_TRACK` convention defines a common environment variable that command-line tools can honor as an opt-out signal. It doesn't replace a data policy or tool-specific documentation, but respecting it makes test behavior less surprising. This is a narrow standard with a narrow job.

Roll back notification routing without deleting collected evidence, rule history, or the previously tested destination. The switch should be exercised before launch, the rollback owner should be named, and the runbook should state which SLO signal remains authoritative while the route is disabled. For a high-urgency path, preserve an independently tested paging route; for a delayed reconciliation job, pausing the poller while retaining its checkpoint may be the safer rollback.

The final go/no-go question is blunt: can the team prove delivery, duplication control, capacity, and reversal inside the error-budget window? If not, the integration may still be useful for diagnosis or reporting, but it isn't ready to serve as an on-call dependency.

## References

- https://consoledonottrack.com/
