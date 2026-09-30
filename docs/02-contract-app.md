# 02 — Contract application

The contract application is the user-facing entry point of the pipeline. It is a PHP web app
(`contract-app`) fronted by a separate UI (`contract-frontend`), and it is the **producer** of
the files that the agents later verify.

## Two workloads

| Workload            | Image                                   | Port | Shared mount |
|---------------------|-----------------------------------------|------|:------------:|
| `contract-frontend` | `…/s3files-frontend:latest`             | 3000 | no |
| `contract-app`      | `…/s3files-contract-app:latest`         | 80   | **yes** |

- **`contract-frontend`** is exposed through a `LoadBalancer` Service (a Network Load Balancer),
  giving users an external endpoint. It talks to `contract-app` over the cluster network and
  does not touch the file system.
- **`contract-app`** is the PHP backend. It renders/handles contracts and **writes contract and
  approval files** to the shared volume.

## Where it writes — the env contract

`contract-app` is configured entirely through environment variables that point its output at
the shared mount:

```yaml
env:
- name: BASE_PATH
  value: /app
- name: QUOTE_DATA_DIR
  value: /var/www/html/quotes        # local, ephemeral
- name: CONTRACT_DATA_DIR
  value: /var/www/html/contracts     # local, ephemeral
- name: CONTRACT_OUTPUT_DIR
  value: /var/contracts              # SHARED — visible to all agents
- name: APPROVAL_DATA_DIR
  value: /var/contracts/approvals    # SHARED — approvals written here
```

The important two are `CONTRACT_OUTPUT_DIR` and `APPROVAL_DATA_DIR`: both live under
`/var/contracts`, the S3 Files-backed RWX volume. When the app writes a finished contract there,
two things happen automatically:

1. Every agent pod can immediately read it (shared NFS mount).
2. S3 Files synchronizes it to the Document S3 bucket, which triggers the
   [event pipeline](04-event-pipeline.md).

## Directory structure on the shared volume

The app and agents organize `/var/contracts` into a few well-known subdirectories (present in
both the pod mount and the backing bucket):

```
/var/contracts/
├── approvals/     # approval records written by the app / agents
├── logs/          # workflow logs
├── signed/        # signed / verified contracts
└── unsigned/      # incoming contracts awaiting verification
```

## Health

`contract-app` exposes `/healthz.php` and defines both liveness and readiness probes against
it, so the deployment only reports Ready when the app can serve traffic. Because the shared
volume is mounted at pod start, a broken mount would prevent the pod from becoming Ready — which
makes pod readiness a useful first signal that the storage layer is healthy.

## Service account

`contract-app` runs under the `contract-app` service account (distinct from the agents'
`strands-agent-sa`). It does not need Bedrock or SQS permissions; its only job is to serve the
web app and write files.
