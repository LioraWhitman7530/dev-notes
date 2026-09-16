# Depot Training Records: Browser PDF/DOCX Storage in Express for Private User Files

Short answer: keep the Express service on the control plane, send document bytes directly from the browser to private object storage, and make retention a database-backed policy that is enforced again by a storage lifecycle rule. For logistics training artifacts, large-file throughput is the deciding constraint; a polished upload button is not.

The bounded scenario is a driver or warehouse-operator training record: a PDF handbook, a DOCX assessment, or a revision bundle that must remain private and reproducible after the person who uploaded it changes teams. I start with the failure path because a 413 at the application boundary, a browser that cannot complete a cross-origin request, and an object that expires before its metadata row are three different incidents. Treating them as one “upload failed” metric makes capacity planning fiction.

The invariant is simple: the application owns identity, object naming, retention intent, and state; storage owns bytes and transfer. A short-lived upload capability crosses that boundary. A separate authorization check issues a download capability later. The record becomes `ready` only after the server has reconciled the expected key, size, checksum policy, and retention class.

That sequence is the part worth keeping when the storage backend changes. The rest is implementation detail.

Keep the key immutable.

## The transfer starts with a training-record ledger

Training material is easy to overwrite accidentally. A revised route-safety PDF can have the same human filename as last month's file, while an audit still needs to establish which revision was assigned to which depot and when its retention clock started. The object key therefore needs an immutable identifier, not `training/current.pdf`; the human filename belongs in metadata.

I use explicit states such as `pending`, `uploaded`, `validated`, `ready`, `expired`, and `rejected`. `uploaded` means bytes were accepted. It does not mean that the file is a valid PDF, a valid DOCX, or approved training content. A validation worker can inspect the object after upload, and the application can then record the validation result without granting a download link early.

Retention is an input to the state transition, not a cleanup afterthought. Store the policy version, effective timestamp, tenant or depot scope, and intended expiry with the document record. When a policy changes, the system can explain which rule produced an expiry date instead of reconstructing it from a filename or a mutable bucket setting.

This is also where compliance language needs discipline. A framework such as FedRAMP is an authorization program, not a magic property inherited by every bucket or upload flow. Map the required controls to the account, region, encryption, access, logging, and deletion evidence you actually operate.

## How can Express make direct browser upload of user documents with presigned targets fit retention policy?

For ordinary PDFs and DOCX files, Express authenticates the user, validates the declared size and retention class, creates an unguessable key, and returns a time-limited upload target. The browser transfers the bytes without routing the file through application memory. The browser then calls a completion endpoint; that endpoint is idempotent and checks the object before advancing the document state. I would trace one upload across these boundaries with a request ID and an object key digest, recording the policy version and the terminal state rather than logging a signed URL. That gives the on-call engineer enough evidence to distinguish a rejected policy from a transfer that reached storage but lost its completion response. The distinction matters during a depot-wide training refresh, when dozens of people may retry the same handbook at once: blindly accepting every retry can create duplicate records, while blindly rejecting a retry can strand a valid object outside the catalog. The completion operation should therefore reconcile by the client operation ID and expected immutable key, then return the already-known state when the same request is repeated.

For files large enough that a retry would waste a meaningful part of the upload SLO, use multipart transfer. Persist the upload identifier and completed parts, cap part concurrency, and retry one failed part rather than restarting the artifact. The exact part size is workload-dependent — I’m not sure what yours should be until the file-size distribution and user network telemetry exist — so measure completion latency, retry volume, and abandoned sessions before setting it.

Here is a deliberately provider-neutral probe for an already-issued upload target. It uses Go because the operational test should exercise the transfer path without coupling the article to a Node.js SDK; the production browser still needs a CORS check from every supported origin.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage: upload-probe <upload-target> <file>")
		os.Exit(2)
	}

	for attempt := 0; attempt < 4; attempt++ {
		file, err := os.Open(os.Args[2])
		if err != nil {
			panic(err)
		}

		req, err := http.NewRequest(http.MethodPut, os.Args[1], file)
		if err != nil {
			file.Close()
			panic(err)
		}
		resp, err := http.DefaultClient.Do(req)
		file.Close()
		if err != nil {
			panic(err)
		}

		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println("upload accepted")
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			panic(fmt.Sprintf("upload failed: status=%d body=%s", resp.StatusCode, strings.TrimSpace(string(body))))
		}
		time.Sleep(time.Second << attempt)
	}
}
```

The code is only a transfer check. It does not decide whether a document may be read, and it does not turn a successful byte transfer into an approved record. The control plane should also expose separate counters for target issuance, completion reconciliation, validation, expiry, and deletion. Those dimensions make an SLO actionable: a slow upload is a transfer problem, while a high `pending` count may be a worker or policy problem.

## What must the storage boundary prove after a provider change?

I compare managed storage and self-hosted storage against the work the platform team must own over the retention lifetime, not against the demo-day upload speed. Direct browser transfer reduces Express bandwidth, but it moves policy into CORS configuration, signed-request rules, lifecycle configuration, and browser observability. Self-hosting can satisfy placement requirements while making capacity, replication, upgrades, and recovery part of the pager rotation.

| Decision area | Managed object storage | Self-hosted object storage |
| --- | --- | --- |
| Large-file throughput | Provider capacity is available, but browser policy and client networks still shape the result | The team must provision disks, network, and headroom for peak training seasons |
| Retention enforcement | Lifecycle rules can remove expired objects; application records still need reconciliation | Deletion jobs, replication, and evidence are the team's responsibility |
| Control and placement | Provider account, region, and contract constrain the design | Placement and implementation are controlled locally, with higher operating cost |
| Incident ownership | The team owns the integration and user-facing SLO | The team also owns the storage service and its recovery path |
| Lock-in | Provider-specific signing and metadata semantics can leak into code | Operational dependence shifts from a vendor API to internal expertise |

The least complex choice is the one that keeps the control-plane contract stable while making the byte path observable. Run a load test with the real artifact distribution, test expiry against a clock you control, and rehearse a lost completion response. A throughput number without these tests is a sales metric, not capacity evidence.

## When should a depot keep the byte path inside Express?

The catch is that private storage is not a permanent public download service. It is unsuitable when a document must be anonymously cacheable, when a long-lived URL is a hard requirement, or when governance requires every byte to traverse an inspection gateway. In those cases, keep authorization and inspection in a controlled service path, accept the extra bandwidth cost, and set the Express body limits and backpressure behavior explicitly.

It is also a poor fit for a team that cannot operate orphan cleanup and retention reconciliation. An expired database row does not prove that the object disappeared, and a lifecycle rule does not explain why a particular training revision was deleted. Stick with a directly managed storage service when you need provider-specific object lock, versioning, replication, or administrative workflows that the application contract cannot express.

Three words matter here: measure before committing. My first test is a 413, a cancelled multipart session, and a download authorization check under the same retention policy; those failures expose the boundary faster than a happy-path upload.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [FedRAMP: Federal Risk and Authorization Management Program](https://www.fedramp.gov/)
