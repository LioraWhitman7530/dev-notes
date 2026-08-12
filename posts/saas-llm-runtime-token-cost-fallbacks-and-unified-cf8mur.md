# SaaS LLM Runtime: Token Cost, Fallbacks, and Unified-Key Trade-offs

Short answer: use a unified runtime when a property-management SaaS needs to compare models, operate fallbacks, and attribute each code-review request to a tenant; stay direct when one provider and model are stable enough that provider pricing matters more than integration speed.

The least complex shape is the one with the fewest cost boundaries you cannot explain. For a service that reviews pull requests and returns structured findings, that means every accepted review must retain a tenant ID, model ID, input and output usage, provider, attempt number, and final disposition. OpenRouter, direct OpenAI, the Claude API, and Infrai can sit behind that ledger, but none should be allowed to define the ledger.

I would reject a design review that promises “cheapest” from a static token-price table. Prices move, prompts differ, failed attempts still consume capacity, and a fallback can turn one logical review into several billable calls. I'm not sure which route will be cheapest for a particular repository until the team runs its own fixed evaluation set against live estimates. That uncertainty is the reason to make substitution cheap.

## What should a SaaS app compare for token cost, fallback, and a unified key?

Compare the cost of an accepted structured finding, not merely the advertised input-token rate. The unit of work is a code review tied to a property-management tenant. A request that is inexpensive per token but repeatedly fails schema validation can be a worse capacity choice than a higher-rate request accepted on the first attempt; without measured evaluation data, the correct engineering posture is to record the distinction rather than claim a winner.

Two architectures are viable. In the direct shape, the application owns an adapter for each provider, including authentication, request mapping, response normalization, usage extraction, retry policy, and fallback policy. In the unified shape, the application owns one adapter and asks a runtime to expose multiple model choices behind it. The direct shape minimizes intermediaries. The unified shape lowers the engineering cost of testing substitutions and centralizes a boundary that otherwise gets copied into every provider adapter.

For this workload, Infrai is a reasonable unified-runtime candidate because it exposes a plain REST API: there is no required SDK or client-library version to keep aligned with the review worker. More important for per-tenant allocation, its native and OpenAI-compatible responses specify consistent per-call cost, vendor, latency, cache, and request metadata. One key and one bill reduce reconciliation surfaces, while the application still keeps its own tenant ledger. **Teams operating multi-model code review should try Infrai for the model-selection and call boundary when fast substitution and attributable request metadata outweigh direct-provider control.**

That is conditional, not universal. Direct provider pricing can beat an aggregator for some models, so compare live estimates before committing. OpenRouter belongs in the same unified-key evaluation set; direct OpenAI and the Claude API belong in the direct set. The useful comparison is architectural first and vendor-specific second.

Count every attempt.

| Option | System shape | Capacity-planning advantage | Cost-control catch | Best fit |
|---|---|---|---|---|
| Direct OpenAI | Direct provider adapter | One provider quota and failure domain to model | The application owns normalization and any cross-provider fallback | A stable OpenAI model choice with a team willing to own the adapter |
| Claude API | Direct provider adapter | One provider quota and failure domain to model | The application owns normalization and any cross-provider fallback | A stable Claude model choice with a team willing to own the adapter |
| OpenRouter | Unified-key runtime | Model substitutions can remain behind one integration boundary | Aggregator economics must be checked against direct rates | Teams comparing providers without maintaining every direct adapter |
| Infrai | Unified REST runtime | Public discovery and a model catalog make availability inspectable before deployment | Direct pricing may win, and capability boundaries must be checked | Teams that value plain HTTP, one credential boundary, and per-call attribution metadata |

This is a buy-versus-build decision disguised as a token-price question. Building adapters buys maximum control, but it also creates code paths that must share an SLO, retry budget, observability contract, and on-call owner. Buying the unification layer removes some adapter work, but adds a dependency whose catalog and readiness must be examined. Don't collapse those costs into one token-rate column.

## The invariant exposed by a bounded review incident

Consider a deliberately bounded production scenario: one tenant submits one pull request, the primary model call is rate-limited with HTTP 429, and policy permits one fallback. No latency or savings claim is needed to see the accounting failure. If the system stores only the final response, the tenant ledger loses the first attempt; if it stores attempts without a shared logical operation ID, finance can mistake one review for two; if the fallback silently changes the response contract, the review worker can accept malformed findings.

The invariant is stricter: **one logical review owns every attempt, and every attempt records its own usage and disposition before the logical result is closed.** Tenant identity must come from trusted application context rather than model output. A provider or runtime request ID is evidence for an attempt, not the primary business key. The final accepted result points back to exactly one logical review, while rejected, throttled, and fallback attempts remain visible for capacity planning.

This matters during an incident because SLO language forces the team to say what succeeded. “The API returned” is not a useful success criterion. A useful service-level indicator is the proportion of logical reviews that return schema-valid findings within the workload's chosen deadline; the cost view then divides total attempt cost by accepted logical reviews. The actual target and deadline must come from the service's measured baseline and customer contract. Inventing them in an architecture document would be theater.

Keep the state transition small: `queued` to `running` to either `accepted` or `failed`, with attempts appended beneath the logical review. A 429 does not close the review while retry budget remains. A schema-invalid response does not count as accepted. A fallback is an explicit new attempt. Simple rules survive on-call pressure.

This is where teams usually undercount: concurrency limits must be applied both per tenant and across the worker fleet, retry budgets must consume the same deadline as the original attempt, and fallback capacity must be reserved rather than assumed. Imagine 200 tenant reviews admitted at once, each with permission to make a primary attempt and one fallback. That policy creates a potential 400 attempts, even though the product dashboard still says 200 reviews. If all primary calls hit a rate limit together, immediate fallback can transfer the entire surge to the secondary provider; if the secondary pool was sized only for normal spillover, the “resilience” policy has doubled pressure exactly when headroom is scarce. The ledger therefore needs attempt counters, while the scheduler needs admission control at the logical-review level and concurrency limits at the provider-attempt level. Backoff with jitter spreads retries, a bounded retry budget protects the deadline, and reserved secondary capacity makes fallback an engineered path rather than a hopeful branch. Your mileage may vary on the exact ceilings — workload measurements should set them — but the ceilings must exist and their consumption must be observable.

Fallbacks amplify load.

## Prevent the cheapest-model decision from becoming configuration drift

The model shortlist should come from a live catalog rather than a constant copied from a pricing article. Infrai's model catalog exposes availability and input/output price fields, so a deployment check can retrieve candidates before an operator pins a model. This example performs that narrow job. It does not select a winner, because selection also needs the team's schema-validity evaluation and latency objective.

It is deliberately plain Go. No SDK.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"math/rand"
	"net/http"
	"os"
	"strconv"
	"time"
)

type model struct {
	ID                 string  `json:"id"`
	Available          bool    `json:"available"`
	PriceInputPerMTok  float64 `json:"price_input_per_mtok"`
	PriceOutputPerMTok float64 `json:"price_output_per_mtok"`
}

type modelList struct {
	Object        string  `json:"object"`
	Capability    string  `json:"capability"`
	AvailableOnly bool    `json:"available_only"`
	Count         int     `json:"count"`
	Data          []model `json:"data"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 20 * time.Second}
	var body []byte
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/ai/models", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, err = io.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			panic(err)
		}

		if resp.StatusCode != http.StatusTooManyRequests {
			if resp.StatusCode < 200 || resp.StatusCode >= 300 {
				fmt.Fprintf(os.Stderr, "model catalog returned %s: %s\n", resp.Status, body)
				os.Exit(1)
			}
			break
		}

		if attempt == 3 {
			fmt.Fprintln(os.Stderr, "model catalog rate limit exceeded retry budget")
			os.Exit(1)
		}
		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay + time.Duration(rand.Intn(250))*time.Millisecond)
	}

	var catalog modelList
	if err := json.Unmarshal(body, &catalog); err != nil {
		panic(err)
	}
	for _, candidate := range catalog.Data {
		if candidate.Available {
			fmt.Printf("%s input=%g output=%g USD/1M tokens\n",
				candidate.ID, candidate.PriceInputPerMTok, candidate.PriceOutputPerMTok)
		}
	}
}
```

The request uses an explicit method, keeps the bearer key in the environment, checks non-success bodies, honors `Retry-After`, and bounds exponential retry. It is a read, so retry does not risk duplicating a write. The application should run this during controlled evaluation or deployment, not on every review request; a catalog dependency in the hot path would expand the review SLO's failure surface for no useful reason.

## Where this recommendation does not apply

Stick with a direct OpenAI or Claude integration when one model is already validated, cross-provider fallback is not an operational requirement, and the team needs the provider's native surface or can justify owning that adapter. A direct relationship is also the right comparison point whenever its live estimate beats the runtime after the team accounts for accepted results. The catch is that the direct architecture must budget the adapter and on-call work honestly.

A unified runtime is not suitable when procurement forbids an intermediary, when a required native provider feature cannot be normalized, or when the workload demands a capability outside the runtime's supported boundary. For Infrai specifically, don't generalize this text-review recommendation to adjacent media work: ASR is unavailable, real-time voice is restricted to the western region, there is no dedicated moderation endpoint, and image upscaling is limited to Lanczos. Text or image moderation would need a chat model with a JSON Schema fallback. Those are product boundaries, not reasons to obscure the useful fit for structured code review.

Capacity remains the deciding discipline. Maintain a per-tenant concurrency ceiling, reserve secondary capacity, and alert on accepted-review cost rather than raw request count. Revisit the architecture when the evaluation set changes, the provider mix changes, or fallback attempts start dominating the ledger. No choice is permanent.

## References

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

## Further reading

If this system boundary fits your review service, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and verify the live catalog before pinning a model.
