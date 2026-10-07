# Changing User Email Addresses Safely: An API Transaction for Media Recovery

To change a user's email address safely, make the API create a pending identity transition rather than update a profile immediately: reauthenticate the user, record a single-use request, require proof from the new mailbox, notify the old mailbox, and preserve a separately governed recovery path. For a media service that scores login risk from device fingerprints, the deciding constraint is recovery: a familiar device is evidence, but it must never become proof that the person controlling a session also controls either mailbox.

TL;DR: keep the current email authoritative until the second step succeeds, bind both steps to one opaque, expiring request, and make completion atomic. A high-risk device score should increase friction or route the attempt to recovery; it should not silently approve or permanently deny the address change.

## How should an API safely change a user's email address?

A session can outlive the circumstance that made it trustworthy. A shared newsroom workstation, a stolen browser profile, or a cookie replay may present the same device attributes as yesterday, while the person at the keyboard is different. Device fingerprints are also probabilistic and can change after routine browser or hardware updates. Treat the score as an input to policy, never as an authentication factor or account ownership record.

Sessions drift.

The damaging failure is straightforward: an attacker with a live session replaces the login email, confirms only the new address, and then uses password reset to make the takeover durable. Updating the database first and sending email later creates a second failure mode. If delivery fails, the recovery identifier has changed even though nobody proved control of the destination.

OWASP's authentication guidance draws the useful boundary. For an email change without MFA, require the current password, verify both the current and proposed email addresses, and use two separate links. With MFA, require the current password and an MFA challenge, verify the proposed address, and notify the current address. In either case, authentication responses should avoid account-enumeration differences, and sensitive operations should require reauthentication.

The recovery policy therefore has to be explicit before the endpoint ships. If access to the old mailbox is mandatory in the no-MFA branch, say what happens when that mailbox is gone. Manual review, previously enrolled MFA, and recovery codes are distinct routes with distinct assurance; a device score can prioritize review, but letting it substitute for all of them collapses risk scoring into an undocumented master key. This two-step practice is intentionally inconvenient for a user who has lost the old mailbox and has no enrolled second factor, yet weakening the normal route for that case would give a session thief the same shortcut. Route the exception into a slower recovery process with a separate audit trail.

## Model the change as a transaction

Store a pending request separately from the user record. At minimum it needs an opaque request identifier, the user identifier, normalized old and new addresses, hashes of the two confirmation tokens, creation and expiry times, per-side confirmation state, an attempt counter, and one of three terminal states: `completed`, `cancelled`, or `expired`. Store token hashes rather than bearer tokens so a database read does not immediately grant confirmation.

Do not overwrite an existing pending request casually. Invalidate superseded tokens, rate-limit creation and verification, and make every token single-use. A new request should also reject an address already assigned under the service's normalization and uniqueness rules; keep the external error generic while recording the internal reason. Short-lived links reduce exposure, but the exact lifetime belongs to the service's threat model and mail-delivery SLO rather than to folklore.

Completion is the narrow critical section: lock the user and request rows, confirm that the request is still pending and belongs to the user, recheck uniqueness, update the authoritative email, mark the request completed, and enqueue notifications in the same database transaction through an outbox. The outbox matters because a successful commit followed by a crashed mail worker must be retryable without replaying the identity change.

```go
package emailchange

import (
	"context"
	"crypto/sha256"
	"crypto/subtle"
	"errors"
	"time"
)

var ErrInvalidConfirmation = errors.New("invalid or expired confirmation")

type Request struct {
	ID, UserID        string
	NewTokenHash      [32]byte
	NewEmail          string
	ExpiresAt         time.Time
	NewConfirmedAt    *time.Time
	CurrentConfirmedAt *time.Time
	RequiresCurrent   bool
	State             string
}

type Store interface {
	WithTransaction(ctx context.Context, fn func(context.Context) error) error
	LockRequest(ctx context.Context, requestID string) (Request, error)
	EmailAvailable(ctx context.Context, normalized string) (bool, error)
	SetEmail(ctx context.Context, userID, normalized string) error
	CompleteRequest(ctx context.Context, requestID string, at time.Time) error
	EnqueueSecurityNotice(ctx context.Context, userID, requestID string) error
}

func ConfirmNew(ctx context.Context, db Store, requestID, token string, now time.Time) error {
	presented := sha256.Sum256([]byte(token))
	return db.WithTransaction(ctx, func(tx context.Context) error {
		r, err := db.LockRequest(tx, requestID)
		if err != nil || r.State != "pending" || !now.Before(r.ExpiresAt) {
			return ErrInvalidConfirmation
		}
		if subtle.ConstantTimeCompare(presented[:], r.NewTokenHash[:]) != 1 {
			return ErrInvalidConfirmation
		}
		if r.RequiresCurrent && r.CurrentConfirmedAt == nil {
			return ErrInvalidConfirmation
		}
		available, err := db.EmailAvailable(tx, r.NewEmail)
		if err != nil || !available {
			return ErrInvalidConfirmation
		}
		if err := db.SetEmail(tx, r.UserID, r.NewEmail); err != nil {
			return err
		}
		if err := db.CompleteRequest(tx, r.ID, now); err != nil {
			return err
		}
		return db.EnqueueSecurityNotice(tx, r.UserID, r.ID)
	})
}
```

This focused function deliberately omits token issuance, password and MFA verification, HTTP parsing, and address normalization. Those belong at separate boundaries and need their own tests. It also returns one public confirmation error for mismatched, expired, and already-used requests, while internal telemetry can preserve the actual cause. A Node.js API should enforce the same transaction and constant-time token comparison semantics; the implementation language does not change the state-machine invariant, and mixing the HTTP handler with the transition logic makes race tests harder in any runtime.

After completion, decide session handling from the threat model. Rotating or revoking other sessions limits an attacker's persistence but interrupts legitimate readers and editors; retaining every session reduces disruption but leaves a stolen session alive. For a privileged media account, revoking other sessions and requiring a fresh login is usually the defensible policy. For a low-privilege consumer account, a service may choose a narrower response, provided that the recovery and notification controls do not weaken. The limitation is real: dual-mailbox confirmation is not suitable as the only recovery mechanism because the legitimate owner may no longer control the current address.

## Choose ownership by failure responsibility

The buy-or-build question is not decided by whether a provider exposes an email-update call. It is decided by who owns pending-state correctness, reauthentication policy, message delivery, recovery exceptions, audit evidence, and the pager when one of those components disagrees.

| Decision area | Managed identity boundary | Application-owned workflow |
|---|---|---|
| State and token lifecycle | Less custom state if the service natively models staged changes; behavior is constrained by its contract | Full control over dual confirmation and expiry; more security-critical code and migrations |
| Recovery policy | Consistent with the identity service's enrolled factors; exceptional cases may be inflexible | Can reflect newsroom and subscriber roles; manual paths expand abuse and on-call risk |
| Delivery and SLOs | Delivery telemetry and retries may sit across an operational boundary | Outbox, retry policy, suppression handling, and alerting stay under one team's control |
| Portability | Identity semantics and audit exports can create switching work | Standards-oriented interfaces reduce lock-in, but the team owns every invariant |
| Capacity planning | Quotas and service limits must cover bursts | Database locks, mail queues, and verification traffic must be load-tested directly |

This is not a generic preference for managed or self-hosted systems. Pick the boundary whose failure modes the team can observe and repair within its SLO. If the managed contract cannot represent the required old-mailbox branch, recovery route, or audit trail, wrapping it with application state may be necessary; if the team cannot staff token security and mail-delivery operations, owning the whole workflow is a liability. A managed boundary also has a downside when incident evidence is available only through delayed exports, while an application-owned design has the sharper downside: every missed race, token leak, and delivery retry belongs to the platform team's pager.

Capacity estimates should start with peak change requests, not monthly active users. Model retry amplification when mail is delayed, simultaneous confirmations racing on the same account, and notification backlog after a delivery outage.

The steady-state volume may be tiny.

The burst is what pages people.

## Verify the invariants before rollout

Tests should prove state transitions, not merely endpoint status codes. Exercise an expired token, a replayed token, a superseded request, confirmations arriving in either order, concurrent attempts to claim one address, a transaction rollback after the user row update is attempted, and an outbox worker delivering the same event twice. Property-based tests are useful for asserting that no event sequence changes the authoritative address unless every policy-required proof is present. Then run the same sequence through the public API, because a correct state machine can still be undermined by middleware that caches a response, loses the authenticated subject, or accepts a state change through the wrong method.

In staging, use controlled mailboxes and verify that links carry opaque secrets, redirects do not leak them, logs redact them, and scanners or preview clients cannot complete a change through a state-changing `GET`. The link can open a confirmation page; the mutation should require an intentional action protected against cross-site request forgery. Log request IDs and policy outcomes, not raw tokens or full addresses.

Operational signals need denominators. Track request-to-completion ratio, expiry ratio, verification failures by internal reason, delivery latency, outbox age, uniqueness conflicts, recovery escalations, and changes followed by password resets or session revocations. Segment by risk-policy band without storing a fingerprint as an identity claim. Alert on SLO symptoms such as sustained outbox delay or elevated transaction failures, rather than on raw change volume alone.

Ship behind a policy flag and begin with roles whose recovery rules are understood. The rollback is to stop accepting new requests while continuing to honor or explicitly cancel already-issued requests; rolling application code back while leaving live tokens and ambiguous pending rows is not a rollback. Keep the schema backward-compatible until the maximum token lifetime has passed, and rehearse cancellation notifications before production.

No shortcuts.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc6749#section-10.10
- https://www.rfc-editor.org/rfc/rfc9110.html#name-safe-methods
- https://owasp.org/www-community/attacks/csrf
