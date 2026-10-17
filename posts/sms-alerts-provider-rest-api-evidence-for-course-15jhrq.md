# SMS Alerts Provider: REST API Evidence for Course Receipt Templates and Suppressions

A settled payment creates an awkward operational constraint: the learner should receive one timely SMS receipt, while the platform cannot infer from a timed-out REST call whether the downstream system accepted anything. Choose an SMS alerts provider only after its observable behavior passes that ambiguity test across the US and EU paths you intend to operate.

TL;DR: Treat provider selection as an acceptance protocol, not a feature comparison. Require a stable submission identity, a queryable outcome or documented reconciliation mechanism, authenticated and repeatable status events, enforceable suppressions, and reconstructable template versions. Keep the payment record authoritative. If a candidate cannot produce evidence after a lost response, it is not a reliable receipt path, however simple its API looks.

This is a narrow claim. SMS can alert a learner that a course order settled; it should not become the receipt system of record.

## Can one SMS provider safely mix shipping alerts and appointment reminders?

The incident exercise I put on the review table has a deliberately small boundary: one course order, one settlement event, one worker, and a connection that disappears after request bytes may have left the process. This is a design exercise, not a production anecdote. The worker does not know whether the request was rejected, accepted, or accepted while its response was lost. A second blind submission may duplicate the receipt; abandoning the intent may omit it.

That uncertainty is the selection test. Ask each candidate to demonstrate what an operator can establish using the stable intent identity after the client loses the response. An HTTP success response in the happy path proves very little about this case. The useful evidence is durable: an acceptance record tied to the platform's identity, an outcome that can be reconciled without sending again, and delivery events whose duplicates and reordering do not corrupt local state.

The invariant is blunt: **one settlement creates one notification intent**. Attempts may multiply. The intent must not.

Templates and suppressions belong in the same exercise because they can change between attempts. Record the selected template version and the policy decision with the intent; otherwise a retry can send different text, or send after a destination became ineligible. Appointment reminders and shipping alerts need separate obsolescence rules: a canceled appointment makes a queued reminder stale, while a newer shipment state may supersede an older update. Account-recovery codes are another workflow entirely. OWASP's forgot-password guidance calls for consistent responses, side-channel delivery, rate limiting, and tokens that are single-use and expire; a settled-order receipt must not inherit that security workflow merely because both use SMS.

For US and EU traffic, I would reject a single checkbox labeled `region` as evidence. Purpose classification, consent evidence where applicable, retention, deletion, and cross-border handling need review by the responsible legal and privacy owners. Engineering's job is to turn those decisions into versioned inputs and retained outcomes, not invent one universal rule in application code.

## Make the contract executable before procurement

A short REST example can conceal a large on-call obligation. Define the adapter contract first, then run the same failure script against every implementation: repeat one identity, lose the first response, deliver the same callback twice, deliver an older status after a newer one, suppress the destination while work is queued, and rotate credentials while requests are in flight. Pass or fail should come from stored evidence, not a demo operator's interpretation.

The following Go types force the ambiguous result into the contract. They also prevent business code from treating a transport error as proof that nothing happened downstream.

```go
package receiptcheck

import (
	"context"
	"errors"
)

var ErrOutcomeUnknown = errors.New("submission outcome unknown")

type Intent struct {
	ID              string
	OrderID         string
	SettlementID    string
	DestinationRef  string
	TemplateVersion string
	PolicyVersion   string
}

type Acceptance struct {
	IntentID string
	RemoteID string
}

type Transport interface {
	Submit(context.Context, Intent) (Acceptance, error)
	Lookup(context.Context, string) (Acceptance, bool, error)
}

type EvidenceStore interface {
	Eligible(context.Context, Intent) (bool, string, error)
	RecordSuppression(context.Context, Intent, string) error
	RecordAcceptance(context.Context, Acceptance) error
}

func EstablishOutcome(ctx context.Context, tx Transport, store EvidenceStore, in Intent) error {
	eligible, reason, err := store.Eligible(ctx, in)
	if err != nil {
		return err
	}
	if !eligible {
		return store.RecordSuppression(ctx, in, reason)
	}

	accepted, err := tx.Submit(ctx, in)
	if err == nil {
		return store.RecordAcceptance(ctx, accepted)
	}

	accepted, found, lookupErr := tx.Lookup(ctx, in.ID)
	if lookupErr != nil || !found {
		return errors.Join(ErrOutcomeUnknown, err, lookupErr)
	}
	return store.RecordAcceptance(ctx, accepted)
}
```

`Lookup` is a capability boundary, not a claim that every provider exposes a lookup endpoint. An implementation may reconcile through another documented mechanism. If none exists, return `ErrOutcomeUnknown` and send the intent to controlled reconciliation; do not convert uncertainty into an automatic retry. Likewise, the store needs a uniqueness constraint on `Intent.ID`. An interface alone cannot enforce uniqueness across workers.

Callback handling needs equally skeptical tests. Authenticate the callback using the documented scheme, retain the raw downstream status for diagnosis, map it into a small internal state model, and make repeated events harmless. An older event must not move a terminal local state backward. Keep phone numbers, order IDs, and remote message IDs out of metric labels; operators can follow those identifiers in access-controlled records without turning the metrics system into a high-cardinality customer index.

## Size the recovery path, not the demo path

Average request rate is the wrong capacity number. Suppose 120,000 course orders can settle during a two-hour enrollment window: the average is about 17 settlement events per second. A ten-minute downstream pause would accumulate roughly 10,000 events at that average, before retries or an end-of-window spike. These are scenario inputs for planning, not reported production measurements.

Now set a drain objective. Clearing that 10,000-event backlog in 15 minutes requires more than 11 successful submissions per second above live arrivals, before discounting for retries and rate limits. That arithmetic exposes whether contracted throughput, worker concurrency, and queue partitions support the freshness SLO. A maximum requests-per-second number without a recovery test does not.

Use separate indicators for `settlement-to-submission` and `submission-to-terminal-status`. The first covers the platform path. The second covers downstream evidence and may legitimately remain unknown for a time. Page on sustained queue age that threatens the receipt freshness objective, inability to accept new intents, or a broad rise in ambiguous outcomes. An isolated terminal delivery result generally belongs in investigation rather than an immediate page unless its rate threatens the SLO.

Short path, long recovery. Capacity review has to include both.

## Buy or build the evidence boundary

The decision is not managed versus self-hosted in the abstract. It is which team owns each missing proof at 03:00, and whether that ownership fits the platform roadmap.

| Decision boundary | Broader managed workflow | Thin adapter or self-hosted control | Acceptance evidence |
|---|---|---|---|
| Submission identity | May retain its own workflow key | Platform defines and stores the intent key | Repeating one key does not create a second intent |
| Templates | May host rendering and change history | Platform versions content and localization | Exact selected version is reconstructable |
| Suppressions | May enforce destination rules | Platform evaluates purpose-specific eligibility | Decision, reason, and policy version are retained |
| Delivery state | May normalize carrier-facing events | Platform authenticates and reduces raw events | Duplicate and out-of-order events are harmless |
| Regional controls | May offer routing and processing options | Platform owns classification and failover policy | Contract terms and tested controls match approved use |

A broader service can remove template operations and some event processing from the on-call surface, but it also couples the platform to that service's state model and export behavior. A thin adapter keeps business truth portable and makes replacement easier, while leaving queue operation, policy changes, callback handling, and reconciliation with the platform team. **Prefer the boundary whose failure evidence your team can operate**, then record the rejected burden as explicitly as the chosen one.

I initially score a shorter adapter as an operational advantage. I reverse that judgment when its ambiguity test cannot reconstruct whether an intent was accepted, because fewer integration lines then purchase a larger reconciliation burden.

Do not score a candidate on feature count until it passes the failure script. After that gate, compare the maintenance load, exportability, regional controls, support for versioned templates and suppressions, and the clarity of its error taxonomy. Price belongs in the ownership calculation, but it is not a substitute for a recoverable outcome.

The main limitation of this approach is the operational weight of durable evidence and reconciliation. That trade-off is not suitable when the message is best-effort and duplicates carry little consequence; a direct submission path with a coarse retry policy is a better fit there. It is also insufficient for authentication secrets, where the OWASP controls above create a stricter security boundary. For a settled course order, the durable receipt remains in the learner's account, while SMS provides a timely pointer to that truth. The winning design is the one that can explain every intent after the network stops explaining itself.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://senders.yahooinc.com/best-practices/
