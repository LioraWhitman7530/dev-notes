# Invoice PDF Localisation: How to Handle Multi-Currency Right-to-Left Layouts

TL;DR: For invoice PDF localisation across multi-currency and right-to-left layouts, page on missing or unreadable monthly freight invoices, not on every slow render. The least complex reliable API approach is a queued render job whose input contains decimal amount strings, ISO currency codes, locale, explicit text direction, a pinned template revision, and a pinned font bundle. Validate that input before rendering, inspect the finished artifact, then archive the PDF and its manifest together. Choose a managed or self-hosted renderer only after the same representative Arabic and English fixtures pass that contract. Fidelity is the gate. Render cost is the capacity constraint.

The page arrives at 02:17: `monthly-invoice-archive availability below objective`. On-call can see 1,842 expected invoices, 1,817 archived objects, 14 terminal render failures, and 11 jobs whose state is unknown. A generic latency alert would be much less useful: it neither identifies missing customer records nor says whether a retry could create a duplicate. Start from this operator view and work backward.

## How should invoice PDF localisation handle multi-currency right-to-left layout?

The earlier signal is not mean latency. It is deadline risk: accepted jobs that have no verified artifact and whose remaining time is smaller than the queue drain estimate. That signal connects capacity to the business outcome without waking someone for a harmless five-minute burst. A second warning should track fixture fidelity at deployment time, before production traffic reaches a changed template, font bundle, shaping engine, or renderer.

Latency can wait.

Treat the monthly run as a reconciliation problem. Give every document a stable key such as `billing-period/account/template-revision`, record one terminal state, and compare the expected ledger with the archive index. A successful response from a render boundary is evidence of transport success, not evidence that Arabic text was shaped, columns stayed on the page, or the right amount reached the document. Completion is stricter: artifact stored, checksum recorded, manifest stored, and validation passed.

The artifact decides.

For an initial SLO, define the population and deadline before picking a percentage. One defensible example policy is: 99.9% of accepted monthly invoice jobs produce a validated, retrievable artifact within 30 minutes, excluding jobs rejected synchronously for an invalid contract. Those numbers are a planning example, not a universal target; finance, support, batch size, and recovery time should set the real values. The useful alert burns that deadline budget.

## Make localisation part of the job contract

Do not let a renderer infer money or direction from arbitrary host defaults. `1,234.50` is display text, not a safe monetary value, and a floating-point number is a poor interchange representation for an invoice amount. Carry the exact decimal string and currency code through the boundary, while the localized display string is produced and tested as presentation. Keep logical field order in the data model; apply right-to-left behavior in the document layer and isolate identifiers, dates, tracking numbers, and Latin abbreviations that must retain their intended order.

This runnable Go program validates a compact job envelope and derives a stable idempotency key. It intentionally does not implement currency formatting or bidirectional layout: those belong behind versioned, fixture-tested interfaces, rather than being guessed by a few string replacements in billing code.

That omission is deliberate.

```go
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "errors"
    "fmt"
    "regexp"
)

type Money struct {
    Decimal  string `json:"decimal"`
    Currency string `json:"currency"`
}

type RenderJob struct {
    AccountID        string `json:"account_id"`
    BillingPeriod    string `json:"billing_period"`
    Locale           string `json:"locale"`
    Direction        string `json:"direction"`
    Total            Money  `json:"total"`
    TemplateRevision string `json:"template_revision"`
    FontBundle       string `json:"font_bundle"`
}

var decimal = regexp.MustCompile(`^-?[0-9]+(?:\.[0-9]+)?$`)

func (j RenderJob) Validate() error {
    if j.AccountID == "" || j.BillingPeriod == "" {
        return errors.New("account_id and billing_period are required")
    }
    if j.Direction != "ltr" && j.Direction != "rtl" {
        return errors.New("direction must be ltr or rtl")
    }
    if !decimal.MatchString(j.Total.Decimal) {
        return errors.New("total.decimal must be an exact decimal string")
    }
    if len(j.Total.Currency) != 3 {
        return errors.New("currency must be a three-letter code")
    }
    if j.Locale == "" || j.TemplateRevision == "" || j.FontBundle == "" {
        return errors.New("locale, template_revision, and font_bundle are required")
    }
    return nil
}

func key(j RenderJob) (string, error) {
    if err := j.Validate(); err != nil {
        return "", err
    }
    canonical, err := json.Marshal(j)
    if err != nil {
        return "", err
    }
    sum := sha256.Sum256(canonical)
    return hex.EncodeToString(sum[:]), nil
}

func main() {
    job := RenderJob{
        AccountID: "carrier-1042", BillingPeriod: "2026-08",
        Locale: "ar-AE", Direction: "rtl",
        Total: Money{Decimal: "128450.75", Currency: "AED"},
        TemplateRevision: "freight-monthly-v7", FontBundle: "billing-fonts-v3",
    }
    id, err := key(job)
    if err != nil {
        panic(err)
    }
    fmt.Println(id)
}
```

The contract should also carry line-item amounts, tax labels, addresses, and shipment references in structured form; they are omitted above to keep the operational boundary visible. Never overwrite an archived object merely because a retry reused the same account and month. Compare the manifest and content checksum, return the existing verified result for an identical job, and quarantine a conflicting payload for review.

## Test the artifact, not the request

A golden test suite needs documents that are difficult in different ways: Arabic prose beside a Latin tracking code, a long carrier name, negative adjustments, zero values, two currencies that format differently, a page break inside a shipment table, and a missing optional address line. Use approved output from a pinned toolchain as the baseline. On an engine or font change, render every fixture and examine both machine checks and page images; text extraction alone cannot prove visual order, while screenshots alone cannot prove searchable text or metadata.

The deployment gate should fail on missing glyphs, unexpected page-count changes, content outside the media box, absent invoice identifiers, or a mismatch between the manifest total and extracted document fields. Pixel comparison is useful, but make its tolerance explicit because rasterization can change without changing meaning. A reviewer should see the diff rather than a single pass/fail bit.

Inspect the page.

Keep production instrumentation close to the state machine: accepted, queued, rendering, validated, archived, and terminal failure. Record durations at each transition, attempts, template revision, font bundle, locale, direction, page count, byte count, and a low-cardinality failure class. Do not put account identifiers or full locale combinations into metric labels if that creates unbounded series; retain job-level detail in logs and traces, linked by the stable key.

The earlier alert can now be expressed as pending validated artifacts divided by estimated drain capacity, grouped by billing deadline. Capacity planning then has teeth. If 60,000 documents are due inside 30 minutes and the measured sustained rate under the representative fidelity suite is 25 documents per second, nominal drain time is 40 minutes before retries or headroom; the batch cannot meet that example deadline. This is arithmetic for a load test, not a claimed benchmark. Measure the actual renderer, document mix, storage path, and concurrency ceiling.

## Buy or build around a measurable boundary

The decision is not `API versus library`. It is who owns layout-engine upgrades, font distribution, isolation, retries, regional processing, archive durability, and the pager. Price per render is rarely the dominant variable once engineers are manually reconciling missing invoices. Still, a self-hosted system can be rational when volume is steady, data placement is constrained, and the team already operates the required sandbox and queue; a managed boundary can be rational when demand is spiky and the avoided on-call surface is worth the loss of control.

This approach has limits. A queued asynchronous boundary is unsuitable when a user must edit and preview every invoice interactively, and pixel baselines impose review work whenever an intentional typography change lands. Self-hosting gives the team more control but also makes engine patching, sandboxing, burst capacity, and font licensing its operational responsibility. A managed boundary reduces some of that ownership, yet quotas, data-location terms, supported fonts, and engine controls may constrain the fidelity contract. Neither choice removes archive reconciliation. That trade-off is why the acceptance pack must precede procurement or a build commitment.

Measure it first.

| Decision factor | Managed rendering boundary | Self-hosted rendering pool | Evidence to collect |
|---|---|---|---|
| Fidelity control | Engine and font controls may be bounded by the contract | Team pins and upgrades the whole toolchain | Golden Arabic/English fixture results |
| Render cost | Usually tied to requests, pages, or capacity terms | Compute, storage, engineering, and on-call time | Cost per validated archived artifact |
| Burst handling | Contracted quotas and queue behavior matter | Team provisions headroom and backpressure | Sustained drain rate under month-end mix |
| Failure ownership | Provider and client responsibilities meet at the API | Platform team owns the full path | Recovery drill and escalation map |
| Lock-in | Request and response semantics can become proprietary | Engine assumptions can leak into templates | Time to replay the neutral job contract elsewhere |

Run the same acceptance pack against every candidate and keep the archive interface independent of the renderer. The scorecard should weight fidelity failures as disqualifying, then compare deadline attainment, operator effort, data handling, migration effort, and total operational cost. Averaging those dimensions into one attractive number hides the failure that matters.

Reject broken output.

## Close the loop without paging on noise

After a successful render, write the PDF under an immutable key and store a small manifest containing the job key, checksum, validation result, template and font revisions, renderer revision, creation time, and archive location. Reconciliation reads expected invoice keys from the billing ledger and verified keys from the archive, then emits missing, conflicting, and late sets. That gives on-call a bounded recovery action: retry an idempotent job, investigate a conflict, or restore capacity.

Tune warnings with shadow evaluation before attaching a pager. A threshold based on one slow queue sample will flap during normal bursts; a threshold that waits for the deadline will arrive too late. Require sustained deadline risk across multiple observation windows, suppress duplicate pages for the same batch, and send fidelity-regression failures from deployment fixtures to the owning team before rollout.

False positives have a direct cost: they train responders to distrust the month-end page and may trigger unnecessary concurrency increases, which can raise render cost or overload storage while no invoice is actually at risk. False negatives cost missing archives and manual reconciliation. The threshold belongs between those losses, backed by queue age, measured drain rate, remaining deadline, and the count of unverified artifacts. Page on customer-impact risk. Keep exploratory latency anomalies in a dashboard until they predict that risk.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Unicode Bidirectional Algorithm: https://www.unicode.org/reports/tr9/
- Unicode Common Locale Data Repository, currency formatting guidance: https://cldr.unicode.org/translation/number-currency-formats/number-and-currency-patterns
- Go package documentation for `crypto/sha256`: https://pkg.go.dev/crypto/sha256
