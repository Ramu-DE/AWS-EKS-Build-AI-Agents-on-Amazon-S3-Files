# 05 — Runbook: mounting the S3 Files file system

This runbook captures the actual setup path used to get `/var/contracts` mounted and live on
the cluster — including the real problem hit along the way (mount targets in the wrong VPC) and
how it was fixed. Commands assume `kubectl` is pointed at the cluster and `S3FILES_FS_ID` holds
the S3 Files file system ID.

```bash
export S3FILES_FS_ID=fs-0fdd210641db7f124   # your S3 Files file system ID
```

---

## 0. Starting state

- The PVC `s3-contracts-pvc` is **Bound** (storage layer applied).
- Deployments are running but do **not** yet mount `/var/contracts`.
- Goal: open the network path from the EKS nodes to the S3 Files mount targets, mount the volume
  on the five file-using deployments, and confirm it is live.

---

## 1. Verify mount targets are available

```bash
aws s3files list-mount-targets \
  --file-system-id "$S3FILES_FS_ID" \
  --query 'mountTargets[].[mountTargetId,availabilityZoneId,status]' \
  --output table
```

Every row must read `available` before continuing.

---

## 2. Open the network path — and the VPC gotcha

The intended step is: attach the **cluster security group** to each mount target so the EKS
nodes can reach it on NFS port 2049.

```bash
CLUSTER_SG=$(aws eks describe-cluster \
  --name workshop-cluster-s3files \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' \
  --output text)
```

### The failure

Running the workshop's `update-mount-target` loop failed on every mount target with:

```
ResourceNotFoundException: You have specified two resources that belong to different networks.
```

### Root cause

The mount targets had been created in the account's **default VPC**, not the cluster VPC:

| Resource                         | VPC                     | CIDR            |
|----------------------------------|-------------------------|-----------------|
| EKS cluster + cluster SG         | `vpc-…cluster` (eksctl) | `192.168.0.0/16`|
| Mount targets (as created)       | `vpc-…default`          | `172.31.0.0/16` |

Security groups are **VPC-scoped**, and there was no peering / transit gateway between the two
VPCs. You cannot attach the cluster SG to a mount target in a different VPC — and even if you
could, the nodes still couldn't route to it. The fix is not an SG edit; the mount targets must
be **recreated in the cluster VPC**.

Confirm the mismatch:

```bash
# cluster VPC
aws eks describe-cluster --name workshop-cluster-s3files \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text

# each mount target's VPC
for MT in $(aws s3files list-mount-targets --file-system-id "$S3FILES_FS_ID" \
    --query 'mountTargets[].mountTargetId' --output text); do
  aws s3files get-mount-target --mount-target-id "$MT" \
    --query '[mountTargetId,vpcId,subnetId]' --output text
done
```

### The fix — recreate mount targets in the cluster VPC

Find the cluster's private subnets (one per AZ the cluster uses):

```bash
CLUSTER_VPC=$(aws eks describe-cluster --name workshop-cluster-s3files \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$CLUSTER_VPC" "Name=tag:Name,Values=*Private*" \
  --query 'Subnets[].[SubnetId,AvailabilityZoneId,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

Delete the misplaced mount targets, wait for them to disappear, then create new ones in the
cluster's private subnets with the cluster SG attached at creation:

```bash
# 1. delete the ones in the wrong VPC
for MT in $(aws s3files list-mount-targets --file-system-id "$S3FILES_FS_ID" \
    --query 'mountTargets[].mountTargetId' --output text); do
  aws s3files delete-mount-target --mount-target-id "$MT"
done

# 2. wait until none remain
while [ "$(aws s3files list-mount-targets --file-system-id "$S3FILES_FS_ID" \
    --query 'length(mountTargets)' --output text)" != "0" ]; do sleep 10; done

# 3. create in the cluster's private subnets (replace with YOUR subnet IDs)
for SUBNET in subnet-AAA subnet-BBB subnet-CCC; do
  aws s3files create-mount-target \
    --file-system-id "$S3FILES_FS_ID" \
    --subnet-id "$SUBNET" \
    --security-groups "$CLUSTER_SG"
done
```

> Note: you get **one mount target per AZ the cluster has subnets in**. In this project the
> cluster spanned three AZs (az2/az3/az4), so there are three mount targets — not four. That is
> correct; pods only schedule onto nodes in those AZs.

Wait for all new mount targets to be `available`, and confirm they are in the cluster VPC with
the cluster SG:

```bash
for MT in $(aws s3files list-mount-targets --file-system-id "$S3FILES_FS_ID" \
    --query 'mountTargets[].mountTargetId' --output text); do
  aws s3files get-mount-target --mount-target-id "$MT" \
    --query '[mountTargetId,vpcId,securityGroups]' --output text
done
```

---

## 3. Attach the shared volume to the five deployments

Only the workloads that actually use `/var/contracts` get the mount — `contract-app`,
`validator`, `risk-assessor`, `supervisor`, `benchmark`. (`contract-frontend` and
`workflow-streamer` do not.)

Applying the manifests in `k8s/base/` already includes the volume + mount. If patching existing
deployments in place instead:

```bash
for DEPLOY in contract-app validator risk-assessor supervisor benchmark; do
  kubectl patch deployment "$DEPLOY" -n contracts --type=json -p='[
    {"op":"add","path":"/spec/template/spec/volumes",
     "value":[{"name":"s3-contracts","persistentVolumeClaim":{"claimName":"s3-contracts-pvc"}}]},
    {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts",
     "value":[{"name":"s3-contracts","mountPath":"/var/contracts"}]}
  ]'
done
```

> How to know which deployments need the mount: grep each running pod for `/var/contracts`
> references in its env and code. Exactly five reference it; the other two don't.

Wait for the rollouts:

```bash
for D in contract-app validator risk-assessor supervisor benchmark; do
  kubectl -n contracts rollout status deploy/$D --timeout=180s
done
```

Pods reaching `Ready` is itself a signal the mount succeeded — a broken mount leaves pods stuck
in `ContainerCreating`.

---

## 4. Confirm the mount is live

### NFS mount on all five

```bash
for d in contract-app validator risk-assessor supervisor benchmark; do
  pod=$(kubectl -n contracts get pod -l app=$d -o jsonpath='{.items[0].metadata.name}')
  echo -n "$d: "
  kubectl -n contracts exec "$pod" -- sh -c 'grep " /var/contracts " /proc/mounts | awk "{print \$3}"'
done
# expect: nfs4 for each
```

### Cross-pod shared read/write

```bash
ca=$(kubectl -n contracts get pod -l app=contract-app -o jsonpath='{.items[0].metadata.name}')
val=$(kubectl -n contracts get pod -l app=validator -o jsonpath='{.items[0].metadata.name}')

kubectl -n contracts exec "$ca"  -- sh -c 'echo hello > /var/contracts/.check'
kubectl -n contracts exec "$val" -- sh -c 'cat /var/contracts/.check'   # -> hello
kubectl -n contracts exec "$ca"  -- sh -c 'rm -f /var/contracts/.check'
```

### S3 round-trip (storage ↔ bucket)

```bash
BUCKET=s3files-contracts-198645490240-us-west-2
kubectl -n contracts exec "$ca" -- sh -c 'echo hi > /var/contracts/roundtrip.txt'
# appears in the bucket within ~1 minute (asynchronous write-back)
aws s3 ls s3://$BUCKET/contracts/roundtrip.txt
kubectl -n contracts exec "$ca" -- sh -c 'rm -f /var/contracts/roundtrip.txt'
aws s3 rm s3://$BUCKET/contracts/roundtrip.txt
```

---

## Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `update-mount-target` → *"two resources that belong to different networks"* | Mount targets in a different VPC than the cluster SG | Recreate mount targets in the **cluster VPC** subnets (§2) |
| Pods stuck in `ContainerCreating`, events show mount timeout | No network path to mount target (SG missing 2049, or wrong VPC) | Attach cluster SG to mount targets; confirm same VPC |
| `Bound` PVC but empty `/var/contracts` | Access point `basePath` / sync rules | Check StorageClass `basePath` and file system sync rules |
| File written in pod not in S3 | Normal async write-back lag | Wait ~1 minute; S3 Files syncs asynchronously |
| Permission denied writing to `/var/contracts` | uid/gid mismatch | StorageClass `uid/gid` and pod `securityContext` should both be `33` |
