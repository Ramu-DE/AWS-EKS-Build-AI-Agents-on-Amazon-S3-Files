# 04 — Event pipeline: S3 → EventBridge → SQS

The event pipeline is what turns a passive file write into an active verification job. It
connects the [storage layer](01-storage-layer.md) to the
[agentic verification system](03-agentic-verification.md) without any polling of the file
system itself.

## The chain

```
/var/contracts/signed/ (S3 Files)  ── synchronized ──►  Document S3 Bucket (contracts/signed/)
                                                                │
                                                                │  s3:ObjectCreated (signed/ prefix)
                                                                ▼
                                                         EventBridge rule
                                                                │  Send Message
                                                                ▼
                                                            SQS queue
                                                                │  Polls (every 5s)
                                                                ▼
                                                         supervisor (Orchestrator)
```

1. **File write → S3 object.** The `contract-app` writes PDFs into subdirectories of
   `/var/contracts` — `quotes/`, `unsigned/`, and `signed/`. S3 Files synchronizes them to the
   backing **Document S3 bucket** (`s3files-contracts-<account>-<region>`, under the
   `contracts/` prefix). Observed write-back lag is roughly one minute.
2. **S3 → EventBridge.** The bucket emits `s3:ObjectCreated` events. An **EventBridge rule**
   matches objects specifically under the **`contracts/signed/` prefix** — i.e. it fires only
   when a *signed* contract lands, not on every write. (The bucket has versioning and
   EventBridge notifications enabled, both required by S3 Files.)
3. **EventBridge → SQS.** The rule's target sends a message to the **SQS queue**
   (`s3files-contract-validation`). SQS decouples event production from consumption and provides
   retries / visibility timeouts.
4. **SQS → supervisor.** The `supervisor` agent long-polls the queue
   (`SQS_QUEUE_URL`, `POLL_INTERVAL=5`), reads the referenced signed contract from the shared
   mount, orchestrates the `validator` and `risk-assessor` (A2A), and writes the approval result
   back to `approvals/` on the same volume.

## Why this shape

- **Decoupling.** The web app never calls the agents directly. It just writes a file; the
  pipeline discovers it. Producers and consumers scale and fail independently.
- **Durability & retries.** SQS holds work until an agent successfully processes it. If the
  supervisor restarts, unacknowledged messages become visible again.
- **No file-system polling.** Nothing has to `ls` the volume in a loop. Object-created events
  drive the workflow, which is cheaper and lower-latency at scale.

## Synchronization rules (file system side)

The S3 Files file system defines how objects and files stay in sync. In this project the
import rules are approximately:

| Prefix       | Trigger                       | Notes |
|--------------|-------------------------------|-------|
| `contracts/` | `ON_DIRECTORY_FIRST_ACCESS`   | contract files imported on first access |
| `logs/`      | `ON_FILE_ACCESS`              | logs imported per-file |
| (root)       | `ON_DIRECTORY_FIRST_ACCESS`   | small objects |

Expiration is configured to expire cached data 7 days after last access. These rules govern how
data written directly to the bucket (or by another producer) becomes visible inside
`/var/contracts`, and vice versa.

## IAM

The `supervisor` reads from SQS using permissions granted through **IRSA** on the
`strands-agent-sa` service account. The EventBridge rule and its SQS target are provisioned
outside the cluster (as part of the file system / event infrastructure) and are referenced by
the supervisor only via the `SQS_QUEUE_URL` environment variable.

## Verifying the round-trip

You can confirm the storage→S3 half of the pipeline directly:

```bash
BUCKET=s3files-contracts-198645490240-us-west-2
ca=$(kubectl -n contracts get pod -l app=contract-app -o jsonpath='{.items[0].metadata.name}')

# write via the file system
kubectl -n contracts exec "$ca" -- sh -c 'echo hi > /var/contracts/roundtrip.txt'

# appears in the bucket within ~1 minute
aws s3 ls s3://$BUCKET/contracts/roundtrip.txt

# cleanup
kubectl -n contracts exec "$ca" -- sh -c 'rm -f /var/contracts/roundtrip.txt'
aws s3 rm s3://$BUCKET/contracts/roundtrip.txt
```

An object created this way is exactly what triggers the EventBridge → SQS → supervisor chain in
the full system.
