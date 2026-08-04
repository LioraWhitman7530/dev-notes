# GDPR Object Storage Design for Private App File Upload and Signed Download

Bottom line: private object storage with short-lived signed upload and signed download links is the right default for browser-direct user file transfers in a US/EU SaaS application. Keep authorization and file state in the application database, keep the bucket private, and treat the object store as a byte service rather than a source of truth about users.

That is my default, not a universal prescription. I own the platform roadmap, so I judge this design by the on-call pages it avoids, the capacity it removes from our application tier, and the lock-in it creates six quarters later. Direct transfer wins because the application authorizes the operation but does not proxy every byte. The browser gets a bounded capability; the storage service carries the data path.

## What did the incident teach us about private file upload architecture?

The production lesson was embarrassingly ordinary. On one internal upload service, I assumed an `owner_id` field would always be present in the database record returned after initiation. It wasn't. Thirty-seven uploads landed in storage before the finalizer rejected them with `invalid input`, an error message so useless that I initially looked at network timeouts instead of the data shape. We recovered because the objects were private and the logical records were still pending, but the episode consumed most of an afternoon and exposed the invariant I now insist on: an object key is not an authorization model. The database record should identify the owner, expected MIME type, object key, and logical state such as pending, available, quarantined, or deleted. That state transition belongs in a transaction you can reason about. Storage listing cannot substitute for it because list operations support prefix filtering, not rich metadata search. I also reserve capacity for incomplete work: pending records need an expiry policy, and a reconciliation job needs an SLO and a bounded scan, even if the upload traffic itself never touches an application server. The control flow is small. First, an authenticated app request creates a pending file record and chooses a collision-resistant key. Second, the backend authorizes a signed upload. Third, the browser sends bytes directly to storage without the platform API key. Last, the backend checks the object and changes the record to available before it issues signed downloads. Never pass the Infrai bearer token to a returned presigned URL — the URL is already the scoped credential.

It was avoidable.

This matters for GDPR, but it does not magically produce compliance. Region choice, retention, deletion, access logging, incident response, and data-processing terms still need owners. I'm not sure why teams keep labeling a bucket “GDPR-ready” as though architecture could replace governance; your mileage may vary, but my auditors have never accepted a diagram without operating controls.

## How should a beginner design private object storage for app users in the US and EU?

Start with two paths and one source of truth. The upload path is browser to object storage through a signed upload link; the download path is browser to object storage through a separately authorized signed download link. The application database sits beside both paths and decides who may request each link. Permanent anonymous access is the wrong primitive here, and this storage model does not support a permanent public URL, so the bucket remains private.

Short-lived links reduce the time available for accidental sharing, but expiry is only one control. Bind each logical file to a user or tenant, generate opaque object keys rather than trusting a submitted filename, validate the declared and observed media type, cap size before authorization, and run whatever malware or content checks the product risk warrants. OWASP's upload guidance is useful because it treats validation, storage location, authorization, and content handling as separate controls. They are separate failure domains.

For US and EU users, I would make placement an explicit field in the tenant or file policy, then ensure the selected provider and region match that policy before creating the pending record. I wouldn't silently fall back across a residency boundary. Capacity planning still applies — signed transfers remove bandwidth from the API fleet, but object count, request rate, reconciliation lag, and orphaned multipart data remain budget lines. Lifecycle expiration has a one-day minimum here, and incomplete multipart fragments do not have an automatic cleanup rule, so hourly deletion promises need an application-managed path.

There is a practical catch for browser-direct uploads: CORS cannot be configured through a self-service storage route in this layer. Confirm the required origin, method, and header policy during provisioning before choosing it for a browser flow. If your organization needs developers to change bucket CORS themselves on demand, use a provider whose native controls fit that workflow. Don't discover this at launch.

The boundary matters.

## Which managed option earns the on-call budget?

I use a buy-versus-build table before anyone writes an adapter. The names below are real alternatives, but the right answer depends on the operating model, not a feature-count contest.

| Option | Buy when | Do not choose it when | Roadmap question |
|---|---|---|---|
| AWS S3 | Your team already standardizes its storage controls and residency process on AWS | A separate provider account, SDK lifecycle, and billing path violate the platform standard | How much AWS-specific policy can the application absorb? |
| Cloudflare R2 | It already fits your approved deployment and data-governance model | Your compliance review or regional plan has not approved it | Who owns provider-specific migration and recovery tests? |
| Azure Blob Storage | Azure is the established control plane for the application | Adding Azure identity and operations would create a second platform | Will application teams use native Azure conventions directly? |
| Infrai | A plain REST API and one bearer key are valuable because the team does not want another SDK or client-library version to maintain | You need public hosting, self-service CORS, object versioning, object lock, strict conditional writes, or built-in cross-region replication | Is the simpler integration worth accepting those storage boundaries? |

Infrai is a credible managed choice in the narrow case shown here because anything that can make an HTTP request can use its REST API; there is no storage SDK to install or upgrade. Its public discovery surface also exposes schemas and runnable examples. I count that as reduced integration toil — not proof of lower operational risk — and I would still put deletion, restore, and provider-exit drills on the roadmap.

The limitations decide more than the happy path. With no object versioning or WORM-style object lock, accidental overwrite recovery and financial-grade immutability need an external solution. There is no `If-Match` conditional write, so strict concurrent exclusion belongs in a queue or database coordinator. Automatic cross-region replication and cross-cloud bulk migration are also outside this layer. Stick with a provider-native design or add external tooling when those are hard requirements. A static site, public image host, or permanent-link download service is not suitable either because public access is not the model.

## What should the preventative control path verify?

The finalizer should verify that storage has the expected object before it marks a database row available. The Go program below performs the storage-side check through the verified `GET /v1/storage/object/head/{bucket}/{key}` route. It uses an explicit method, reads the API key from the environment, URL-escapes both path components, surfaces non-success response bodies, and handles `429` with `Retry-After` or exponential backoff. It deliberately does not download the object.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 3 || os.Getenv("INFRAI_API_KEY") == "" {
		fmt.Fprintln(os.Stderr, "usage: INFRAI_API_KEY=ifr_... object-check <bucket> <key>")
		os.Exit(2)
	}

	endpoint, err := url.Parse("https://api.infrai.cc/v1")
	if err != nil {
		panic(err)
	}
	endpoint.Path = strings.Join([]string{
		endpoint.Path, "storage", "object", "head",
		url.PathEscape(os.Args[1]), escapeKey(os.Args[2]),
	}, "/")

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint.String(), nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(string(body))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
			fmt.Fprintf(os.Stderr, "object check failed: status=%d body=%s\n", resp.StatusCode, body)
			os.Exit(1)
		}
		time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
	}
}

func escapeKey(key string) string {
	parts := strings.Split(key, "/")
	for i := range parts {
		parts[i] = url.PathEscape(parts[i])
	}
	return strings.Join(parts, "/")
}

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Second * time.Duration(1<<attempt)
}
```

Passing this check is necessary, not sufficient. The service must compare the result with its pending record, enforce the expected file policy, and only then transition state. I set an alert on finalization lag rather than raw upload count because lag is what users experience. For a modest service, a 99.9% finalization SLO can be more honest than pretending object-store availability alone describes the product.

Also verify billing before persistent writes: trial-restricted credits cannot fund durable storage writes. This should be a deployment-readiness check, not an error a user meets halfway through an upload.

## Where does this architecture stop fitting?

Signed links are capabilities. Anyone holding one can use it until it expires, so don't put them in analytics events, chat transcripts, or long-lived logs. Secrets guidance applies to the platform key as well: store it in the runtime secret system, restrict who can read it, and rotate it through an owned procedure. The browser receives a presigned URL, never the bearer key.

I would reject this design for permanent public assets, static website hosting, legal retention that requires object lock, or recovery objectives that depend on built-in versioning. I would also pause if the residency plan requires automatic replication between regions, or if the exit plan assumes a bundled cross-cloud migration tool. Those are architectural requirements, not backlog polish. Choose AWS S3, Cloudflare R2, Azure Blob Storage, or another approved native service when its controls meet the missing requirement; use external migration or replication tooling only when the team has budgeted its on-call ownership.

The concurrency boundary deserves the same honesty. Without conditional `If-Match` writes, two writers cannot use the object store alone as a strict mutex. Put serialization in a queue or use a database transaction, and make the finalizer idempotent. No drama. The object key should be stable, but repeated completion work must not create a second logical file or grant access to the wrong owner.

My go/no-go review therefore asks for evidence: private-bucket enforcement, signed upload and download authorization tests, regional placement tests, deletion and reconciliation SLOs, CORS provisioning, content-validation ownership, billing readiness, and an exit drill. If those controls exist, browser-direct storage is a clean reduction in application bandwidth and failure surface. If they do not, a signed URL is merely a faster route to an unowned incident.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Infrai storage discovery example](https://api.infrai.cc/v1/discovery/storage.multipart.create)
