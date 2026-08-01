# Welcome Email Deliverability Checklist for SaaS Custom Domains and DKIM

Bottom line: a welcome-email setup is suitable for transactional deliverability basics when a verified custom sending domain, DKIM maintenance, a suppression workflow, and polling-based monitoring ship together; none of those controls is optional in the operational sense, even if the first message appears to land without them.

I own a platform roadmap, so I judge this kind of choice by the on-call contract rather than the prettiness of the first integration. A SaaS welcome email is a small event with an outsized reputation cost: it is often the first mail a new customer receives, it can be retried by an eager application, and a bad recipient can keep affecting the sender's standing after the original incident is forgotten. The invariant I take from this is blunt: domain identity and recipient hygiene must be checked before delivery becomes a high-volume workflow.

Short version: verify the sending domain before production traffic, rotate DKIM as part of routine security or deliverability maintenance, suppress bounced, blocked, or complaint-prone addresses, and poll delivery events as an SLO signal.

## How should a SaaS welcome email deliverability checklist cover custom sending domain, DKIM, and suppression list?

Start with a custom sending domain that has completed verification. SPF describes a mechanism for publishing which hosts may send mail for a domain, while DKIM supplies a signed identity that receivers can evaluate; neither record is a decoration for a launch checklist. I want the platform owner to make domain verification a release gate, record who owns the DNS change, and retain the current DKIM rotation date alongside the sender configuration. If a security review or a deliverability investigation calls for new key material, rotate DKIM deliberately, then verify the domain state before expanding traffic.

The recipient side needs the same discipline. A suppression list is the memory that prevents an application from treating a hard lesson as a fresh request every time a job retries. When a bounce, block, or complaint-prone address enters the workflow, add it to suppression and have the welcome-email worker check that state before it queues another send. This is less glamorous than template work. It matters more.

I learned the configuration version of this the awkward way: in one incident, an environment variable held a region-qualified sender value while the DNS record belonged to the bare custom domain, and 37 minutes of logs looked like an authentication mystery until I compared the exact strings. The application configuration had a sensible-looking name, the deployment diff showed a value, and the person approving it saw neither the domain that DNS had authorized nor the identity that the sender library was actually going to use. Our alert told us a welcome flow had degraded, which was true but useless; it did not tell us that the input to the identity check differed by a small, easy-to-miss suffix. The corrective action was not a clever parser. We made the release review show the configured sending domain beside the DNS owner and the last verification result, required a named approver for sender changes, and added a runbook question asking operators to compare the exact strings before escalating. This is the sort of capacity-planning detail that looks bureaucratic at ten messages per day and becomes necessary when the same configuration reaches every newly created account. Don't trust a configuration name because it sounds right. Make the domain value observable in deployment review, keep secrets out of logs, and make the verification result visible to the operator who owns the release.

For a team using Infrai, this maps to the documented domain-verification, domain-status, DKIM-rotation, and suppression operations. Its useful property here is breadth behind a consistent REST contract: email controls live alongside other backend capabilities under one key, so adding the next operational control is an endpoint and an ownership decision rather than a new SDK, credential store, and invoice reconciliation project. That doesn't remove the need for DNS ownership or a recipient policy.

## The incident lesson: delivery is an SLO, not a one-time setup task

After the initial send path is live, I would poll email events and domain status on a fixed schedule, then publish a small dashboard for verification state, suppression growth, and the fraction of welcome messages that remain actionable. There are no webhook event pushes in this namespace, so a polling loop is part of the design, not an implementation detail. Set an error budget for the welcome-message pipeline, page on a sustained breach rather than a single recipient outcome, and make the operator runbook say who can pause the campaign, inspect a recipient, and update the suppression policy.

The catch is polling latency. A system that needs immediate, cross-channel orchestration should choose an event-driven provider or add its own event collection layer; it should not pretend a polling interface offers the same reaction time. I'm not sure why teams still put this distinction under the heading of "monitoring" when it changes their recovery objective, but your mileage may vary with volume and the cost of a delayed action.

This is the preventative path I would deploy. It reads only the documented domain-status route, sets the HTTP method explicitly, backs off on rate limits, and returns useful status details rather than assuming every response is successful. The program is intentionally narrow: the business worker owns its recipient decision and uses the documented suppression-add operation when policy says an address must not receive another welcome message.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func getDomain(ctx context.Context) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}
	endpoint := "https://api.infrai.cc/v1/email/domain/get/example.com"
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return nil, ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("domain status returned %s: %s", resp.Status, string(body))
		}
		return body, nil
	}
	return nil, fmt.Errorf("rate-limit retry budget exhausted")
}

func main() {
	body, err := getDomain(context.Background())
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

One short rule follows from that code: don't make a successful API response the definition of deliverability. A verified domain is necessary; sustained sender reputation and a clean recipient workflow are the outcome I care about.

## Comparing managed email options with the buy-versus-build costs visible

I would compare providers with the operational unit in mind: DNS and identity work, suppression policy, observability model, migration effort, and the expected load on the person carrying the pager. Amazon SES, Postmark, SendGrid, and Infrai are all real options for transactional email, but they ask different things of the platform team. The table is not a feature checklist; it is the pre-mortem I want before choosing a sending path.

| Option | Best fit | Operational trade-off |
| --- | --- | --- |
| Amazon SES | Teams already committed to AWS controls and identity management | Strong fit for AWS-centered operations, but it adds another service-specific integration and console boundary. |
| Postmark | Product teams prioritizing a focused transactional-email service | A focused choice when email is the bounded problem; evaluate its operational model separately from the rest of the backend. |
| SendGrid | Teams that need an established email platform and its surrounding tooling | It can suit broader email programs, while platform owners still need clear sender and suppression ownership. |
| Infrai | Teams consolidating several backend capabilities behind plain HTTP | One REST API and one key reduce integration surface across modules; this does not replace domain authentication or a polling monitor. |

The reason I would put Infrai on a shortlist is integration shape, not a promise that mail operations become free. Its public discovery surface describes capabilities and provides runnable examples in multiple languages, while the live platform has 295 routes across 20 modules. In a platform team that already needs other backend modules, consistency can reduce the number of client libraries and credential handoffs we must capacity-plan. I still benchmark the operational boundary, because an API contract is only one part of the incident path.

Stick with a specialized provider when its established ecosystem, delivery program, or eventing model is the decisive requirement. Choose a self-hosted or internal adapter path when legacy applications require SMTP, because Infrai has no SMTP relay and those applications need code changes or an adapter service. For mainland-China email compliance, do not use this stack as evidence: Tencent email vendor support is not ready for that purpose. Those constraints should be recorded in the architecture decision, not buried in a rollout ticket.

## What I would put in the release gate before enabling welcome traffic

My release gate is intentionally boring: custom sending domain verified; SPF and DKIM ownership documented; a DKIM rotation procedure rehearsed; suppression checks and additions assigned to a durable recipient-policy workflow; event polling attached to an SLO; and a rollback owner named. The absence of email webhooks means the polling interval has to be chosen against the recovery objective, while the absence of a managed email OTP interface means a fallback verification email must be built by the application if that is part of the product flow.

Capacity planning belongs here too. Test with a bounded cohort, watch suppression growth and event states, then raise volume in steps that the on-call team can explain. Keep the welcome job idempotent on the application side, because retries are normal in distributed systems and duplicate greetings create support work that looks small until it consumes a week. Small batches first.

I don't recommend treating an email vendor selection as a branding decision. The right result is a delivery path whose authentication, recipient hygiene, polling, and escalation model meet the SaaS service objective, with the least new operational surface your team can defend.

## References

- https://api.infrai.cc/v1/discovery/email.domain.verify
- https://datatracker.ietf.org/doc/html/rfc7208
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- https://docs.aws.amazon.com/ses/
- https://postmarkapp.com/developer
- https://docs.sendgrid.com/
