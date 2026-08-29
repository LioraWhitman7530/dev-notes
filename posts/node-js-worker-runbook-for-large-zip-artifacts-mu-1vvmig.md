# Node.js Worker Runbook for Large ZIP Artifacts: Multipart Storage and Expiring Retrieval

Large export jobs need two separate contracts: durable bytes and temporary access. Short answer: let the worker stream the ZIP into numbered multipart parts, persist the upload state after each acknowledged part, complete or abort the upload explicitly, and issue a short-lived signed download link only after the object and its retention decision are recorded.

That sequence matters most when the archive is a training artifact rather than a disposable report. A reproducible retention policy has to answer which run produced the bytes, how long they may exist, and what happens to an interrupted upload. The file format is the easy part.

## The operational constraint: deletion must be reproducible

Start with a job record, not with a bucket call. Give each export an immutable job identifier, a content description, a retention class, an expiry timestamp, and a storage key derived from the job ID. The retention class should be resolved before the worker starts writing: for example, a short-lived preview and a regulated training snapshot are different policies even when both happen to be ZIP files.

The worker then owns a small state machine:

| State | Durable record | Allowed next actions |
| --- | --- | --- |
| planned | job ID, key, retention expiry | create upload or cancel |
| uploading | upload ID and acknowledged part metadata | upload next part, resume, abort |
| committed | object key, size, checksum if available, expiry | mint link, delete at policy boundary |
| cancelled | cancellation reason and timestamp | verify cleanup |

Do not infer deletion from a queue message that may be delivered twice. The deletion worker should read the job record, compare the current time with the recorded policy boundary, and make deletion idempotent. Keep the record long enough to explain why the object disappeared; deleting the audit row at the same instant as the object makes a later investigation guesswork.

Capacity planning belongs here too. Track bytes in flight, active multipart uploads, part count, worker concurrency, and the oldest nonterminal upload. A retry that starts a second upload doubles the pressure precisely when the queue is already unhealthy. My rule is simple: a worker restart reloads the existing upload ID before it creates anything new. It doesn't get to invent a fresh identity because the process got a fresh PID.

No guessing.

## How should a Node.js worker make large ZIP exports resumable with multipart storage and signed retrieval?

The implementation boundary should be boring. The ZIP producer emits bytes; a storage adapter buffers a bounded part; the job store records the provider's returned part identifier; and a delivery service signs a GET request for the completed key. The buffer is bounded so archive size does not become process memory. A part is eligible for the completion request only after the storage service has acknowledged it.

The example below is Go because the editorial contract for this repository requires all code in Go. Its interfaces are intentionally provider-neutral; the production adapter must map them to the selected object-storage API and preserve the returned part metadata exactly.

```go
package export

import (
	"context"
	"fmt"
)

type Part struct {
	Number int
	ETag   string
}

type Upload struct {
	ID   string
	Key  string
	Parts []Part
}

type Store interface {
	Create(ctx context.Context, key string) (string, error)
	PutPart(ctx context.Context, uploadID string, number int, data []byte) (string, error)
	Complete(ctx context.Context, uploadID string, parts []Part) error
	Abort(ctx context.Context, uploadID string) error
	SignGet(ctx context.Context, key string, expiresInSeconds int64) (string, error)
}

// Run persists each returned ETag in the job store in the real worker.
func Run(ctx context.Context, store Store, jobKey string, parts [][]byte) (string, error) {
	uploadID, err := store.Create(ctx, jobKey)
	if err != nil {
		return "", fmt.Errorf("create upload: %w", err)
	}

	upload := Upload{ID: uploadID, Key: jobKey}
	for number, data := range parts {
		etag, err := store.PutPart(ctx, upload.ID, number+1, data)
		if err != nil {
			_ = store.Abort(ctx, upload.ID)
			return "", fmt.Errorf("put part %d: %w", number+1, err)
		}
		upload.Parts = append(upload.Parts, Part{Number: number + 1, ETag: etag})
		// Persist upload.Parts here before the worker acknowledges the job step.
	}

	if err := store.Complete(ctx, upload.ID, upload.Parts); err != nil {
		_ = store.Abort(ctx, upload.ID)
		return "", fmt.Errorf("complete upload: %w", err)
	}
	return store.SignGet(ctx, upload.Key, 900)
}
```

The snippet omits ZIP creation and persistence details on purpose: those are where application-specific policy lives. In a real worker, the job store write after `PutPart` must be transactional with the worker's progress acknowledgement, and completion must move the job to `committed` only after the completion response is accepted. A signed link returned before that transition is a race, not a delivery feature.

Retries deserve their own review. Retry a failed part with the same part number and replace its recorded ETag with the newly acknowledged value. Treat create and complete as operations that need an idempotency strategy at the job layer; otherwise a process timeout can leave the caller unsure whether it created a second upload or merely lost a response. Picture the recovery path after a worker dies halfway through a 12-part archive: the replacement process reads the job row, finds parts 1 through 6 and their returned identifiers, checks that the upload is still owned by this job, and starts at part 7. If the row is missing, the worker must stop and ask the reconciler to inspect the upload; silently creating another archive is how a routine restart becomes a storage leak. Use exponential backoff with jitter for transient transport failures, bound the total retry time by the export deadline, and abort on cancellation. Never log the signed URL as ordinary request metadata because URLs are bearer capabilities.

## What should verification and rollback prove before a signed download link is issued?

Verification is a sequence of assertions, not a single health check. Confirm that the ZIP producer closed cleanly, every expected part is represented by acknowledged metadata, completion succeeded, the object key belongs to this job, and the retention expiry is present in durable state. Then issue the link with the smallest practical lifetime and record its expiry without recording the secret query string.

Test the failure paths with an object-storage emulator or a test account: interrupt the worker after a part acknowledgement, replay the same job, cancel during upload, lose the completion response, and request a link for a job whose retention class forbids delivery. The expected result is deterministic: resume from recorded parts, abort on cancellation, reconcile completion before signing, and refuse delivery when policy says the artifact is no longer eligible.

Rollback has two clocks. The immediate clock aborts an upload that cannot become a valid object. The policy clock deletes a committed object after its retention boundary. Configure lifecycle cleanup for abandoned multipart material as a backstop, but keep it separate from the application decision; a lifecycle rule cannot know whether a worker is paused, retrying, or permanently cancelled. Alert on age and count of nonterminal uploads, not only on storage bytes.

## Buy, build, or operate the storage boundary

The useful comparison is operational ownership, not a feature-count contest. A managed object store usually removes disk and replication operations but adds provider-specific IAM, egress, and lifecycle semantics. A self-hosted compatible store keeps placement under your control but puts capacity, upgrades, failure domains, and recovery testing on the platform team. A gateway can reduce client coupling, though its own availability and observability become part of the path.

| Boundary | Good fit | Cost or limitation to accept |
| --- | --- | --- |
| Managed object storage | The team wants durable storage without operating disks | Provider IAM, billing, and API semantics become part of the design |
| Self-hosted object storage | Data placement and network control are hard requirements | The team owns capacity forecasts, upgrades, replication, and restore drills |
| Storage gateway | Many workers need one narrow contract | The gateway adds a failure domain and must expose useful metrics |
| Database large-object storage | Objects are small and transactional with metadata | Large streaming exports can increase database backup and recovery pressure |

The catch is that a gateway is not suitable when its abstraction hides the retention controls or conditional-write behavior required by the compliance boundary. Use the storage system that can prove the needed deletion and recovery semantics. Stick with direct provider APIs when a destination-specific feature is part of the contract, and choose self-hosting only when the team is prepared to operate the capacity and failure domains rather than treating them as someone else's problem.

## The runbook decision

Before rollout, set an SLO for export completion and a separate SLO for link issuance; they fail differently. Monitor queue age, worker memory, part retry rate, active upload count, nonterminal upload age, completion latency, deletion lag, and signed-link issuance failures. Alerting on only request latency misses the abandoned-upload leak.

The durable rule is this: generate into a job-scoped key, persist multipart progress, complete or abort explicitly, verify the committed state, and make deletion a recorded policy transition. That design works across storage backends because it keeps the risky decisions in the job state machine instead of scattering them through SDK calls.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
