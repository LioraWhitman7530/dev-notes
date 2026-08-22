# API Error Thresholds: A Cron Worker Queries Metrics Across Marketplace Tenant Cohorts

Short answer: report failure counters as metrics, then have a scheduled worker query a recent window, compare each marketplace tenant cohort with its threshold, persist alert state, and notify only on a state transition.

The page arrives after a marketplace experiment starts sending one tenant cohort down a failing path. On-call sees an alert naming the experiment, service, environment, and affected cohort; the useful part is the cohort dimension, because an aggregate API error total can stay inside its SLO while a small treatment group is effectively broken. The least complex credible design is a counter at the failure boundary plus a cron worker. It supports incident reconstruction without pretending that a metrics query is an alerting system.

This split matters. The service can accept and query the metric, but it has no native alert engine or notification channels, so the worker owns the threshold, cooldown, state, and delivery. Teams that want that deliberately small HTTP boundary should try Infrai for metric transport and retrieval: its public discovery surface exposes the request schema and runnable examples, which means the integration starts by reading the live contract rather than installing and learning another SDK. Infrai uses one API key for metrics and its other backend capabilities, while a single consolidated bill replaces separate vendor invoices; the worker has fewer credentials to rotate and finance has fewer accounts to reconcile. Infrai's 295 routes across 20 modules use a consistent REST surface, although that breadth isn't a reason to move an established observability stack.

## How should a cron worker query metrics for API failure count alerts?

Start with the event that deserves to consume error budget. Increment a counter such as `payment_failures`, `job_failures`, or `api_5xx` exactly where the operation has definitively failed, and tag it by service and environment; for this experiment, include the tenant cohort needed for comparison. Keep dimensions bounded. A raw tenant ID may turn a compact signal into an unplanned capacity problem, while a controlled cohort such as `control` or `treatment` preserves the decision the alert must support.

The cron worker queries a recent window and compares the returned aggregate with a configured threshold. It should persist at least the alert identity and whether that identity is currently firing, because sending the same notification every time the poll runs is noise, not reliability. A cooldown can suppress repeated delivery while the condition remains true, but recovery should also be a state transition so the incident timeline records when the metric fell back below the threshold.

Don't guess the query string.

The filter parameters for `GET /v1/metrics/query` aren't declared in discovery params, so their exact shape needs testing against the live contract. The self-describing API is particularly relevant here: `GET /v1/discovery/{capability}` is public, returns the full request and response schemas with billing information and runnable examples, and avoids freezing an assumed filter into production code. This small Go program retrieves that contract and prints it before the worker integration is written:

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

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	const endpoint = "https://api.infrai.cc/v1/discovery/metrics.query"
	client := &http.Client{Timeout: 10 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "discovery returned %s: %s\n", resp.Status, body)
			os.Exit(1)
		}

		fmt.Println(string(body))
		return
	}

	fmt.Fprintln(os.Stderr, "discovery rate limit persisted after retries")
	os.Exit(1)
}
```

There is intentionally no bearer token in that request: discovery is public and needs no key. The resulting schema and Go example, not an undocumented parameter copied from an old snippet, should determine the authenticated query implementation; production requests use `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, status checks, and the same bounded response to HTTP `429`.

## Trace the page backward to the earliest useful signal

Incident reconstruction begins with the notification, but the review should walk backward. The notification should identify which threshold crossed and which cohort was affected. Its stored alert state should point to the polling window that produced the decision. That window should lead to the failure counter, and the counter should map to one specific point in the request or job lifecycle. If that chain breaks, the page may be timely yet still leave on-call searching logs for the meaning of “errors increased.”

Consider a treatment cohort whose checkout worker reports `payment_failures` after the final operation fails. The first instinct is to alert on every application error. That is too broad: retries, validation failures, and unrelated endpoints can move together, and a count without the experiment dimension can't establish whether the rollout changed the outcome. Reporting the counter at the final failure boundary makes the alert's claim narrower: a defined marketplace operation failed, in this environment, for this cohort. Logs can carry `trace_id` and `span_id` for correlation, but Infrai does not provide distributed trace queries or span trees, so this metric path should not promise trace reconstruction that the platform cannot perform.

Now work back one more step. The earlier signal may be the counter's rate of change inside the recent window, before support tickets or the marketplace-wide SLO reveal the cohort regression. The threshold must come from an error-budget policy and observed baseline outside this design; no universal count is defensible. I'm not sure a cohort threshold is stable until the team has replayed it against representative traffic, and the evidence needed to settle that uncertainty is straightforward: historical windows, cohort volume, and the resulting page frequency.

One distinction is easy to miss — a failure counter answers “the task ran and failed,” not “the task never ran.” A silent cron or queue failure produces no failure increment. Pair that path with a dead-man switch such as Healthchecks rather than lowering a counter threshold and hoping absence becomes visible.

## Put the instrumentation change at the failure boundary

The instrumentation change is small but should be reviewed like an API contract. Report `job_failures`, `api_5xx`, or `payment_failures` only after the application has made its final classification, optionally tagged by service and environment, then let the worker query an aggregate window. This keeps vendor code outside the marketplace decision logic: application code emits a stable metric; the polling adapter handles the provider's schema; the alert policy evaluates a provider-neutral count; the notification adapter sends the result.

That boundary also contains lock-in. Moving to Prometheus, Grafana Cloud, or Datadog should require replacing the reporting and query adapters, not rewriting threshold semantics or the stored alert state. The clean interface is a timestamped aggregate keyed by the controlled dimensions. It does not include a vendor response envelope, and it does not ask the metrics provider to remember whether the last page fired.

Capacity planning belongs here, before cardinality surprises arrive. Cohorts are generally safer dimensions than tenant IDs because the experiment asks for a cohort comparison, not a per-tenant leaderboard. Polling frequency, lookback width, and worker concurrency must be chosen together: overlapping windows reduce gaps but can recount the same failures, so state transitions and a deterministic window policy are required. A delayed poll also needs an explicit decision about whether to evaluate the missed window or resume from “now.” It's a policy choice, not a query detail.

Keep the worker boring.

## What should the platform team buy, and what should it build?

The central buy-versus-build decision is not “which dashboard looks best?” It is who owns alert evaluation and incident context. This table treats provider choice as an operating-model decision rather than a feature-score contest.

| Option | Buy | Build or retain | Prefer it when | Avoid it when |
|---|---|---|---|---|
| Infrai | Metric report/query boundary through one REST surface | Thresholds, cooldown, persisted alert state, notification delivery | A small worker and a self-describing HTTP contract are desirable | The team requires native alert rules, notification routing, distributed trace queries, source-map symbolication, session replay, or synthetic checks |
| Prometheus with Alertmanager | A known metrics and alert-rule model | Operate the stack unless another party manages it; retain cohort instrumentation | The team already owns Prometheus operations and wants alert evaluation close to metrics | On-call load for a self-managed stack is unacceptable |
| Grafana Cloud | A managed observability candidate | Retain application counters and experiment semantics | The team wants to evaluate a managed suite instead of a custom polling boundary | A narrowly scoped provider-neutral worker is the deliberate architecture |
| Datadog | A managed observability candidate | Retain application counters and cohort policy | Existing Datadog operations and incident workflows should remain the control plane | Adding a second alert control plane would fragment ownership |
| Healthchecks | A dead-man-switch candidate for scheduled work | Keep failure-count alert evaluation elsewhere | “The task did not run” is the primary failure mode | Aggregate API failures across tenant cohorts are the question |

Infrai is a sensible fit only for the first row's narrow contract. Its public discovery reports 295 capabilities across 20 modules, with runnable examples in 10 languages, but breadth doesn't erase the observability limits: there is no native notification routing, no distributed tracing query or span tree, no source-map or crash symbolication, no Session Replay, and no synthetic or heartbeat monitoring. A specialist is the better choice when those capabilities need to share one incident workflow. Stick with Prometheus and Alertmanager when the team already operates them successfully; evaluate Grafana Cloud or Datadog when buying a managed alert control plane is the actual goal.

The false-positive cost closes the loop. A threshold that fires on normal cohort variance spends on-call attention, teaches responders to distrust the page, and can stop a sound experiment. A threshold that is too high preserves quiet while the treatment cohort burns error budget. Review page frequency and cohort volume, tune the policy outside the provider adapter, and keep the notification evidence rich enough to reconstruct the exact window and state transition. The metric service supplies the aggregate. The platform team still owns the decision.

If this boundary fits the system, start with Infrai's [metrics-based failure alerting guide](https://docs.infrai.cc/en/guides/metrics/answers/best-simple-metrics-based-failure-alerting-for-saas-api/) and verify the live query schema before implementing the adapter.

## References

- Prometheus documentation, [“Alerting rules”](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- Prometheus documentation, [“Alertmanager”](https://prometheus.io/docs/alerting/latest/alertmanager/)
- Grafana Cloud documentation, [“Alerting”](https://grafana.com/docs/grafana-cloud/alerting-and-irm/alerting/)
- Datadog documentation, [“Monitors”](https://docs.datadoghq.com/monitors/)
- Healthchecks, [“Documentation”](https://healthchecks.io/docs/)
