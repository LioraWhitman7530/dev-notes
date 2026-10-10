# Next.js Phone Verification: SMS OTP Backend Controls for Property Access

Short answer: keep the resend deadline, attempt budget, and recipient suppression state on the backend; let Next.js display that state, never invent it. For a property-management login, the least complex defensible design is one verification record per challenge, an opaque browser token, a server-calculated `resend_at`, and an append-only decision log. A countdown improves the interface. It is not the control.

The page fires at 02:13: `otp_delivery_failure_rate` has exhausted its error budget for resident access logins. On-call sees a rising rejected-send count, one hashed recipient repeated across several property IDs, and no corresponding rise in successful verifications. The immediate action is to suppress that destination, stop further sends, and preserve the decisions that explain why; repeatedly sending because a browser timer reached zero merely turns a bad address or unreachable number into a larger compliance and operational problem.

## Which evidence should precede an SMS OTP alert?

The earlier signal is a recipient-level state transition, not a global delivery-rate alarm. A permanent rejection from the delivery channel should move a normalized, keyed recipient identifier into a suppressed state before another login flow can enqueue work. A transient outcome should remain eligible for bounded retry, but it still consumes the challenge's attempt budget. Those two cases cannot share a generic `failed` bucket because the next action differs.

This distinction matters when a resident enters a phone number for an SMS code and the property team also uses email for notices. An email bounce says nothing about whether the phone number is valid; an SMS rejection says nothing about the mailbox. Suppression therefore belongs to the tuple `(channel, recipient_key, purpose)`, with a policy-defined scope, rather than to a resident profile boolean. Broad suppression is tempting. It also creates hard-to-explain lockouts.

A useful page includes the SLO symptom, not message contents: challenge starts, accepted sends, permanent rejections, transient outcomes, verification successes, suppressions, and resends denied by reason. Keep property, region, and channel as bounded dimensions. Do not put raw phone numbers, email addresses, or challenge IDs into metric labels. The detailed audit event can carry a keyed digest and correlation ID under the retention and access policy; the metric cannot.

## How can Next.js keep phone verification login state authoritative?

The Next.js button should render `resend_at` from the server and recompute its label against the current clock. Refreshing the page then changes nothing important. Two tabs racing to resend also change nothing, because the backend accepts or rejects the transition atomically.

The browser waits. The server decides.

The API response needs only the state the interface can act on: an opaque challenge token, the next permitted time, and the remaining attempt count. The code that enforces the decision belongs behind that interface. This Go example shows the core transition; a Next.js route can call it without becoming the source of truth.

```go
package otp

import (
    "errors"
    "time"
)

var (
    ErrSuppressed = errors.New("recipient suppressed")
    ErrTooSoon    = errors.New("resend not yet allowed")
    ErrExhausted  = errors.New("attempt budget exhausted")
)

type Challenge struct {
    ID           string
    RecipientKey string // keyed digest, not the raw destination
    Channel      string
    Purpose      string
    ResendAt     time.Time
    AttemptsLeft int
    Version      int64
}

type Decision struct {
    ChallengeID string
    Kind        string
    At          time.Time
    Version     int64
}

type Store interface {
    IsSuppressed(recipientKey, channel, purpose string) (bool, error)
    CommitResend(current Challenge, next Challenge, event Decision) error
}

func AuthorizeResend(store Store, c Challenge, now time.Time, delay time.Duration) (Challenge, error) {
    blocked, err := store.IsSuppressed(c.RecipientKey, c.Channel, c.Purpose)
    if err != nil {
        return c, err
    }
    if blocked {
        return c, ErrSuppressed
    }
    if now.Before(c.ResendAt) {
        return c, ErrTooSoon
    }
    if c.AttemptsLeft <= 0 {
        return c, ErrExhausted
    }

    next := c
    next.AttemptsLeft--
    next.ResendAt = now.Add(delay)
    next.Version++
    event := Decision{ChallengeID: c.ID, Kind: "resend_authorized", At: now, Version: next.Version}
    if err := store.CommitResend(c, next, event); err != nil {
        return c, err
    }
    return next, nil
}
```

`CommitResend` must compare the stored version and write the new state plus decision event in one transaction. A conflict means another request won; reload and return its `resend_at`. Do not enqueue two messages and attempt to reconcile them later. The delivery worker should use the committed decision ID as its idempotency key, while the verification secret is stored separately as a one-way verifier with an expiry.

The delay itself is policy, not a constant hidden in React. Set it with threat modeling, delivery latency, support burden, and the capacity of the downstream channel in view. A longer delay lowers retry pressure but makes delayed messages painful; a shorter one increases duplicate attempts and can amplify abuse. There is no universal number in the available evidence, so the policy owner should record the chosen value and the observations that would trigger review.

## From rejection to suppression

The change that prevents the 02:13 page is straightforward: translate delivery outcomes into typed events, apply suppression before accepting another resend, and join the full trace by an internal correlation ID. Record timestamps at acceptance, dispatch, provider acknowledgement, terminal delivery outcome, suppression, and verification. Preserve the policy version used for each decision.

Email evidence deserves special skepticism. DKIM defines domain-level message signing and verification; it does not prove that a human opened a message or that an address remains suitable for authentication. Apple Mail Privacy Protection can prevent senders from learning Mail activity and can obscure IP information, so an open pixel is weak evidence for recipient validity. For an SMS login, neither an email signature nor an email-open event proves possession of the phone. Keep these evidence classes separate.

No click is proof.

A concise audit record can answer: which policy version evaluated the request, whether suppression was checked, which attempt number was authorized, when it became eligible, what terminal outcome arrived, and whether verification succeeded. Limit access and retention according to the system's policy. The operational question is reconstructability, not maximal collection.

Capacity planning follows from state transitions. Forecast peak challenge starts, multiply by the permitted attempts under policy, and reserve worker and datastore headroom for the resulting authorization rate. Then test the uglier shape: a burst of delayed outcomes followed by residents pressing resend together. The queue must absorb that burst without allowing expired challenges to consume useful delivery capacity.

## Where should the evidence boundary sit?

The delivery transport and the decision ledger have different lock-in profiles. Treat them separately.

| Layer | Buy favors | Build favors | Evidence question |
|---|---|---|---|
| Delivery transport | Carrier reach and less telecom operations | Direct control of routing and contracts | Can terminal outcomes be exported with stable identifiers? |
| Challenge state | Faster adoption of established controls | Policy portability and local transaction boundaries | Can every deny and authorize decision be reconstructed? |
| Suppression store | Managed ingestion of channel outcomes | Cross-transport rules under one schema | Can suppression reasons and policy versions be retained? |
| Audit archive | Managed retention and access controls | Existing governance tooling and query model | Can records be exported without losing ordering or fields? |

The decision is not a feature-count contest. If a managed component cannot export the evidence needed for an access dispute, its lower on-call load does not close the gap. If a self-hosted component adds an operator rotation, schema migrations, and carrier integrations that the team cannot sustain, nominal portability is equally unconvincing. I would require a replay test before either choice: given stored inputs and a policy version, can the team explain the outcome without consulting a mutable dashboard?

This design has limits. Backend-owned timing cannot make a delayed carrier route fast, prove that the current resident still controls a recycled number, or turn a delivery acknowledgement into proof that a person saw the code. A strict suppression rule is unsuitable where the only recovery path depends on the same destination, because one incorrect permanent classification can strand a legitimate resident; provide a separately governed recovery route instead, and require stronger review before clearing suppression. An append-only ledger also carries a retention trade-off: it improves reconstruction, but keeping recipient-linked evidence indefinitely increases exposure and eventually weakens the compliance argument it was meant to support. Minimize the event schema, separate the keyed recipient digest from operational fields, restrict who can resolve it, and delete or aggregate records when the documented purpose expires. Teams unable to operate atomic challenge state and audited recovery should choose a managed authentication component whose exports pass the replay test. Teams with unusual regional routing or established telecom operations may instead keep transport under direct control. Neither choice removes the need to test concurrent resends and terminal outcomes.

There is no free path here.

## Replay before enforcement

Roll out the state machine in shadow mode first, comparing its proposed decisions with the existing path while preventing it from sending. Then enable enforcement for a bounded property cohort and region. Watch verification success, permanent rejection, resend denial, support contacts, and time from challenge start to success; a single aggregate success rate can hide a damaging regional or channel split.

Test concurrency with two browser tabs, stale pages, repeated clicks, worker redelivery, delayed terminal outcomes, and an outcome arriving after a resident has corrected the destination. Test policy changes while challenges are active. A decision must use an explicit version, or the audit trail will explain today's rule rather than the rule that actually ran.

Fail closed on extra sends when the suppression store or atomic commit is unavailable, but keep the resident-facing response restrained so it does not reveal whether a destination is registered. The support path should use the correlation ID, never ask for the OTP, and expose the recorded decision rather than inviting an operator to bypass suppression without evidence.

Recovery is part of authentication.

## When suppression is wrong

An aggressive suppression threshold can block legitimate residents after a transient routing problem; a loose threshold can repeatedly contact an invalid destination, spend attempt capacity, and bury the useful signal under duplicates. Both errors reach on-call eventually. Only one is easy to count.

Set separate alerts for sudden suppression growth and for suppressed recipients later proving valid through an approved recovery path. Review samples by channel, region, and policy version, then change the threshold through the same controlled process as an authentication rule. The countdown remains a presentation detail throughout. The durable control is the backend decision, and the durable evidence is the ordered record showing why that decision was allowed or denied.

## Further reading

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Apple, Mail Privacy Protection guide: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
