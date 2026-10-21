# Password Reset Email: Prefer Links, Keep Codes as a Measured Fallback

Prefer an application-owned reset link for a customer-support recovery flow when compliance evidence is the deciding constraint; add an email code only after segmented redemption data shows that browser handoff is the failure. The reason is operational, not aesthetic: a link lets the application issue, expire, and consume one opaque credential while the mail system remains transport, whereas a managed code service also owns challenge state, resend behavior, and attempt semantics that the evidence policy must reconcile.

Short answer: route the contact form to an identity-support queue, but never let the ticket carry the link, code, or message body. Keep a provider-independent recovery record, measure completed redemption rather than accepted mail, and choose the code path only for users who cannot reliably complete the link path.

## The incident begins in the support queue

Consider a bounded production drill. I would start with a contact-form submission categorized as `account_access`, route it using a pseudonymous account reference, and then have an authorized support actor initiate the normal recovery workflow. The mail transport accepts the message. The user still returns to support because recovery did not finish.

That single sequence exposes the weak claim behind many recovery dashboards: “sent” is neither “delivered” nor “redeemed.” If the ticket contains a pasted reset URL, the support system has accidentally become another bearer-credential store. If it contains only a provider receipt, an incident reviewer still cannot determine whether policy allowed the request, the credential expired, a second redemption was rejected, or the password actually changed.

The invariant is narrow: **support routing and credential redemption need correlation, not shared secrets.** Record a correlation identifier, pseudonymous account reference, actor class, timestamp, result, and reason code at each application-owned transition. Keep the token, code, password, raw address, and rendered message out of general logs and tickets.

No secret crosses that boundary.

This changes the investigation order. Start at the user-visible outcome and walk backward through consumed, dispatched, policy-approved, and requested states. Do not start from a mail acceptance receipt and infer success.

## Should a Password Reset Email Use a Link or Fallback Code?

The two approaches can meet the same user need, but their control boundaries differ. A link keeps redemption authority in the application. A managed email code delegates more of the challenge lifecycle, which can reduce implementation ownership while increasing the number of external semantics that must be mapped into the local audit record.

| Control question | Application-owned browser link | Managed email code |
|---|---|---|
| Who defines single use and expiry? | The recovery service and its store | The verification contract, reconciled locally |
| What proves completion? | An atomic local consume event | A verification result correlated to a local recovery request |
| What fails during resend? | A retry can accidentally mint overlapping links | Older and newer challenges can create attempt ambiguity |
| What must support see? | Status and correlation only | Status and correlation only |
| What makes migration hard? | Credential-store and policy operations | Event export, semantic mapping, and dual-run behavior |

My choice is the link because the support organization needs a stable account of the security-sensitive outcome even when mail transport changes. A transport timeout should retry the same message intent without creating a new credential. Redemption should atomically verify the credential, mark it consumed, and return the account reference, so two concurrent attempts cannot both succeed.

There is a real cost. The platform team owns secure credential storage, abuse controls, key handling, migrations, backup restoration, and the on-call path. Managed verification is the better buy when its event export, retention controls, regional handling, attempt rules, and incident access satisfy the evidence policy in writing, and when the team does not have the staffing to operate that boundary to an explicit SLO. This is not a price comparison; it is a decision about whose state is authoritative at 03:00 during an audit-sensitive incident. The trade-off is explicit: local state improves evidence continuity and transport portability, but every schema migration, saturation alarm, restore test, and expired-record purge remains the platform team's responsibility. External challenge state removes part of that operational surface, but the audit then depends on exported events preserving the meanings the local policy requires.

## Make illegal transitions boring

The preventative mechanism is a small state machine around an append-only evidence record. The example deliberately excludes bearer values and email addresses. Token generation and the durable compare-and-consume operation sit behind `Store`; message rendering and transport sit elsewhere.

```go
package recovery

import (
	"context"
	"errors"
	"time"
)

type State string

const (
	Requested  State = "requested"
	Dispatched State = "dispatched"
	Consumed   State = "consumed"
	Expired    State = "expired"
	Denied     State = "denied"
)

type Event struct {
	CorrelationID string
	AccountRef    string // Pseudonymous; never an email address or credential.
	From          State
	To            State
	ActorClass    string
	ReasonCode    string
	OccurredAt    time.Time
}

type Store interface {
	Current(context.Context, string) (State, error)
	Append(context.Context, Event) error
}

func Advance(ctx context.Context, store Store, event Event) error {
	current, err := store.Current(ctx, event.CorrelationID)
	if err != nil {
		return err
	}
	if current != event.From || !allowed(event.From, event.To) {
		return errors.New("invalid recovery transition")
	}
	return store.Append(ctx, event)
}

func allowed(from, to State) bool {
	switch from {
	case Requested:
		return to == Dispatched || to == Denied
	case Dispatched:
		return to == Consumed || to == Expired || to == Denied
	default:
		return false
	}
}
```

Boring is useful.

The store still needs an atomic consume operation; `Advance` only illustrates transition validation, not credential verification. Store a one-way representation rather than the bearer value, expire it, enforce single use, and return consistent outward responses so the recovery interface does not reveal whether an account exists. OWASP's forgot-password guidance calls for random tokens, secure storage, expiration, single use, side-channel delivery, consistent responses, and protection against excessive requests.

Template behavior deserves a deployment test because evidence quality is irrelevant if the message is malformed. Mustache escapes variables by default, while triple braces and ampersands produce unescaped output. A reset URL placed into HTML therefore needs a reviewed context and an exact rendering test, not a casual assumption that every placeholder is equivalent.

## Capacity planning starts with hostile traffic

Average contact-form volume is the wrong capacity input. Recovery is a public workflow, sends can be amplified toward one mailbox, and a support event can move legitimate demand quickly. Model independent limits per source, per pseudonymous account reference, and per recovery purpose; the exact thresholds must come from the local threat model and observed demand rather than a universal number.

I would operate two SLOs. One covers the application's policy decision and credential issuance. The other covers successful user redemption within the allowed lifetime, segmented by client class and by link versus code. Transport acceptance belongs on the dependency dashboard, but it is not the user-facing SLI.

Page for sustained inability to issue or consume credentials. A slow drift in transport acceptance or template-rendering failures can create a ticket before it creates an availability page. A rise in denied attempts needs security and support context, since the control may be doing its job.

The release gate should exercise an unknown account, expiry, a second redemption, simultaneous redemption, resend, transport timeout, template escaping, a scanner-style GET, explicit user confirmation, and queue routing. Then restore the ledger from backup and verify that its retention behavior still satisfies policy. Evidence that disappears during recovery of the recovery system is not defensible.

Test the ugly path.

## When does the code become the right fallback?

The limitation of a browser link is the handoff. It is the wrong default when measured failures cluster there: the person reads mail on one device but resets on another, an embedded browser loses necessary context, or security software visits the URL before the person does. Automated fetching must never complete a password change; the landing step should require explicit user action, and the server must still enforce expiry and single use.

For those segments, a code is a justified fallback. It also creates another verification surface: bound attempts, prevent resend races, define whether only the newest challenge is active, and correlate verification results without logging the code. Run its redemption SLI separately. Otherwise the aggregate can hide a failing link path behind successful code entries, or the reverse.

Do not maintain two permanent paths merely because both are familiar. Each path adds policy states, tests, alerts, runbooks, and evidence mappings. **Use link-first and code-by-evidence** when the application can operate the redemption store; choose managed verification instead when its documented controls meet the same evidence requirements and the operational ownership is the more credible boundary. That recommendation does not apply where a browser cannot be assumed, where cross-device completion is the established dominant journey, or where the team cannot sustain the store's SLO and audit obligations. In those conditions, a bounded code flow or a managed challenge boundary is the more honest choice.

The contact form remains deliberately uninteresting: category, pseudonymous reference, correlation identifier, and safe status. That is enough to reach the correct support queue without turning the support platform into part of the credential path.

## Sources

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- NIST, Digital Identity Guidelines, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- FTC, CAN-SPAM Act compliance guide for business: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
