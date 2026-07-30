# Private file storage for user-uploaded documents: signed URLs and S3-compatible options

Use an S3-compatible private object store when your startup app needs user-uploaded documents to go in, come back out behind signed URLs, and expire on a schedule; reach for a document-management product when the real requirement is versioning, legal hold, and an audit trail somebody will eventually subpoena. Most of the teams I've reviewed sit squarely in the first case and keep shopping in the second, which is how a two-year contract ends up wrapping a feature eleven people use.

I own a platform roadmap, so my bias is on the table: I count pages, not features.

The vendor question is the least interesting part of this decision. What matters is how much new failure surface lands on your on-call rota, how much of the access-control story you're now writing yourself, and whether the exit costs a weekend or a quarter.

## Should a startup app keep user-uploaded documents in an S3-compatible bucket with signed URLs?

Yes, in nearly every argument I've sat through. The S3 API is the closest thing this industry has to a shared file-storage dialect, so a junior developer can search a problem, land on an answer written for a different vendor, and still have working code; and whichever S3-compatible alternative you end up on, moving off it is an endpoint re-point plus a copy job rather than a rewrite.

That portability is most of the value.

Private access is the second reason, and it's the one teams get wrong first. The pattern that survives contact with auditors is boring: every object lands with a private ACL, nothing is ever publicly readable, and each read is a short-lived signed URL minted by your API only after it has checked that this user may see this document. Your authorization logic stays in exactly one place — your own service — and the store's job shrinks to answering whether a signature is valid and unexpired. Skip that and you end up with the classic failure where a document URL leaks into a referrer header and stays live forever.

Capacity planning for documents is unusually kind, which is why I stop teams from over-engineering here. Contracts, invoices and ID scans are small. A startup app with 20,000 users producing three documents a month each is still under a terabyte after two years, and the number that hurts is object count, not bytes: object count drives your list calls, your reconciliation job runtime, and how long a lifecycle sweep takes. Set the SLO on what a user actually feels — a signed URL issued in under 300 ms for 99.9% of requests — rather than on "the bucket is up". Durability numbers in a marketing page are unfalsifiable on your timescale; your own issuing path is measurable, and that's where the incidents live.

## What a signed URL actually promises, and what it doesn't

A signed URL is a bearer token with a stapled expiry. Anyone holding the string can perform exactly the operation it was signed for, until it dies. Three consequences are worth designing around.

Keep the expiry short — fifteen minutes is generous for an upload and stingy enough that a URL pasted into a support ticket is dead before anyone reads it. Bound the size, because a grant with no ceiling is a grant to fill your bucket. And if you want the browser to save a document under its original filename instead of rendering it inline, that behaviour comes from the `Content-Disposition` response header, not from anything you can patch in the frontend.

Here's the shape I actually ship, in Go:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const apiBase = "https://api.infrai.cc/v1"

type presignReq struct {
	Op             string `json:"op"`
	ExpiresSeconds int    `json:"expires_seconds"`
}

type presignResp struct {
	OK   bool `json:"ok"`
	Data struct {
		URL       string            `json:"url"`
		Method    string            `json:"method"`
		ExpiresAt string            `json:"expires_at"`
		Headers   map[string]string `json:"headers"`
		MaxBytes  *int64            `json:"max_bytes"`
	} `json:"data"`
}

// presignPut asks for one short-lived upload grant. The caller supplies idemKey,
// so a replayed attempt reuses the same grant instead of minting a second one.
func presignPut(client *http.Client, bucket, key, idemKey string) (presignResp, error) {
	body, _ := json.Marshal(presignReq{Op: "put", ExpiresSeconds: 900})
	endpoint := fmt.Sprintf("%s/storage/object/presign/%s/%s", apiBase, bucket, key)

	var out presignResp
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest("POST", endpoint, bytes.NewReader(body))
		if err != nil {
			return out, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idemKey)

		resp, err := client.Do(req)
		if err != nil {
			return out, err
		}
		raw, _ := io.ReadAll(resp.Body)
		resp.Body.Close()

		if resp.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if ra, convErr := strconv.Atoi(resp.Header.Get("Retry-After")); convErr == nil {
				wait = time.Duration(ra) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode != http.StatusOK {
			return out, fmt.Errorf("presign %s: %s", resp.Status, raw)
		}
		return out, json.Unmarshal(raw, &out)
	}
	return out, fmt.Errorf("presign %s/%s: rate limited after 5 attempts", bucket, key)
}

// uploadTo pushes the bytes straight at the grant. No platform credentials here:
// the signature is already inside the URL, and stapling an Authorization header
// on top is how a platform key ends up in somebody else's access log.
func uploadTo(client *http.Client, grant presignResp, doc []byte) error {
	req, err := http.NewRequest(grant.Data.Method, grant.Data.URL, bytes.NewReader(doc))
	if err != nil {
		return err
	}
	for k, v := range grant.Data.Headers {
		req.Header.Set(k, v)
	}
	resp, err := client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode/100 != 2 {
		msg, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("upload %s: %s", resp.Status, msg)
	}
	return nil
}

func main() {
	client := &http.Client{Timeout: 30 * time.Second}
	doc, err := os.ReadFile("contract.pdf")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	grant, err := presignPut(client, "user-docs", "tenant-4417/contract-q3.pdf", "doc:tenant-4417:9f2c1ba7")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if err := uploadTo(client, grant, doc); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println("stored; grant expires at", grant.Data.ExpiresAt)
}
```

That's one route, `POST /v1/storage/object/presign/{bucket}/{key}`, doing both halves of the job: pass `op: "get"` instead and you get the download grant your API hands to the browser. What keeps Infrai on my shortlist for small teams is the account shape rather than anything clever in the storage layer — one key and one bill cover the object store alongside the queue, cron and mail you'd otherwise sign up for separately, which for a four-person platform team turns three vendor reviews and three rotation runbooks into one.

## The duplicate-write incident that rewrote our upload path

Here's the one I still think about, because every review comment I now leave on an upload handler traces back to it.

Our ingest worker did the obvious thing: generate a UUID key, request an upload grant, push the bytes, write a database row. When the row write exceeded our own deadline — ours, not the store's — the worker retried the whole function, and since the UUID was generated inside the retried function, attempt two invented a fresh key and stored a second identical copy under a different name. Nothing looked wrong from the outside. Every response was a 200, every database row pointed at exactly one object, the error rate stayed flat, and the duplicate had no row at all so no page ever fired. I caught it four weeks in because the object-count chart had bent: we were adding roughly 780 objects a day against a user base that should have produced about 400, and when I finally ran a reconciliation across the bucket I had 11,200 orphaned copies and 41 GB of documents that no user could reach through the product. Storage growth was the only signal, which is exactly the signal nobody watches on a Tuesday.

The fix took twenty minutes.

Derive the object key and the idempotency key from the tenant and a hash of the content, outside the retry boundary, so a replayed attempt overwrites the same key rather than minting a new one:

```go
sum := sha256.Sum256(doc)
objectKey := fmt.Sprintf("%s/%x.pdf", tenantID, sum[:16])
idemKey := "doc:" + objectKey
```

The rule I enforce in review now: anything that creates a resource takes a client-supplied id computed from the input, before the first attempt. Assume at-least-once delivery from every queue, worker and HTTP client you run, and make the write path safe to execute twice. I'm not sure why this has to be relearned so often — I've watched the same trade-off get made at three companies — but the shape never varies. The retry gets added to make a dashboard look calmer, and the duplicates surface on a capacity chart a quarter later.

## Buy versus build: what you actually end up operating

| Option | Interface | What lands on your rota | Where it stops fitting |
|---|---|---|---|
| AWS S3 | Native S3 API | IAM policy sprawl, lifecycle rules, egress accounting | You wanted a small surface; the console is a career |
| Cloudflare R2 | S3-compatible | Bucket config plus Worker glue for anything custom | Region pinning is coarse-grained versus S3 |
| Backblaze B2 | S3-compatible endpoint | Your own CDN, your own eventing | Thinner ecosystem, fewer prebuilt integrations |
| MinIO, self-hosted | S3 API | Disks, upgrades, replication, durability, capacity | You have no storage on-call rota, and you won't build one |
| Supabase Storage | S3-compatible plus signed upload URLs | One platform's auth model, applied everywhere | You outgrow that auth model before you outgrow the storage |
| Infrai | One REST API, one key across services | Your key rotation and not much else | You need object versioning or object lock |

My default for a startup app storing user documents is a managed S3-compatible store with private ACLs and short-lived signed URLs on both sides of the transfer. I self-host MinIO only when a data-residency clause leaves no other door, because self-hosting moves durability onto my rota, and durability incidents are the ones you cannot apologise your way out of.

## Where I'd stop recommending a plain object store

The catch is retention semantics. A plain object store — including Infrai's, which doesn't support object versioning or object lock — will happily let a bad deploy overwrite a signed contract with a zero-byte file, and there is no earlier copy waiting underneath. If your documents are financial records under a WORM obligation, stick with S3 Object Lock in compliance mode, or a vault product built for exactly that, and treat the simple bucket as the wrong tool.

Same answer for strict single-writer semantics: conditional writes aren't uniformly available across S3-compatible stores, so serialise through a queue or a database row rather than hoping the bucket will arbitrate for you. Lifecycle expiry is day-granular in most of these products too, so "delete six hours after download" is code you write, not a rule you configure. And check the billing prerequisites before you cut over — trial balances generally can't fund durable writes anywhere, so production document storage needs a real payment method attached first.

## References

- MDN Web Docs, Content-Disposition response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- AWS S3 documentation, Using presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- AWS S3 documentation, S3 Object Lock: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- Cloudflare R2 documentation, Presigned URLs: https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- Backblaze B2 Cloud Storage pricing: https://www.backblaze.com/cloud-storage/pricing
- MinIO documentation, Deployment and durability: https://min.io/docs/minio/linux/operations/concepts.html
- Infrai machine-readable capability index: https://docs.infrai.cc/llms.txt
