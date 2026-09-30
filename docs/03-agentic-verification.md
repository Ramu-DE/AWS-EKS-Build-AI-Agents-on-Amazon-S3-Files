# 03 — Agentic verification system

Once a contract lands on the shared volume and the [event pipeline](04-event-pipeline.md)
signals new work, the **agentic verification system** takes over. It is a set of cooperating
agents that read the contract from `/var/contracts`, reason about it with **Amazon Bedrock**,
and write results back to the shared volume.

## The agents

| Agent           | Image                              | Port | Role |
|-----------------|------------------------------------|------|------|
| `supervisor`    | `…/s3files-supervisor:latest`      | —    | **Orchestrator** — polls SQS, coordinates the others |
| `validator`     | `…/s3files-validator:latest`       | 8001 | Validates contract content |
| `risk-assessor` | `…/s3files-risk-assessor:latest`   | 8002 | Scores contract risk |
| `benchmark`     | `…/s3files-benchmark:latest`       | 8003 | Benchmarking agent |

All four mount the shared `/var/contracts` volume and run as uid/gid `33` to match the file
ownership set by the StorageClass.

## Orchestration model (A2A)

The `supervisor` is the **Orchestrator** in the "after" architecture diagram. It:

1. **Polls the SQS queue** (`SQS_QUEUE_URL`) every `POLL_INTERVAL` seconds (5s) for
   object-created events describing a new contract file.
2. Reads the contract from `/var/contracts`.
3. Calls the specialized agents **agent-to-agent (A2A)** over HTTP using their in-cluster
   service URLs, then aggregates their outputs.

The supervisor discovers the other agents purely through environment variables — no hardcoded
addresses:

```yaml
env:
- name: SQS_QUEUE_URL
  value: https://sqs.us-west-2.amazonaws.com/198645490240/s3files-contract-validation
- name: AWS_REGION
  value: us-west-2
- name: AGENT_MODEL
  value: us.amazon.nova-pro-v1:0            # Bedrock model used for inference
- name: VALIDATOR_URL
  value: http://validator-svc.contracts.svc.cluster.local:8001
- name: RISK_ASSESSOR_URL
  value: http://risk-assessor-svc.contracts.svc.cluster.local:8002
- name: BENCHMARK_URL
  value: http://benchmark-svc.contracts.svc.cluster.local:8003
- name: WORKFLOW_STREAMER_URL
  value: http://workflow-streamer.contracts.svc.cluster.local:8001
- name: POLL_INTERVAL
  value: "5"
```

## Inference — Amazon Bedrock

`validator` and `risk-assessor` (and the orchestrator) call **Amazon Bedrock** with the model
in `AGENT_MODEL` (`us.amazon.nova-pro-v1:0`). Bedrock access is granted through **IRSA**: the
agents run under the `strands-agent-sa` service account, which is associated with an IAM role
that permits `bedrock:InvokeModel` (and SQS read for the supervisor).

The validator and risk-assessor expose an agent descriptor at `/.well-known/agent.json`, used
as their readiness probe — the deployment is only Ready once the agent server is up.

## How results flow back

Agents write their outputs back to the shared volume — for example a verified contract to
`contracts/signed/` and an approval record to `approvals/`. Because the volume is the same
physical storage everywhere:

- The `contract-app` UI can display results by reading the same files.
- The `workflow-streamer` streams live status to the UI (see below).
- Anything written under `/var/contracts` is also synchronized back to the Document S3 bucket.

## Live status — workflow-streamer

`workflow-streamer` (port 8001) streams workflow progress to the UI. It is **not** a file-using
component — it receives status updates over HTTP (the agents post to `WORKFLOW_STREAMER_URL`)
and relays them to the frontend. It runs under its own `workflow-streamer` service account and
reads its port from the `workflow-streamer-config` ConfigMap.

## Why the shared file system matters here

No agent ever ships contract *contents* to another agent. The SQS message and the A2A calls
carry only lightweight references ("contract X is ready", "please validate X"). The actual bytes
live once, on `/var/contracts`, and every agent reads them directly. This keeps the messaging
layer thin and avoids copying large documents around the cluster.
