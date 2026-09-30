# 01 — Storage layer: Amazon S3 Files as a shared POSIX file system

This is the heart of the design. Every file-using workload mounts **one** ReadWriteMany
(RWX) volume at `/var/contracts`, and that volume is backed by **Amazon S3 Files** — an S3
bucket surfaced as a POSIX/NFS file system through the **EFS CSI driver**.

## Why S3 Files instead of plain S3 or EBS

| Option | Shared across pods? | POSIX semantics? | Backed by S3 object storage? |
|--------|:-------------------:|:----------------:|:----------------------------:|
| EBS (block)         | No (RWO, one node)  | Yes | No |
| Plain S3 SDK access | Yes                 | No (object API)  | Yes |
| **S3 Files (this)** | **Yes (RWX)**       | **Yes (NFS)**    | **Yes (synchronized)** |

The application and the agents were written to read and write ordinary files
(`open()`/`read()`/`write()`), so they need POSIX semantics. They also need to *share* those
files across many pods on different nodes, which rules out block storage. S3 Files gives both:
a mountable NFS file system whose contents are transparently synchronized to a backing S3
bucket, so the same bytes are reachable as files (in-cluster) and as objects (for the event
pipeline).

## Components

### StorageClass — `k8s/storage/storageclass.yaml`

```yaml
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: s3files-ap        # provision via an S3 Files access point
  fileSystemId: fs-0fdd210641db7f124  # the S3 Files file system
  basePath: /contracts                # access-point root inside the file system
  directoryPerms: "777"
  uid: "33"                           # www-data — matches the app/agent runtime user
  gid: "33"
```

Key points:
- `provisioner: efs.csi.aws.com` — S3 Files is mounted through the EFS CSI driver.
- `uid/gid = 33` (`www-data`) matches the containers' runtime user so files are readable and
  writable without permission juggling. The agent deployments set
  `securityContext.runAsUser/runAsGroup/fsGroup = 33` to line up with this.
- `basePath: /contracts` roots the access point at the `contracts/` prefix in the S3 bucket.

### PersistentVolumeClaim — `k8s/storage/pvc.yaml`

```yaml
accessModes: [ ReadWriteMany ]   # RWX — the whole point; many pods, many nodes
resources:
  requests:
    storage: 1200Gi
storageClassName: s3files-sc
```

`ReadWriteMany` is what allows `contract-app`, `supervisor`, `validator`, `risk-assessor`, and
`benchmark` to all mount the *same* volume simultaneously. The `1200Gi` request is a logical
size for the claim; S3 Files itself is elastic.

## How a pod mounts it

Each of the five file-using deployments declares the volume and mounts it at `/var/contracts`:

```yaml
volumes:
- name: s3-contracts
  persistentVolumeClaim:
    claimName: s3-contracts-pvc
containers:
- name: <app-or-agent>
  volumeMounts:
  - name: s3-contracts
    mountPath: /var/contracts
```

Inside the pod this shows up as an `nfs4` mount:

```
127.0.0.1:/ on /var/contracts type nfs4 (rw,relatime,vers=4.2,...)
```

The `127.0.0.1` address is the EFS CSI driver's local NFS proxy; traffic egresses to the S3
Files **mount target** ENI in the pod's Availability Zone over NFS port 2049.

## The network requirement (this is where setups usually break)

A mount target is an ENI in a specific subnet, protected by a security group. For a pod to
reach it:

1. The mount targets must live in **the cluster's VPC**, one per Availability Zone the cluster
   uses.
2. Each mount target's security group must allow inbound **TCP 2049** from the EKS nodes. The
   simplest correct choice is the **cluster security group**, which already permits all traffic
   among cluster members.

If mount targets are created in the wrong VPC (a common console default is the *default* VPC),
you cannot fix it by editing security groups — SGs are VPC-scoped, and there is no network path
between two unpeered VPCs. The mount targets must be **recreated** in the cluster VPC. See
[`05-runbook.md`](05-runbook.md) for the exact remediation performed in this project.

## Data synchronization (to the backing bucket)

Files written under `/var/contracts` are asynchronously written back to the backing S3 bucket
(observed lag ~1 minute), and objects created in the bucket are imported into the file system
according to the file system's synchronization rules (e.g. `ON_DIRECTORY_FIRST_ACCESS` for the
`contracts/` prefix). That two-way sync is what connects this storage layer to the
[event pipeline](04-event-pipeline.md).
