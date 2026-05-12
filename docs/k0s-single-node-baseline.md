# k0s Single-Node Baseline (M1)

## Purpose

Define a reproducible baseline for a single-node k0s cluster (controller + worker) that is suitable for the reference architecture pilot.

## Scope

- Single Linux host running k0s in single-node mode.
- Stable-channel k0s install and verification.
- Baseline readiness checks for API, scheduling, storage, and registry reachability assumptions.

## Non-goals

- Multi-node HA control plane.
- Service-level tuning and app-specific add-ons.
- Multi-cluster bootstrap.

## Baseline Bootstrap Sequence

1. Install k0s from the current stable release.
2. Install k0s as a controller with worker enabled on the same node.
3. Start k0s and wait for node readiness.
4. Export kubeconfig for operator access.
5. Verify control-plane and workload scheduling paths.

Example bootstrap (single node):

```bash
curl -sSLf https://get.k0s.sh | sudo sh
sudo k0s install controller --enable-worker
sudo k0s start

sudo k0s status
sudo k0s kubectl get nodes -o wide

sudo mkdir -p /root/.kube
sudo k0s kubeconfig admin > /root/.kube/config
```

## Readiness Checks

Run these checks after bootstrap and after each stable upgrade.

```bash
# API health
sudo k0s kubectl get --raw='/readyz?verbose'

# Node scheduling readiness
sudo k0s kubectl get nodes

# Core system health
sudo k0s kubectl -n kube-system get pods

# Storage baseline
sudo k0s kubectl get storageclass

# Registry reachability assumptions (DNS + HTTPS)
curl -I https://git.sansnom.uk
```

Expected outcomes:

- `readyz` returns ok status lines.
- Node shows `Ready`.
- `kube-system` pods are Running/Completed with no persistent crash loops.
- At least one default StorageClass is present.
- Registry endpoint is reachable over HTTPS.

## Stable-Track Upgrade Posture

- Track current stable k0s release only.
- Validate in place on the single-node baseline before any wider adoption.
- Re-run the full readiness checks after upgrade.

Upgrade flow:

```bash
curl -sSLf https://get.k0s.sh | sudo sh
sudo k0s stop
sudo k0s start

sudo k0s version
sudo k0s kubectl get nodes
sudo k0s kubectl get --raw='/readyz?verbose'
```

## Evidence Format

Record command-level evidence in PR checklist table rows:

- Command executed
- Expected outcome
- Actual outcome (concise)
- PASS/FAIL

For DG-1 approval, include:

- Bootstrap command transcript summary
- Readiness check results
- Stable-upgrade verification note
