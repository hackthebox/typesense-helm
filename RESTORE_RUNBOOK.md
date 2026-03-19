# Typesense Disaster Recovery Runbook

## Overview

This runbook covers restoring a Typesense cluster from a restic snapshot stored in S3 in the event of data corruption or data loss across all nodes.

Backups are taken daily by a cron sidecar running on `pod-0` only. Snapshots are uploaded to S3 via restic.

---

## Prerequisites

- `kubectl` access to the target cluster
- AWS CLI access (for S3 verification), or equivalent access to your restic backend
- The `RESTIC_REPOSITORY` and `RESTIC_PASSWORD` values used by the backup sidecar

---

## Step 1 -- Identify the snapshot to restore

List available snapshots from the restore pod or directly via AWS CLI:

```bash
aws s3 ls s3://<bucket-name>/snapshots/ --profile <profile>
```

Or spin up a temporary restic pod (see Step 3) and run:

```bash
restic snapshots
```

Note the snapshot ID you want to restore (typically `latest`).

---

## Step 2 -- Scale down the StatefulSet

```bash
kubectl scale statefulset <release-name> -n <namespace> --replicas=0
```

Pods have a `terminationGracePeriodSeconds: 300` (recommended by Typesense for graceful shutdown). If running Istio and `terminationDrainDuration` is not configured, the sidecar may hold the pod alive for the full 300s. To avoid this, set the following annotation in the HelmRelease values:

```yaml
podAnnotations:
  proxy.istio.io/config: '{"terminationDrainDuration": "5s"}'
```

If pods are still stuck terminating during a DR scenario, force delete them:

```bash
kubectl delete pod <release-name>-0 <release-name>-1 <release-name>-2 -n <namespace> --force --grace-period=0
```

Verify no pods are running:

```bash
kubectl get pods -n <namespace>
# Expected: No resources found
```

---

## Step 3 -- Wipe all PVCs

Data corruption is assumed across all nodes, so all PVCs must be wiped:

```bash
for i in 0 1 2; do
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: wipe-${i}
  namespace: <namespace>
spec:
  restartPolicy: Never
  containers:
    - name: wipe
      image: busybox
      command: ["sh", "-c", "rm -rf /data/* && echo 'PVC ${i} wiped'"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: <release-name>-data-<release-name>-${i}
EOF
done
```

> **Note:** If your namespace has Istio sidecar injection enabled, add the annotation `sidecar.istio.io/inject: "false"` to the pod metadata.

Wait for completion and verify:

```bash
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/wipe-0 pod/wipe-1 pod/wipe-2 -n <namespace> --timeout=60s
kubectl logs -n <namespace> wipe-0 && kubectl logs -n <namespace> wipe-1 && kubectl logs -n <namespace> wipe-2
kubectl delete pod wipe-0 wipe-1 wipe-2 -n <namespace>
```

---

## Step 4 -- Restore snapshot to pod-0's PVC

Only pod-0's PVC needs the snapshot. The other nodes will sync from the Raft leader after startup.

Create the restore pod using the service account configured for your backup credentials. Run as UID 10000 (same as the Typesense process) to avoid xattr/lchown errors during restore:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restic-restore
  namespace: <namespace>
spec:
  serviceAccountName: <release-name>
  restartPolicy: Never
  securityContext:
    runAsUser: 10000
    runAsGroup: 3000
  containers:
    - name: restic
      image: ghcr.io/restic/restic:0.18.1
      command: ["sh", "-c", "sleep 600"]
      env:
        - name: RESTIC_REPOSITORY
          value: "<your-restic-repository>"
        - name: RESTIC_PASSWORD
          value: "<your-restic-password>"
        - name: RESTIC_CACHE_DIR
          value: /tmp/.cache/restic
      volumeMounts:
        - name: data
          mountPath: /usr/share/typesense/data
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: <release-name>-data-<release-name>-0
    - name: tmp
      emptyDir: {}
EOF
```

Wait for the pod to be ready:

```bash
kubectl wait --for=condition=Ready pod/restic-restore -n <namespace> --timeout=60s
```

Verify snapshots are accessible:

```bash
kubectl exec -n <namespace> restic-restore -- restic snapshots
```

Run the restore:

```bash
kubectl exec -n <namespace> restic-restore -- restic restore latest --target /
```

> **Note:** SELinux xattr errors (`xattr.LSet security.selinux: operation not permitted`) may appear but are non-fatal. The data is restored correctly despite these errors.

Copy the snapshot contents into the data directory root (Typesense reads from the data dir, not the snapshots subdir):

```bash
SNAPSHOT_DIR=$(kubectl exec -n <namespace> restic-restore -- sh -c "ls /usr/share/typesense/data/snapshots/")
kubectl exec -n <namespace> restic-restore -- sh -c "cp -r /usr/share/typesense/data/snapshots/${SNAPSHOT_DIR}/. /usr/share/typesense/data/"
```

Verify the data directory:

```bash
kubectl exec -n <namespace> restic-restore -- ls /usr/share/typesense/data/
# Expected: snapshots  state
```

Clean up the restore pod:

```bash
kubectl delete pod restic-restore -n <namespace>
```

---

## Step 5 -- Scale back up

```bash
kubectl scale statefulset <release-name> -n <namespace> --replicas=3
```

Pod-0 may crash-loop 1-2 times on startup because it tries to resolve the other peers before their DNS entries are registered. This is expected and self-heals within ~2 minutes once all pods are running.

Monitor the rollout:

```bash
kubectl get pods -n <namespace> -w
```

---

## Step 6 -- Validate

**Health check on all nodes:**

```bash
for pod in <release-name>-0 <release-name>-1 <release-name>-2; do
  echo -n "$pod: "
  kubectl exec -n <namespace> $pod -c typesense -- wget -q -O - "http://localhost:8108/health"
  echo
done
# Expected: {"ok":true} on all nodes
```

**Cluster state:**

```bash
for pod in <release-name>-0 <release-name>-1 <release-name>-2; do
  APIKEY=$(kubectl get secret <release-name>-secret -n <namespace> -o jsonpath='{.data.TYPESENSE_API_KEY}' | base64 -d)
  echo -n "$pod: "
  kubectl exec -n <namespace> $pod -c typesense -- \
    wget -q -O - --header="X-TYPESENSE-API-KEY: ${APIKEY}" "http://localhost:8108/debug"
  echo
done
# state=1 means LEADER, state=4 means FOLLOWER
# Expected: exactly one node with state=1, two nodes with state=4
```

**Collections and data:**

```bash
APIKEY=$(kubectl get secret <release-name>-secret -n <namespace> -o jsonpath='{.data.TYPESENSE_API_KEY}' | base64 -d)
kubectl exec -n <namespace> <release-name>-0 -c typesense -- \
  wget -q -O - --header="X-TYPESENSE-API-KEY: ${APIKEY}" "http://localhost:8108/collections"
# Verify collection names and num_documents match expected values
```

---

## Lessons Learned

| Finding | Detail |
|---|---|
| Pods stuck terminating | Istio sidecar holds pod for full 300s if `terminationDrainDuration` is not set. Use `--force --grace-period=0` in DR scenarios as a last resort. |
| Pod-0 crash-loops on startup | Expected race condition when pod-0 starts before peers are DNS-registered. Self-heals. |
| Only pod-0 needs the snapshot | Other nodes sync via Raft after the cluster forms. |
| Any node can become leader | After restore, Raft elects any node as leader, not necessarily pod-0. |
| SELinux xattr errors on restore | Non-fatal. Data is restored correctly. |
| state=1 is LEADER, state=4 is FOLLOWER | Counter-intuitive but confirmed from Typesense docs. |
| Run restore pod as UID 10000 | Avoids xattr/lchown errors. Matches Typesense container UID. |
