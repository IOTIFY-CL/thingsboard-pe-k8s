# ThingsBoard PE Upgrade Instructions (GCP Monolith)

## Summary
This runbook upgrades ThingsBoard PE from `4.0.1PE` to `4.0.2PE` in the `thingsboard` namespace with a safe sequence:
1. Take a database backup.
2. Export current live Kubernetes manifests (for rollback).
3. Clean exported manifests (remove runtime metadata).
4. Run DB upgrade job via `k8s-upgrade-tb.sh`.
5. Update app images in `tb-node.yml` and redeploy.
6. Verify rollout.
7. Roll back quickly if needed.

## Prerequisites
- You are in repo root: `thingsboard-pe-k8s`.
- Your `kubectl` context points to the correct cluster.
- Namespace is `thingsboard`.
- You have permissions to `get/apply/exec/delete` in namespace.

## 1. Take DB Backup (Required)
Take a Cassandra backup/snapshot before running upgrade.

Example checks before backup:
```bash
kubectl get pods -n thingsboard
kubectl get svc -n thingsboard
```

Use your team-approved Cassandra backup method (snapshot, volume snapshot, or managed backup).
Do not continue without a verified backup.

## 2. Export Live Manifests (Rollback Baseline)
Export current live workload definitions:

```bash
kubectl get statefulset tb-node -n thingsboard -o yaml > gcp/monolith/tb-node-live.yaml
kubectl get deployment tb-web-report -n thingsboard -o yaml > gcp/monolith/tb-web-report-live.yaml
```

Optional: keep services too:
```bash
kubectl get service tb-node -n thingsboard -o yaml > gcp/monolith/tb-node-svc-live.yaml
kubectl get service tb-web-report -n thingsboard -o yaml > gcp/monolith/tb-web-report-svc-live.yaml
```

## 3. Clean Exported Manifests
When using `kubectl get -o yaml`, remove runtime-only fields before storing/applying:
- `metadata.resourceVersion`
- `metadata.uid`
- `metadata.generation`
- `metadata.creationTimestamp`
- `metadata.managedFields`
- `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]`
- `metadata.annotations["deployment.kubernetes.io/revision"]`
- `metadata.annotations["kubectl.kubernetes.io/restartedAt"]` (optional, recommended)
- any `status` block
- nested `creationTimestamp: null`
- StatefulSet PVC template `status`

Validation after cleanup:
```bash
kubectl apply --dry-run=server -f gcp/monolith/tb-node-live.yaml -n thingsboard
kubectl apply --dry-run=server -f gcp/monolith/tb-web-report-live.yaml -n thingsboard
```

## 4. Prepare Upgrade Version in Manifests
Set target version to `4.0.2PE` in these files:
1. `gcp/monolith/database-setup.yml`
2. `gcp/monolith/tb-node.yml`

Update image tags:
- `thingsboard/tb-pe-node:4.0.2PE` in `database-setup.yml`
- `thingsboard/tb-pe-node:4.0.2PE` in `tb-node.yml`
- `thingsboard/tb-pe-web-report:4.0.2PE` in `tb-node.yml`

Preview workload diff:
```bash
kubectl diff -f gcp/monolith/tb-node.yml -n thingsboard
```

## 5. Run DB Upgrade Script
Run from `gcp/monolith`:

```bash
cd gcp/monolith
./k8s-upgrade-tb.sh --fromVersion=4.0.1PE
```

What this does:
- Applies `database-setup.yml`.
- Starts `tb-db-setup` pod.
- Runs upgrade logic with `UPGRADE_TB=true` and `FROM_VERSION`.
- Deletes `tb-db-setup` pod afterward.

## 6. Redeploy Workloads with New Images
Apply updated runtime manifest:

```bash
kubectl apply -f gcp/monolith/tb-node.yml -n thingsboard
```

Kubernetes should roll pods automatically when image/tag changes.
If needed, force restart:

```bash
kubectl rollout restart statefulset/tb-node -n thingsboard
kubectl rollout restart deployment/tb-web-report -n thingsboard
```

## 7. Verify Upgrade
```bash
kubectl rollout status statefulset/tb-node -n thingsboard
kubectl rollout status deployment/tb-web-report -n thingsboard
kubectl get pods -n thingsboard
kubectl get statefulset tb-node -n thingsboard -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get deployment tb-web-report -n thingsboard -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## 8. Rollback Procedure
If app behavior is bad after upgrade:

1. Reapply previous workload manifests:
```bash
kubectl apply -f gcp/monolith/tb-node-live.yaml -n thingsboard
kubectl apply -f gcp/monolith/tb-web-report-live.yaml -n thingsboard
```

2. Wait for rollout:
```bash
kubectl rollout status statefulset/tb-node -n thingsboard
kubectl rollout status deployment/tb-web-report -n thingsboard
```

3. If rollback needs DB restore, restore from the backup taken in Step 1 according to your Cassandra recovery procedure.

## Notes
- `kubectl diff` is read-only. It does not update local files and does not apply changes.
- Prefer keeping `gcp/monolith/tb-node.yml` as your source of truth and syncing cluster changes back to Git-managed manifests.