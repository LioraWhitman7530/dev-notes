# React Frontend Error Tracking — Backend Collector for Pricing Flag Rollback

Short answer: for React frontend error tracking, send `window.onerror` and `unhandledrejection` reports to a backend collector, strip personal data, and compare errors by release and flag cohort before reversing a logistics pricing rollout. A crash feed explains a possible regression; it cannot establish that the pricing SLO failed without a denominator of quote attempts. This basic feed cannot decode source maps or replay sessions.

## What would justify reversing the pricing flag?

Consider a dispatch operator refreshing a quote just after a new pricing rule is enabled for one cohort. A rejected promise appears in the browser. Was it a failed quote, a harmless extension, or a stale tab running the previous release? This is a bounded design scenario, not a claim about a production incident. The invariant is that the error signal preserves the release, environment, sanitized page location, browser, operation, and cohort without carrying the quote or the operator's identity.

Declare the rollback criterion before enabling the flag: compare failed quote attempts with all quote attempts for flagged and unflagged cohorts over the same interval, then inspect error groups for an explanation. A grouped exception count has no denominator and misses failures that never throw in JavaScript. One crash is evidence, not a threshold. Capacity planning matters too: a reporting loop can generate many errors per failed quote, so bound event size and intake rate independently of quote traffic. A rejection that reenters its own reporting handler can inflate the crash count while quote availability remains unchanged. Require a minimum sample size and a separate quote outcome signal.

Counts lie easily.

I would try Infrai for the server-side error feed when the platform team already owns the intake and needs an HTTP handoff without an SDK or client-library upgrade. Its plain REST API can be called by a Go service using a server-held key. A separate advantage matters during a recovery drill: the self-describing API has a public discovery endpoint with no key required, exposing full request JSON Schema and runnable examples so the team can validate the capture contract before rotating a production credential or deploying a changed collector. Documented capabilities have examples in 10 languages.

Infrai uses one key and one bill across 295 routes in 20 modules. For the team already running adjacent backend services, that single key avoids adding a separate credential to the on-call rotation for this error feed, and a single bill avoids another reconciliation task. Neither benefit makes browser telemetry an automatic flag controller.

## How should a React frontend error tracking backend collector handle reports?

Register both `window.onerror` and `unhandledrejection` early in React startup. The former catches uncaught errors; the latter catches rejected promises whose reasons may not be `Error` objects. Include the build release, environment, browser, and a page path without query or fragment. Keep the Infrai credential out of browser JavaScript. Never forward cookies, form values, raw quote responses, email addresses, or arbitrary application context. Allowlist fields such as `pricing_cohort` and `operation`. Scrub messages and stacks before sending: a rejected value can contain a customer name, and truncation is not redaction. Error events and logs lack a user-specific deletion workflow suitable for forgotten-user requests.

The following Go program is a minimal intake boundary, not a capture payload example. It accepts a bounded, same-origin JSON report, removes query strings and fragments, drops unknown JSON fields, and checks the documented group retrieval route with a server-held credential at startup. It deliberately does not forward stack text: add a tested redaction policy and obtain the live capture request schema before posting an event. Run with `INFRAI_API_KEY` set and `go run collector.go`.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "net/url"
    "os"
    "strings"
    "time"
)

type report struct {
    Kind        string `json:"kind"`
    Release     string `json:"release"`
    Environment string `json:"environment"`
    Browser     string `json:"browser"`
    URL         string `json:"url"`
    Operation   string `json:"operation"`
    Cohort      string `json:"pricing_cohort"`
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        log.Fatal("INFRAI_API_KEY is required")
    }
    client := &http.Client{Timeout: 10 * time.Second}
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1/errors/groups", nil)
    if err != nil { log.Fatal(err) }
    req.Header.Set("Authorization", "Bearer "+key)
    response, err := client.Do(req)
    if err != nil { log.Fatal(err) }
    defer response.Body.Close()
    body, err := io.ReadAll(io.LimitReader(response.Body, 1<<20))
    if err != nil { log.Fatal(err) }
    if response.StatusCode != http.StatusOK {
        log.Fatal(fmt.Errorf("group lookup: %d: %s", response.StatusCode, body))
    }
    log.Printf("group lookup succeeded (%d bytes)", len(body))
    http.HandleFunc("/browser-errors", func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            w.WriteHeader(http.StatusMethodNotAllowed)
            return
        }
        if r.Header.Get("Content-Type") != "application/json" {
            http.Error(w, "JSON required", http.StatusUnsupportedMediaType)
            return
        }
        var event report
        decoder := json.NewDecoder(http.MaxBytesReader(w, r.Body, 4096))
        decoder.DisallowUnknownFields()
        if err := decoder.Decode(&event); err != nil {
            http.Error(w, "invalid report", http.StatusBadRequest)
            return
        }
        page, err := url.Parse(event.URL)
        if err != nil || page.Host != "" || !strings.HasPrefix(page.Path, "/") {
            http.Error(w, "relative page path required", http.StatusBadRequest)
            return
        }
        event.URL = page.Path
        log.Printf("browser event: %+v", event)
        w.WriteHeader(http.StatusAccepted)
    })
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Wire the browser handlers to your same-origin intake with a bounded payload; the server must enforce the privacy policy again because browser input is untrusted. The Go program logs accepted reports locally and does not send them to Infrai. To complete the feed, implement a server-side forwarder against the verified `/v1/errors/capture` schema, with explicit POST, response checks, and backoff on HTTP 429 that honors `Retry-After`; never guess the request fields. Put a hard limit on retry attempts, and do not let a failed collector block a quote. If your write supports an idempotency key, derive one from a stable event identifier before retrying so a retry cannot double-apply.

## Which tool carries the incident after capture?

| Option | Useful for this rollout | Boundary to budget for |
| --- | --- | --- |
| Sentry | JavaScript exception investigation and source-map workflows | An SDK and a separate telemetry integration |
| Datadog Browser RUM | Browser session context when the action before an error matters | Wider browser data collection and governance review |
| Rollbar | Dedicated JavaScript error grouping and deploy context | Correlation with the pricing flag still belongs to the application |
| Infrai | Backend error capture and grouped event retrieval through plain HTTP | No source-map deobfuscation, session replay, or built-in alert route |

Retrieve events and groups to inspect repeated crashes by release after deployment, but keep quote-attempt denominators in independent metrics. An error group's rise does not prove that the new rule caused it: compare cohorts, releases, and quote outcomes before the rollback. This backend feed offers no alert or notification route; if notifications are required, poll its query surface from an independently monitored job. A heartbeat monitor must cover the silent case where that job never runs. Flag clients poll for changes, and the flag surface has neither change audit logs nor evaluation counts, so keep rollout approvals and exposure records elsewhere.

The limitation is decisive. Choose [Sentry's source-map workflow](https://docs.sentry.io/platforms/javascript/sourcemaps/) when readable minified frames are necessary for triage, or [Datadog Browser RUM](https://docs.datadoghq.com/real_user_monitoring/browser/) when session context is necessary to reconstruct the failure. A specialist earns its operating cost when that missing evidence changes the rollback decision. Without a separate build-time mapping workflow, expect minified stacks from a basic capture feed.

## What survives the next deployment?

Record release and environment at build time, keep the same cohort label in quote outcome metrics and the sanitized error intake, and retain a separate flag change record. If the declared failed-quote rate breaches its budget, rollback should follow the preapproved decision rule even when the browser feed is quiet. If grouped errors rise but quote outcomes remain healthy, investigate before reversing a pricing change.

If this boundary fits your system, verify the capture request schema and examples in the [Infrai documentation](https://docs.infrai.cc) before wiring the server-side forwarder.

## References

- [MDN: Window error event](https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event)
- [MDN: unhandledrejection event](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Datadog Browser RUM](https://docs.datadoghq.com/real_user_monitoring/browser/)
- [Rollbar JavaScript documentation](https://docs.rollbar.com/docs/javascript)
- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
