# Node.js Object Storage Manifest Cleanup When Image Replace Leaves Old Thumbnails

Short answer: treat an image replacement as a version transition, record the derivative keys you created, delete the retired keys in bounded batches, and use a one-day lifecycle rule only as an orphan backstop. A prefix scan can repair a missing manifest, but it should not be the normal request path when the product promise is that old thumbnails disappear promptly.

The least complex design that meets that promise is a small manifest beside the image record. The renderer writes immutable keys such as `images/{imageID}/{generation}/thumb.webp`, then the application moves one pointer to the new generation. Cleanup consumes the old generation's manifest after the pointer change.

That sequence matters. It separates “which image is current?” from “which bytes are safe to remove?” and gives an SRE an observable boundary for each operation.

## The replacement incident I design for

In a production review, I assume the resize worker can stop after uploading two of four derivatives, or after the uploads finish but before the database transaction commits. That is not a dramatic edge case; it is the ordinary failure window between two durable systems. A cleanup design that requires a perfect, atomic write across both systems will eventually leave an orphan. The useful exercise is to trace one image through the half-finished path: the original row still points to generation 17, the worker has already placed `thumb.webp` and `card.webp` under generation 18, and a retry must know whether those two objects are safe to reuse or should be replaced. A key derived from a mutable filename lets the retry overwrite bytes that a still-running request is reading. A generation-bearing key lets the retry inspect a manifest, finish the missing derivatives, and publish one complete set without guessing. That is the kind of small naming decision that keeps a cleanup incident from becoming a serving incident.

The invariant is simple: every generation has a name, a status, and a bounded set of keys. The current pointer is changed only after the expected set exists and its content metadata has been checked. The old generation becomes `retired` in the same database transaction as that pointer update. A worker can then retry retirement by generation ID without listing the bucket first.

Keep the operation boring.

Emit a record for the generation, the number of keys requested for deletion, the number acknowledged, and the final outcome. Alert on age of the oldest retired generation, not on a single transient request failure. That gives the cleanup SLO a measurable definition: for example, “99% of retired derivatives are unreachable within 10 minutes,” with a separate repair objective for stragglers. The alert should carry the image ID and generation, because an on-call engineer needs to replay one bounded unit rather than launch a bucket-wide scan while traffic is high. I have seen teams make the opposite choice: they attach the alert to total object count, which rises for healthy uploads and says nothing about whether a particular replacement is still serving stale data. Capacity planning and correctness need separate gauges.

## Should Node.js use a prefix list, batch delete, or lifecycle for old thumbnails?

Choose by the required removal latency and the failure you are willing to operate. The Node.js application can own the policy even if the storage adapter is implemented in another language; the important part is that the adapter receives explicit keys and returns per-key outcomes.

| Path | Removal expectation | Operational shape | Poor fit |
| --- | --- | --- | --- |
| Manifest plus batch delete | seconds to minutes, after the worker runs | work scales with image replacements | no durable record of derivative keys |
| Prefix list on replacement | bounded by list and delete calls | simple to prototype, but scans unknown state | high replace rate or large prefixes |
| Scheduled prefix reconciliation | hours, depending on schedule | repair job scans a known generation space | strict deletion SLOs |
| Lifecycle expiration | one day is the shortest useful age rule | low on-call burden; storage evaluates eligibility | immediate removal or audit-grade erasure |

The catch is that lifecycle is asynchronous. It is a retention guard, not a request/response delete API. Use it for abandoned staging generations and other objects whose eventual removal is acceptable. If an old thumbnail must be unreachable during the same user action, issue an explicit delete and wait for the storage API's acknowledgement before reporting the cleanup complete.

## How should a Go worker make prefix cleanup idempotent?

The worker below receives a retired generation and a storage interface. Its batch size is deliberately bounded; the concrete adapter can map `DeleteBatch` to the object store's multi-object delete operation. It does not guess keys from a mutable filename, and it can safely run again after a timeout because deleting an already absent key is treated as an acceptable end state by the adapter contract.

```go
package cleanup

import (
	"context"
	"fmt"
)

const batchSize = 1000

type ObjectStore interface {
	DeleteBatch(ctx context.Context, keys []string) (deleted int, missing int, err error)
}

type RetiredGeneration struct {
	ImageID   string
	Generation string
	Keys      []string
}

func Retire(ctx context.Context, store ObjectStore, gen RetiredGeneration) error {
	for start := 0; start < len(gen.Keys); start += batchSize {
		end := start + batchSize
		if end > len(gen.Keys) {
			end = len(gen.Keys)
		}
		deleted, missing, err := store.DeleteBatch(ctx, gen.Keys[start:end])
		if err != nil {
			return fmt.Errorf("retire image=%s generation=%s offset=%d: %w",
				gen.ImageID, gen.Generation, start, err)
		}
		if deleted+missing != end-start {
			return fmt.Errorf("retire image=%s generation=%s: incomplete acknowledgement", gen.ImageID, gen.Generation)
		}
	}
	return nil
}
```

The `missing` count is intentional. A retry after a worker crash should converge to the same state, not page someone because the second attempt found nothing to remove. Permission failures, malformed keys, and partial responses are different: retain the failed keys in the manifest and retry with an exponential backoff, while a reconciliation job compares the manifest with the current pointer.

For a Node.js service, keep the same boundaries in the caller: persist the generation manifest, enqueue retirement, and return the new image URL only after the new generation is complete. The JavaScript event loop does not change the storage semantics. A queue, a cron process, or a Go worker can consume the record; the contract is what keeps the choice reversible.

## What changes when object versioning is enabled?

An object key is not the same thing as an object version. In a versioned bucket, deleting a key can create a delete marker while older versions remain billable and retrievable by version ID. A cleanup dashboard that counts only visible keys can therefore report success while retained bytes continue to grow.

Decide which guarantee you need. If the requirement is “the old thumbnail is not served,” a delete marker plus application-level access checks may be enough. If the requirement is “the old bytes are removed,” the worker must include noncurrent-version handling and the retention policy must cover those versions. Keep the evidence: generation ID, key or version ID, request result, and timestamp.

NIST's HIPAA security guidance is a useful reminder that disposal is a control with records, not merely a storage optimization. For regulated images, define who can request deletion, what the retention exception is, and how an auditor can connect the business event to the object-store result.

## When is this cleanup pattern the wrong choice?

It is not suitable when the derivatives are append-only and a one-day retention window is acceptable; a lifecycle rule alone is less code and less on-call work. Stick with a direct prefix reconciliation when the catalog is small, replacement traffic is rare, and the list cost is immaterial to your capacity plan.

It is also the wrong fit for a write-once archive with legal hold or WORM requirements. An explicit delete worker should not be allowed to override that policy; route the request into an approval workflow and preserve the hold metadata.

I'm not sure a single global SLO is meaningful for every image class. Your mileage may vary by region, replication mode, and whether thumbnails are cached outside the bucket. Measure reachability at the serving layer, then choose the deletion mechanism that can actually meet that measurement.

The practical compromise is a manifest-driven fast path, a scheduled prefix scan for repair, and a one-day lifecycle rule for abandoned work. Three layers. Each has a different job.

## References

- NIST SP 800-66 Rev. 2: https://csrc.nist.gov/pubs/sp/800/66/r2/final
- Object versioning and delete markers: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
