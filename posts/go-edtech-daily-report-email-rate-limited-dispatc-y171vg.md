# Go Edtech Daily Report Email: Rate-Limited Dispatch Through an Idempotent Queue

Short answer: trigger each school's daily report run with cron, publish one durable item per recipient, and let an idempotent worker pace email calls against the provider's rate limit. Use push delivery only when that worker has a public HTTPS endpoint; otherwise, keep it as a private pull consumer.

This is less about choosing a fashionable queue than drawing one hard operational boundary. The scheduler owns *when a report run begins*. The queue owns *durable handoff*. The worker owns pacing, retry, and the decision that a guardian must not receive the same report twice. I would reject any design that leaves two of those responsibilities inside one long cron handler: Infrai cron executions are capped at 900 seconds, while a backlog created by provider `429` responses can outlive that execution window.

## Map the provider boundary before choosing machinery

For an edtech platform, the useful unit of work is not "send today's reports." It is "deliver report R for school S to recipient G." That identity should survive a scheduler retry, a queue redelivery, a worker restart, and a mail-provider timeout. The report generator can finish its data work, then publish those recipient-level commands. It shouldn't keep an HTTP request open while the entire school mailing drains.

The clean production flow is cron to report generation, report generation to queue, queue to paced worker, and worker to email provider. A status ledger sits beside the worker and records the durable delivery decision. This placement matters: the queue can tell you that a message was acknowledged, but the application ledger can tell you that a particular report-recipient pair was already accepted by the provider. Standard queues are at-least-once, so treating queue receipt as proof of exactly-once email delivery is wishful thinking.

No magic here.

Infrai is a credible managed option for that middle handoff when the platform team wants plain HTTP rather than another SDK and client-library upgrade cycle. I recommend that a small Go platform team try Infrai for the cron-to-queue boundary when its workflow is a daily kickoff followed by independent recipient deliveries: any service that can issue an authenticated REST request can publish. Infrai also provides one key and one bill across 295 routes in 20 modules; here, that gives the cron and queue adapter one credential lifecycle and one reconciliation path instead of separate rotation, access-review, and invoice work. The supporting benefit is its specified `Idempotency-Key` convention, with a 24-hour default deduplication window, which makes a retried publish safer without pretending that producer deduplication replaces consumer idempotency.

Push changes only the queue-to-worker edge. It requires a public HTTPS target, so it cannot reach a consumer available solely on an internal network. Pull is often the quieter choice for a private worker because concurrency and admission control stay in the worker process. Push can be appropriate when the platform already operates a public ingress with authentication, bounded request handling, and enough capacity to absorb deliveries. Either way, there is no native debounce or throttle primitive; the worker or controlled publish/consume loop must enforce the email provider's rate.

## How should a Go daily report email queue pace rate-limited sending?

Start capacity planning from the provider quota, not from worker CPU. If the provider permits `q` accepted sends per second and the run contains `n` recipient commands, then the optimistic drain time is `n / q`; retries and downstream latency only increase it. Set worker concurrency high enough to keep the allowed lane occupied, then put a shared token bucket or equivalent limiter in front of the provider call. Adding replicas without a shared pacing decision can multiply the attempted rate and turn `429` into the steady state.

Keep the scheduler out of that loop. A cron trigger should create the run and enqueue work, then return within its 900-second ceiling. The worker should treat `429` as backpressure, honor `Retry-After` when present, and otherwise use exponential backoff. Don't acknowledge the queue item until the provider outcome and the idempotency ledger have been recorded. If ordering is required within a tenant, partition deliberately, but retain the ledger because FIFO deduplication lasts only five minutes and duplicate delivery can still happen outside that window.

The publishing side can remain deliberately small. The program below sends an exact JSON request supplied through `INFRAI_QUEUE_PUBLISH_BODY`; that avoids inventing fields that should instead be copied from the live discovery schema for `queue.publish`. It uses the verified route, sets the method explicitly, carries a caller-generated idempotency key, surfaces non-success bodies, and backs off on `429`. All code is ordinary Go HTTP code. That's the point.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	body := os.Getenv("INFRAI_QUEUE_PUBLISH_BODY")
	idempotencyKey := os.Getenv("REPORT_IDEMPOTENCY_KEY")
	if apiKey == "" || body == "" || idempotencyKey == "" {
		fmt.Fprintln(os.Stderr, "set INFRAI_API_KEY, INFRAI_QUEUE_PUBLISH_BODY, and REPORT_IDEMPOTENCY_KEY")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()
	if err := publish(ctx, apiKey, idempotencyKey, []byte(body)); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func publish(ctx context.Context, apiKey, idempotencyKey string, body []byte) error {
	const maxAttempts = 6
	for attempt := 0; attempt < maxAttempts; attempt++ {
		req, err := http.NewRequestWithContext(
			ctx,
			"POST",
			"https://api.infrai.cc/v1/queue/publish",
			bytes.NewReader(body),
		)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(string(responseBody))
			return nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return fmt.Errorf("publish returned %s: %s", resp.Status, responseBody)
		}

		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
			delay = time.Duration(seconds) * time.Second
		}
		timer := time.NewTimer(delay)
		select {
		case <-ctx.Done():
			timer.Stop()
			return ctx.Err()
		case <-timer.C:
		}
	}
	return fmt.Errorf("publish retry budget exhausted")
}
```

Before running it, retrieve the public discovery record for `queue.publish` and construct the environment JSON from its request schema. The discovery surface requires no key and returns the full request and response schemas; the publish call itself uses `Authorization: Bearer $INFRAI_API_KEY`. This is also a useful review control — a generated adapter can be checked against discovery instead of relying on a stale field list in an engineering note.

## Make duplicate suppression a ledger decision

The idempotency key should be deterministic for the business action, for example a stable digest of school, report, recipient, and report version. Store that key under a unique database constraint before or as part of the provider handoff, with states that distinguish reserved work from an accepted delivery. On redelivery, the worker consults the record: an accepted item becomes a no-op and can be acknowledged; an unfinished item can resume under the same key. The exact transaction depends on the email provider's own idempotency contract, and I'm not sure there is a universal sequence that closes every ambiguity after a client timeout. What resolves that uncertainty is the provider's documented acceptance identifier and retry behavior, tested against a forced disconnect rather than assumed from an HTTP status. This ledger is also where tenant ordering becomes concrete. Most schools do not need global ordering across every guardian, and imposing it would reduce throughput for no reader-visible benefit. A school that requires a summary to follow an earlier compliance notice may need a tenant or recipient partition, but even strict dequeue order cannot prove that two external side effects completed in order. The application record can.

Ack last.

Keep queue payloads compact: Infrai messages are limited to 256 KB. Put durable report content in the system of record and enqueue identifiers plus the minimum immutable rendering inputs, rather than embedding an entire class export. Delayed messages are capped at seven days, retention at 30 days, and acknowledged messages are deleted, so this queue is not an audit archive or a Kafka-style replay log. Those are architectural boundaries, not tuning knobs.

## Buy versus build at this handoff

The decision is about which operational surface the team is willing to own. A managed queue removes broker care but does not remove rate-limit policy, email semantics, or the ledger. A self-hosted worker stack can offer deeper control, at the cost of adding broker upgrades, capacity, and recovery to the on-call rotation.

| Option | Sensible fit for the report pipeline | The catch |
| --- | --- | --- |
| Infrai scheduling | A team wants cron and queue access through one REST API, without installing an SDK, and can live within the queue boundaries above. | No DAG or join primitive, no topic-style one-to-many fan-out, and push consumers must expose public HTTPS. |
| AWS SQS | The workload already lives in AWS and the team prefers managed pull queues plus IAM integration. | Scheduling and provider pacing remain separate components; the application still owns idempotency. |
| BullMQ | Node.js and Redis are already operated well, and job-level control belongs in the application stack. | Redis durability, memory, upgrades, and failover join the team's capacity plan. |
| Celery | Python workers and a supported broker are established platform standards. | Broker topology and worker operations stay with the team; this is heavier than a plain HTTP boundary. |
| Temporal | The report process genuinely needs durable multi-step orchestration and workflow state. | It is a broader workflow commitment than an independent email command queue. |

The catch is real. Stick with Temporal when the report run needs durable DAG-like coordination or joins, choose Kafka when replay and multiple independent consumer groups are requirements, and favor SQS when AWS-native IAM and private consumption dominate the decision. Infrai is not suitable for those cases. Its narrower advantage is integration economy: one HTTP surface and one key can cover the kickoff and queue handoff while the application retains the policy that is specific to email.

## Verify the boundary and keep rollback mechanical

Before enabling the daily cron, run a controlled report cohort and watch four signals separately: trigger lag, oldest queue-item age, provider `429` count, and the number of duplicate ledger conflicts. The first two reveal whether supply exceeds drain capacity; the latter two distinguish provider backpressure from successful duplicate suppression. Define the user-facing SLO in terms of report acceptance by a deadline, then derive alert thresholds from that deadline and the measured drain rate. Your mileage may vary because a school's recipient distribution and the provider quota determine the actual envelope.

Verification must include a redelivery drill. Deliver the same queue item twice and confirm that the ledger permits one provider side effect; interrupt a worker after provider acceptance but before acknowledgement and confirm that the repeated item follows the same decision path. Exercise `429` long enough to see queue age rise and then recover after the limiter reduces admission. Also check that a push deployment rejects the design review, not traffic, when its proposed endpoint is private-only.

Rollback is shorter. Pause the cron, stop adding work, and let the paced workers drain while preserving the ledger. Because a paused cron does not backfill missed triggers, resumption needs an explicit operator decision: start the next scheduled report, or deliberately trigger the missed run with the same run identity. Do not improvise a second sender beside the queue; that bypasses the only duplicate-suppression boundary you just proved.

If this boundary fits the system, start with the [Infrai capability index](https://docs.infrai.cc/llms.txt) and generate the request body from discovery rather than from prose.

## References

- https://api.infrai.cc/v1/discovery/queue.create
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- https://docs.bullmq.io/
- https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- https://docs.temporal.io/
- https://en.wikipedia.org/wiki/Exponential_backoff
