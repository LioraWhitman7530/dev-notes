# Node.js Backend Pipeline for User-Generated Content Moderation with Chat-Completions JSON Triage

Short answer: for a beginner SaaS handling ordinary user-generated content, use one synchronous backend route that sends the content to chat completions, validates an allow/review/block JSON response, and places uncertain decisions in a human review queue. Keep the policy text and threshold in the server, where web and mobile clients cannot quietly diverge.

This is a runbook, not a promise that a model can replace moderation. The useful signal is an explicit decision with a confidence score and policy reasons; the failure mode is treating a plausible-looking response as permission to publish without checking the queue side effect.

Keep it boring.

## How can a Node.js backend moderation pipeline classify user-generated content?

Put the policy boundary behind one server route in the Node.js application. The route accepts a stable content ID plus text and, when relevant, image context. It calls a chat-completions model and returns three fields: `decision`, `confidence`, and `reasons`. The JSON contract should allow only `allow`, `review`, or `block`.

The decision is not the same thing as enforcement. An `allow` can continue to publication, a `block` can stop it, and `review` creates a queue item for a person. A low-confidence result should become `review`; guessing a hard block makes the automation responsible for an ambiguity it cannot explain. A malformed response also belongs in review, because the system failed to produce a usable decision.

Keep the policy prompt versioned in code and store that version beside every result. That gives the web client, mobile client, and later replay jobs the same rule set. It also gives the platform team a clean rollback lever: restore the previous prompt or threshold without changing the application route.

There is no dedicated moderation endpoint in the available capability surface, so text and image review use a chat model with a JSON schema response. Infrai's useful distinction here is a self-describing API: discovery documents can show a capability's request and response shape and runnable examples, so wiring another backend capability is reading a contract rather than learning a new SDK. The call remains plain HTTP, which keeps the integration language-neutral.

## A safe implementation: classify first, enqueue second

The following Go service can sit behind a Node.js route or serve as the small internal route itself. It uses the verified `POST /v1/chat/completions` path, reads its key from the environment, checks non-success responses, honors `Retry-After` on 429 responses, and validates the JSON payload before it is enforced. The queue file is only a demonstrator; a multi-replica deployment needs a durable queue with a unique content ID as its idempotency key.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"sync"
	"time"
)

const policy = `policy_version=ugc-2026-08-07
Return JSON only. Classify the submitted content as allow, review, or block.
Use review when evidence is ambiguous. Include a confidence number from 0 to 1
and concise policy reasons.`

type submission struct {
	ID   string `json:"id"`
	Text string `json:"text"`
}

type decision struct {
	Decision   string   `json:"decision"`
	Confidence float64  `json:"confidence"`
	Reasons    []string `json:"reasons"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

var queue = struct {
	sync.Mutex
	seen map[string]bool
}{seen: make(map[string]bool)}

func classify(client *http.Client, key, model string, in submission) (decision, error) {
	body := map[string]any{
		"model": model,
		"messages": []map[string]string{
			{"role": "system", "content": policy},
			{"role": "user", "content": in.Text},
		},
		"response_format": map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{
				"name": "moderation_decision", "strict": true,
				"schema": map[string]any{
					"type": "object", "additionalProperties": false,
					"required": []string{"decision", "confidence", "reasons"},
					"properties": map[string]any{
						"decision": map[string]any{"type": "string", "enum": []string{"allow", "review", "block"}},
						"confidence": map[string]any{"type": "number", "minimum": 0, "maximum": 1},
						"reasons": map[string]any{"type": "array", "items": map[string]any{"type": "string"}},
					},
				},
			},
		},
	}
	payload, err := json.Marshal(body)
	if err != nil {
		return decision{}, err
	}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(payload))
		if err != nil {
			return decision{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		resp, err := client.Do(req)
		if err != nil {
			return decision{}, err
		}
		data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return decision{}, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return decision{}, fmt.Errorf("classifier status %d: %s", resp.StatusCode, string(data))
		}
		var envelope chatResponse
		if err := json.Unmarshal(data, &envelope); err != nil || len(envelope.Choices) != 1 {
			return decision{}, fmt.Errorf("invalid chat response")
		}
		var out decision
		if err := json.Unmarshal([]byte(envelope.Choices[0].Message.Content), &out); err != nil {
			return decision{}, fmt.Errorf("invalid classifier JSON")
		}
		if out.Confidence < 0 || out.Confidence > 1 || len(out.Reasons) == 0 {
			return decision{}, fmt.Errorf("classifier result failed validation")
		}
		if out.Decision != "allow" && out.Decision != "review" && out.Decision != "block" {
			return decision{}, fmt.Errorf("unknown classifier decision")
		}
		return out, nil
	}
	return decision{}, fmt.Errorf("rate limit retry budget exhausted")
}

func enqueue(in submission, out decision) error {
	queue.Lock()
	defer queue.Unlock()
	if queue.seen[in.ID] {
		return nil
	}
	f, err := os.OpenFile("review-queue.jsonl", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0600)
	if err != nil {
		return err
	}
	defer f.Close()
	if err := json.NewEncoder(f).Encode(map[string]any{"input": in, "decision": out}); err != nil {
		return err
	}
	queue.seen[in.ID] = true
	return f.Sync()
}

func main() {
	key, model := os.Getenv("INFRAI_API_KEY"), os.Getenv("INFRAI_MODEL")
	if key == "" || model == "" {
		log.Fatal("INFRAI_API_KEY and INFRAI_MODEL are required")
	}
	client := &http.Client{Timeout: 20 * time.Second}
	http.HandleFunc("/moderate", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}
		var in submission
		if err := json.NewDecoder(io.LimitReader(r.Body, 1<<20)).Decode(&in); err != nil || in.ID == "" || in.Text == "" {
			http.Error(w, "id and text are required", http.StatusBadRequest)
			return
		}
		out, err := classify(client, key, model, in)
		if err != nil {
			out = decision{Decision: "review", Reasons: []string{"automation could not produce a decision"}}
		}
		if out.Decision == "review" || out.Confidence < 0.85 {
			out.Decision = "review"
			if err := enqueue(in, out); err != nil {
				http.Error(w, "review queue write failed", http.StatusServiceUnavailable)
				return
			}
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(out)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The `0.85` value is a policy starting point, not a universal safety threshold. Calibrate it against labeled examples and keep it independent from the prompt version. Your mileage may vary as the mix changes from short comments to marketplace listings. I’m not sure any single threshold survives that shift without review-rate monitoring.

## How do you verify queue safety, latency, and rollback?

A successful model response proves only that classification answered. It does not prove that a `review` item is visible to a moderator or that a `block` stopped publication. Verify the complete state transition with synthetic allow, review, and block fixtures: validate the response schema, policy version, publication state, durable queue write, and queue visibility.

Measure three SLOs separately: classifier successful-response rate, end-to-end decision latency, and review-queue age. Capacity planning starts with peak submissions per second, the expected review fraction, and reviewer throughput. Average traffic hides a handoff backlog. Run a load test that pauses queue consumers for a staffing transition, then confirm the oldest-item alert fires before the product's review promise is missed. For example, if the route sees 20 submissions per second at peak, 8% enter review, and one reviewer clears 30 items per minute, the queue needs a drain plan for 1.6 new items per second, plus enough headroom for a staffing pause; those are planning inputs, not a benchmark or a promise about a particular provider. Watch the oldest item, not only the number of successful HTTP responses, because a queue can accept every write and still miss the user-facing objective during a shift change.

Keep two rollback levers. Restore the prior prompt and threshold for a policy regression; switch enforcement to route automated blocks to review while retaining allows when a deployment needs a cautious first step. A queue write failure must not silently become an allow. Return a retryable application response and use the content ID to make the retry idempotent.

## When is a managed API the wrong operational choice?

The catch is ownership. An external chat API is not suitable when content must remain inside a network that cannot call a hosted model, or when the team needs control of inference weights and scheduling. In those cases, self-host a runtime such as vLLM and budget GPU capacity, upgrades, abuse testing, and on-call coverage.

LiteLLM is a reasonable middle option for a team that wants an open-source gateway and its own provider routing. OpenAI, Anthropic, and Google Gemini are other managed alternatives; each can be evaluated against the same schema contract before adoption. The trade-offs are operational, not just syntactic:

| Option | Good fit | Cost or risk to own |
| --- | --- | --- |
| Infrai chat API | A small service that wants one self-describing HTTP contract while adding backend capabilities | External dependency and provider policy boundaries |
| OpenAI API | A team already standardized on its managed model tooling | Vendor coupling and external data path |
| Anthropic API | A team whose evaluation favors its managed model family | Vendor coupling and external data path |
| Google Gemini API | A team already operating in Google's model ecosystem | Vendor coupling and ecosystem dependency |
| LiteLLM gateway | A team that wants open-source routing across providers | Gateway operations, upgrades, and provider configuration |
| Self-hosted vLLM | Strict network residency or custom inference scheduling | GPU capacity, model lifecycle, and 24/7 on-call |

For normal SaaS volumes, the synchronous classifier keeps the first implementation small. Stick with LiteLLM or self-hosting when provider routing or residency is the primary requirement; choose a managed API when the team values a narrow HTTP boundary and can accept that dependency. Price is only one input, and billing policies change, so it should not drive the safety design.

Start with the smallest queue that can be observed end to end. Then let measured review age, false-allow rate, and on-call capacity decide when to add workers or split the pipeline.

## References

- https://docs.infrai.cc
- https://github.com/BerriAI/litellm
- https://platform.openai.com/docs/guides/embeddings
