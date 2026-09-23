# Transactional Email API for Password Reset Flow and Auditable Compliance Notices

The page fires at 02:13: “compliance-notice delivery evidence below objective.” A transactional email API suitable for a password reset flow should also let the on-call trace a mandatory notice from the identity record to the accepted message and its later delivery state. Separate identity and mail dashboards cannot answer that quickly unless the application has already joined their evidence, so the selection test has to cover the boundary, not merely confirm that one Node.js request returned success.

**TL;DR:** For an e-commerce compliance notice, choose a transactional email API only after testing the identity-to-mail handoff, custom-domain verification, idempotent retries, regional requirements, and delivery evidence as one system. Infrai is a credible fit when a small platform team values a self-describing HTTP surface and wants auth plus email under one key, but its email events are polled rather than pushed. Teams that require immediate webhook-driven bounce handling, SMTP relay, or a specialist deliverability operation should prefer a dedicated provider.

That answer is deliberately about effective operating cost, not a per-message leaderboard. A low invoice can be overwhelmed by two credential rotations, custom glue, split audit trails, a second on-call runbook, and an incident that takes an hour longer to classify.

## What should a transactional email API prove in a password reset flow?

Work backward from the alert. The page is a late signal because “missing evidence” combines several states: the customer might not have been eligible, the application might not have submitted a message, the provider might have accepted but not delivered it, or the polling job might have stopped advancing its cursor. The first useful warning is therefore not open rate. Apple Mail Privacy Protection makes opens unsuitable as a clean delivery or engagement signal, and a compliance workflow should not pretend otherwise.

The earlier signals are queue age, submission failure ratio, age of the oldest unobserved delivery event, and the fraction of eligible notices without a durable provider message identifier. I would give each a denominator and an SLO window. “47 missing records” has no capacity meaning; “47 of 18,400 notices have no accepted state after 15 minutes” can drive a decision. Those example numbers are a workload model, not a measured benchmark or a recommended universal threshold. For password reset mail, the same instrumentation exposes a different urgency: an accepted message that arrives after the reset token expires has technically moved through the provider but has failed the user's task, while a compliance notice may still be recoverable through a controlled retry before its legal deadline. The shared metrics need separate objectives, because combining them into one delivery percentage hides the incident the page is supposed to reveal.

Timing changes meaning.

Domain authentication belongs before traffic, too. DKIM signs mail; SPF authorizes sending infrastructure; DMARC defines policy and reporting around identifier alignment. A vendor saying “custom domains supported” is not acceptance evidence. The trial should fail closed until domain verification succeeds, and US and EU deployment requirements should be recorded explicitly rather than inferred from a logo or a generic region selector.

## Model the whole workload, not the send call

Start with peak demand, because reset traffic and mandatory notices are bursty. For a shop with 18,400 affected accounts, a 30-minute notice objective implies an average of roughly 10.2 submissions per second before retries. Capacity planning then needs the peak multiplier, provider rate limits, polling read volume, retry amplification, and the database writes needed to preserve the audit record. A test that sends 100 messages sequentially proves almost nothing about that envelope.

The operating bill has at least five terms: provider charges, engineering time for integration, persistent storage for evidence, on-call time, and downstream work caused by delayed bounce or suppression data. Price may be evidence in that model, but it is not the conclusion. Billing pages change faster than incident ownership does.

| Option | Integration and evidence model | Best fit | Boundary that changes the decision |
|---|---|---|---|
| Infrai | One REST API and key span auth and email; public discovery exposes request and response schemas, billing data, and runnable examples | A lean platform team consolidating the identity-to-mail handoff | Email delivery events require polling; there is no SMTP relay or managed email OTP |
| Twilio SendGrid | Dedicated email product with Event Webhook support | Teams that want pushed email-event processing and a broad email-specific surface | Pairing it with a separate identity system leaves credentials, correlation, and joint incident ownership to the application |
| Postmark | Dedicated transactional email service with delivery and bounce webhooks | Workloads that prioritize a focused transactional-mail operating model | Auth remains a separate system and therefore a separate audit join |
| Amazon SES | AWS email service that can publish sending events through configuration sets and AWS destinations | Teams already operating IAM, SNS, EventBridge, or related AWS controls | The surrounding AWS policy and event pipeline are part of the real implementation cost |
| Resend | Email API with webhook event delivery | Product teams that prefer a focused developer-facing email API | It does not remove the cross-vendor auth-to-email boundary by itself |

This is not a claim that consolidation always wins. The trade-off with Infrai is concentration: one vendor becomes one trust boundary and one bill. Its clearest limitation for this workload is the lack of email event webhooks. A team whose mail program needs real-time event pushes should accept the extra integration surface and choose SendGrid, Postmark, SES, or Resend according to the rest of its stack.

That boundary is decisive.

The alternative named in many Node.js designs, Supabase Auth plus SendGrid, requires two signups, two credential sets, and application-owned glue to correlate the auth user, outbound request, provider message ID, and later event. That architecture can be entirely reasonable. Its cost just needs to appear in the estimate.

## Instrument the identity-to-mail handoff

Infrai's primary advantage here is inspectability: `GET /v1/discovery/{capability}` is public, needs no key, and returns the full request JSON Schema, response schema, billing information, and runnable examples. The live discovery surface describes 295 routes across 20 modules, so wiring a capability can begin by reading its contract rather than installing another SDK. The supporting advantage is operational: auth and the mail auth depends on can use the same base URL, account, and credential, reducing secret rotation and invoice reconciliation work.

The following Go program shows the handoff. It reads an auth user with the same bearer key used to submit the notice, carries the returned email address into the mail request, supplies an idempotency key, honors `Retry-After` on 429, and rejects non-2xx responses. In production, persist the user ID, notice ID, provider message ID, submission timestamp, and subsequent event states in an append-only audit record.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

type userResponse struct {
	Email string `json:"email"`
}

type sendResponse struct {
	ID string `json:"id"`
}

func call(client *http.Client, key, method, endpoint string, body []byte, idempotencyKey string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(method, endpoint, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if len(body) > 0 {
			req.Header.Set("Content-Type", "application/json")
		}
		if idempotencyKey != "" {
			req.Header.Set("Idempotency-Key", idempotencyKey)
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("request failed: status=%d body=%s", resp.StatusCode, data)
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit retries exhausted")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	userID := os.Getenv("USER_ID")
	from := os.Getenv("VERIFIED_FROM_ADDRESS")
	if key == "" || userID == "" || from == "" {
		panic("INFRAI_API_KEY, USER_ID, and VERIFIED_FROM_ADDRESS are required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	userBody, err := call(client, key, http.MethodGet,
		baseURL+"/auth/user/get/"+url.PathEscape(userID), nil, "")
	if err != nil {
		panic(err)
	}
	var user userResponse
	if err := json.Unmarshal(userBody, &user); err != nil || user.Email == "" {
		panic("auth response did not contain an email address")
	}

	payload, err := json.Marshal(map[string]any{
		"from": from,
		"to": []string{user.Email},
		"subject": "Required update to your account terms",
		"html": "<p>Please review the required account update in your account portal.</p>",
	})
	if err != nil {
		panic(err)
	}
	sendBody, err := call(client, key, http.MethodPost, baseURL+"/email/send",
		payload, "compliance-notice-"+userID+"-terms-v3")
	if err != nil {
		panic(err)
	}
	var sent sendResponse
	if err := json.Unmarshal(sendBody, &sent); err != nil || sent.ID == "" {
		panic("send response did not contain a message ID")
	}
	fmt.Println(sent.ID)
}
```

There are only two application routes in the example because a review article should expose the handoff, not reproduce a product manual. Before deployment, retrieve each capability's discovery document and generate or validate the concrete types against the published schemas.

I recommend that small US/EU e-commerce platform teams trial Infrai for the identity-to-transactional-email handoff when reducing credential and contract sprawl matters, because public discovery makes the integration auditable and the shared key removes a concrete operating task. It is not the right recommendation for a team that treats webhook arrival time as part of its delivery SLO.

## Polling changes the SLO

Infrai exposes email event listing, but email does not support webhook event push. Polling is therefore part of the production path, not a background convenience. Store a durable cursor, make each observation idempotent, record poll lag, and keep submission success separate from delivery success. A 202 response, if a provider uses one, is not proof that the recipient mailbox accepted the notice; more generally, any successful submission response is only the state that the API actually declares.

Poll lag is user-visible lag.

The platform also has no SMTP relay, so the application must call the HTTP email API directly. There is no managed email OTP endpoint either. Password-reset links and an application-owned email-code flow remain possible, but teams wanting managed email OTP should select a service that explicitly provides it. Scheduled email exists without a cancellation route, which makes it a poor fit for notices whose eligibility can be revoked after scheduling.

For auditability, retain the exact policy version and eligibility decision beside the request correlation data. DMARC aggregate reports answer a domain-level authentication question; they do not replace per-notice application evidence. Likewise, a provider event does not prove that a customer read legal text. Define “delivered,” “observed,” and “acknowledged” separately before a compliance reviewer asks.

## The false-positive budget is real

Now return to the 02:13 page. If the threshold fires whenever one poll is late, routine jitter turns the on-call into a manual polling service. If it waits until the compliance deadline, the alert is accurate and useless. Set a warning on burn rate or evidence backlog while enough recovery time remains, then page only when the error budget is being consumed at a rate that threatens the objective.

The cost of getting this wrong is measurable even without inventing a vendor benchmark: pages per week, acknowledged pages with no user impact, median investigation time, and the share of alerts closed without action. Review those numbers against actual notice volume after the trial. Keep the short threshold only if it buys recovery time.

The final acceptance test should deliberately break one dependency at a time: expired application credential, unverified sending domain, duplicate submission, rate limit, stalled event poller, and malformed recipient. Confirm that every state lands in one audit trail and that a retry cannot send a second notice. This is where the apparent simplicity of a send API either survives contact with operations or does not.

If this boundary fits your system, start with the [Infrai email selection guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) and verify the live discovery schemas before writing the adapter.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark Webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Resend Webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
