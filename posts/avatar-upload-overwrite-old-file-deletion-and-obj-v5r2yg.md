# Avatar Upload Overwrite: Old File Deletion and Object Storage Rollback Without Versioning

Short answer: upload each new avatar under an immutable key, validate it, switch the database pointer, and delay deletion of the old key until the recovery window expires. Overwrite-in-place without versioning is appropriate only when losing the previous file is an explicit product decision.

The constraint is operational, not aesthetic. At 03:00, an operator needs to know which bytes are authoritative, how fast a bad image can be undone, and whether cleanup can threaten the upload SLO. A stable filename hides those answers. This is a small data path with a surprisingly large failure surface because publication, retention, and deletion happen in different systems and on different clocks; each transition needs to be observable, repeatable, and reversible.

## What should happen when an avatar upload replaces an old object?

Treat replacement as publication of a candidate, not as mutation of the live file. Generate a collision-resistant key such as `users/42/avatars/01J...`; write the candidate; check size, content type, and any image policy; then atomically update `active_avatar_key`. The reader follows that pointer, so a failed upload leaves the current avatar untouched.

This split also makes concurrency explicit. A compare-and-swap on the expected old key prevents two browser tabs from silently publishing in an order neither user intended. A candidate left behind by a failed swap is garbage, but it is harmless garbage that a collector can inspect later.

Stable keys are still reasonable for disposable derivatives that can be regenerated from an authoritative source. They are a poor fit when support needs an undo button, moderation needs the prior image, or a transform may publish the wrong output. With no storage-side versioning or application-held copy, overwriting the sole object removes the application’s rollback target.

## Choose the recovery window before choosing versioning

Define a recovery point objective and recovery time objective for the avatar path first. A community profile might retain replaced objects for seven days; a regulated identity image may require a different retention, audit, and erasure review. I’m not sure one default fits both, and your mileage will vary with policy and jurisdiction.

| Approach | Rollback action | Storage footprint | Operational trade-off |
| --- | --- | --- | --- |
| Stable key, no versioning | Restore from a separate backup, if available | One live object | Simple day-to-day path, painful recovery |
| Stable key with versioning | Select a retained object version | Every retained version | Lifecycle and restore procedures are mandatory |
| Immutable keys plus pointer | Change the active pointer | Active plus retained inactive files | Requires a reaper and reference reconciliation |
| Immutable keys plus versioning | Pointer rollback plus object recovery | Highest footprint | Two retention policies to test and explain |

The final row is not automatically safer. Two retention systems create two expiration semantics and more capacity to forecast. Use it only when the extra failure domain maps to a written objective.

Capacity planning must include churn: active bytes plus replaced bytes created during the recovery window, not merely `users * average avatar size`. Track retained bytes against that forecast, and keep pricing assumptions in an owned runbook because regions and rates change. Cost is a constraint, not the rollback argument.

## How do you write, publish, and delete the old avatar safely?

The sequence is deliberately boring: write, validate, publish, retain, collect. Deletion is asynchronous and guarded by a fresh reference check. Never make the upload request wait for the delete worker; a cleanup outage must not become an upload outage.

```go
package avatar

import (
	"context"
	"fmt"
	"io"
	"time"
)

type ObjectInfo struct {
	Size        int64
	ContentType string
}

type ObjectStore interface {
	Put(context.Context, string, io.Reader, string) error
	Stat(context.Context, string) (ObjectInfo, error)
}

type PointerStore interface {
	Swap(context.Context, string, string, string) (bool, error)
}

type Collector interface {
	RetainUntil(context.Context, string, time.Time) error
}

type Service struct {
	objects ObjectStore
	pointers PointerStore
	collector Collector
	now func() time.Time
}

func (s *Service) Publish(ctx context.Context, userID, oldKey, newKey, contentType string, body io.Reader, expectedBytes int64) error {
	if err := s.objects.Put(ctx, newKey, body, contentType); err != nil {
		return fmt.Errorf("write candidate: %w", err)
	}
	info, err := s.objects.Stat(ctx, newKey)
	if err != nil || info.Size != expectedBytes || info.ContentType != contentType {
		return fmt.Errorf("validate candidate")
	}
	swapped, err := s.pointers.Swap(ctx, userID, oldKey, newKey)
	if err != nil || !swapped {
		return fmt.Errorf("publish pointer")
	}
	if oldKey != "" {
		return s.collector.RetainUntil(ctx, oldKey, s.now().Add(7*24*time.Hour))
	}
	return nil
}
```

The collector must be idempotent and re-check that the key is not referenced before deleting it. A queue message proves that deletion was requested; it does not prove the object is still inactive.

Make rollback an authorized pointer change with an audit record, then apply the same retention rule to the object it replaces. Test it from the user-visible read path, not just from a successful database update.

I once saw a `429` during a batch refresh; retries eventually succeeded, while p99 latency crossed the upload SLO. Count attempts separately from final outcomes, graph throttled responses even when recovered, and bound retry time by the caller deadline. Retries are part of capacity planning.

Run three failure tests: candidate validation failure leaves the active pointer unchanged; concurrent swaps allow only the expected winner; collection after rollback never removes the restored or active object. For the race test, start two uploads from the same `expectedOldKey`, delay their writes differently, and assert that one compare-and-swap succeeds while the other leaves its candidate unreferenced. For the collector test, enqueue deletion, restore the old key before the retention deadline, and verify that the worker re-reads the pointer before issuing a delete. Record pointer-to-object integrity, candidate age, collector lag, retained bytes, and rollback success.

Practice it.

Multipart uploads add one more cleanup obligation. AWS documents that uploaded parts continue to incur storage charges until a multipart upload is completed or aborted, so give abandoned uploads an owner and an abort policy even if avatars are currently small.

Stick with overwrite-in-place when the file is genuinely disposable and recovery has no product value. For recoverable user content, the small pointer-and-retention model buys a clear rollback story and a cleaner on-call handoff.

## References

- AWS S3, “Multipart upload overview”: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- AWS S3 pricing: https://aws.amazon.com/s3/pricing/
