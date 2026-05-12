# Developer Workflow: Container-Native Development with OCI and GitOps

## Document Purpose

This document defines a complete developer workflow for building, signing, distributing, and deploying containerised applications using containerd, nerdctl, OCI registries, Kustomize, and Flux. It is intended to be readable by both humans and AI systems as a reference architecture.

## Philosophy

Three sources govern the lifecycle of an application:

| Layer | Role | Backing Store |
|---|---|---|
| **Git** | Source of Intent | What we want — code, Containerfiles, Kustomize definitions, design decisions |
| **OCI Registry** | Source of Truth | What exists — immutable, signed, digest-pinned artifacts |
| **etcd / API Server** | Canonical Running State | What is — the actual state of the cluster, reconciled continuously |

Each layer has a single responsibility and a clear contract with the next. Git never claims to know what is running. The registry never mutates. etcd never pretends to be the source of what should be running — Flux handles that convergence.

---

## 1. Developer Toolchain

### 1.1 Shell Environment

The default developer shell includes three version-managed tool ecosystems:

- **fnm** — Fast Node Manager (Rust-based Node.js version manager)
- **bun** — JavaScript/TypeScript runtime, bundler, and package manager
- **uv** — Fast Python package manager (Rust-based, from Astral)

Installation:

```bash
# fnm
curl -fsSL https://fnm.vercel.app/install | bash

# bun
curl -fsSL https://bun.sh/install | bash

# uv
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Shell configuration (`~/.bashrc`):

```bash
# fnm
eval "$(fnm env)"

# bun
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"

# uv
export PATH="$HOME/.local/bin:$PATH"
```

### 1.2 Node.js via fnm

Use fnm to install and pin Node.js versions per user without touching system packages.

```bash
fnm install 22
fnm use 22
fnm default 22
fnm ls
```

Node versions are stored under `~/.fnm` and do not interfere with system packages.

### 1.3 Kubernetes CLI Tooling (k8s-tools)

Use `k8s-tools` as the default source of stable current Kubernetes CLI tool versions. Reference: `https://github.com/sansnom-co/k8s-tools/blob/main/README.md`.

Included tool packages: `kubectl`, `helm`, `jq`, `skopeo`, `oras`, `cosign`, `flux`.

APT repository setup (Debian/Ubuntu):

```bash
# 1) Add the GPG key
wget -O- https://sansnom-co.github.io/k8s-tools/public_key.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/sansnom-k8s-tools.gpg

# 2a) Add the repository (APT 2.x, traditional format)
echo "deb [signed-by=/usr/share/keyrings/sansnom-k8s-tools.gpg] \
  https://sansnom-co.github.io/k8s-tools stable main" | \
  sudo tee /etc/apt/sources.list.d/sansnom-k8s-tools.list

# 2b) Add the repository (APT 3.0+, deb822 format)
sudo tee /etc/apt/sources.list.d/sansnom-k8s-tools.sources << 'EOF'
Types: deb
URIs: https://sansnom-co.github.io/k8s-tools
Suites: stable
Components: main
Signed-By: /usr/share/keyrings/sansnom-k8s-tools.gpg
EOF

# 3) Install
sudo apt update

# Install all tools at once (recommended)
sudo apt install kube-essential

# Or install individual tools
sudo apt install kubectl
sudo apt install helm jq
```

`kube-essential` is a meta-package that depends on the individual tool packages, so each tool can upgrade independently.

---

## 2. Container Runtime Architecture

### 2.1 Runtime Stack

The container runtime is containerd, not Docker. This aligns with Kubernetes (and k0s) which uses containerd as its default runtime.

```
User CLI
  ├── nerdctl          (dev: build, run, push)
  │     │
  │     └── containerd native API
  │
  └── crictl           (k8s debug/admin)
        │
        └── CRI gRPC API
              │
              ▼
        containerd daemon
        ├── Image store, snapshots, content management
        ├── CRI plugin (built-in, enabled by default)
        │
        └── containerd-shim (per-container supervisor)
              │
              ▼
        runc / crun (OCI low-level runtime)
              │
              ▼
        Linux kernel
        (namespaces, cgroups, seccomp, overlayfs)
```

### 2.2 Layer Definitions

**runc / crun** — Low-level OCI runtimes. They take an OCI bundle (root filesystem + JSON config) and call Linux kernel primitives to create and start a container process. runc is the Go reference implementation; crun is a lighter C alternative from Red Hat. You never interact with these directly.

**containerd** — High-level container daemon. Manages the full container lifecycle: image pulling, storage, snapshotting, networking setup. Calls down to runc/crun to spawn processes. Exposes a gRPC API consumed by higher-level tools. Maintains state and handles restarts.

**containerd-shim** — Per-container supervisor process. Allows containerd to restart without killing running containers.

### 2.3 nerdctl vs crictl

Both talk to containerd but serve different purposes:

**nerdctl** — Developer tool. CLI for containerd with Docker-compatible syntax for building, running, pushing images. Talks to containerd's native API. Delegates builds to BuildKit.

Capabilities: `build`, `push`, `pull`, `run`, `compose`, `tag`, `save`, `load`, `volume`, `network`, `commit`, `login`.

**crictl** — Kubernetes debug/admin tool. Talks to containerd's CRI gRPC API. Views Kubernetes-managed containers and pod sandboxes.

Capabilities: `pods`, `ps`, `info`, `pull` (via CRI), sandbox inspection, runtime status.

Key distinction: containerd uses internal namespaces (not Linux namespaces). Kubernetes containers live in the `k8s.io` namespace; nerdctl defaults to `default`. They don't see each other by default. Cross-namespace inspection:

```bash
sudo nerdctl --namespace k8s.io ps
sudo nerdctl --namespace k8s.io images
```

### 2.4 nerdctl vs podman

Both are CLIs for building and running containers without Docker, with compatible command syntax but different architectures:

- **nerdctl** shares containerd's image store with Kubernetes. Images built locally are immediately available to k0s without export/import. Requires containerd and BuildKit daemons.
- **podman** is fully daemonless with its own storage (via crun/runc directly). Separate image store from Kubernetes. More mature rootless support. Red Hat ecosystem.

For k0s/k0rdent workflows, nerdctl is the natural fit due to shared containerd runtime alignment.

### 2.5 Installation

```bash
# containerd
sudo apt install containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl enable --now containerd

# nerdctl (full: includes CNI plugins and BuildKit)
curl -fsSL https://github.com/containerd/nerdctl/releases/download/v2.0.4/nerdctl-full-2.0.4-linux-amd64.tar.gz | sudo tar -xz -C /usr/local

# Enable BuildKit
sudo systemctl enable --now buildkit

# CRI tools
sudo apt install cri-tools
```

CRI endpoint configuration (`/etc/crictl.yaml`):

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
```

Optional rootless setup:

```bash
containerd-rootless-setuptool.sh install
containerd-rootless-setuptool.sh install-buildkit
```

---

## 3. OCI Registry

### 3.1 Registry: Forgejo

The OCI registry is the built-in package registry in Forgejo at `git.sansnom.uk`. It supports both OCI container images and OCI artifacts natively.

Authentication uses a Forgejo personal access token with `package` scope.

```bash
# Login for container images
nerdctl login git.sansnom.uk

# Login for OCI artifacts
oras login git.sansnom.uk
```

### 3.2 OCI Images

nerdctl and BuildKit produce OCI-format images by default (`application/vnd.oci.image.manifest.v1+json`).

```bash
# Build
nerdctl build -t git.sansnom.uk/martin/myapp:v1.0.0 .

# Push
nerdctl push git.sansnom.uk/martin/myapp:v1.0.0

# Verify format
nerdctl image inspect git.sansnom.uk/martin/myapp:v1.0.0 | grep -i mediatype
```

### 3.3 OCI Artifacts via ORAS

ORAS (OCI Registry As Storage) pushes and pulls arbitrary artifacts — Helm charts, Kustomize manifests, SBOMs, configs — to any OCI-compliant registry.

```bash
# Install
sudo apt install oras

# Push an artifact
oras push git.sansnom.uk/martin/myartifact:v1 \
  ./chart.tgz:application/vnd.cncf.helm.chart.content.v1.tar+gzip

# Pull
oras pull git.sansnom.uk/martin/myartifact:v1

# Attach metadata to an existing image
oras attach git.sansnom.uk/martin/myapp:v1.0.0 \
  ./sbom.json:application/spdx+json
```

---

## 4. Build, Sign, and Publish

### 4.1 SBOM Generation

SBOMs are generated using syft in SPDX JSON format:

```bash
syft git.sansnom.uk/martin/myapp:v1.0.0 -o spdx-json > myapp-v1.0.0.sbom.json
```

### 4.2 Image Signing

Images are signed using cosign with a keypair:

```bash
# One-time keypair generation
cosign generate-key-pair

# Sign after push
cosign sign --key cosign.key git.sansnom.uk/martin/myapp:v1.0.0

# Attach SBOM
cosign attach sbom --sbom myapp-v1.0.0.sbom.json git.sansnom.uk/martin/myapp:v1.0.0

# Verify
cosign verify --key cosign.pub git.sansnom.uk/martin/myapp:v1.0.0
```

### 4.3 Release Script

The full build-sign-publish pipeline:

```bash
#!/usr/bin/env bash
set -euo pipefail

APP="git.sansnom.uk/martin/myapp"
DEPLOY="git.sansnom.uk/martin/myapp-deploy"
VERSION="${1:?Usage: release.sh <version>}"

# Build and push app image
nerdctl build -t "${APP}:${VERSION}" .
nerdctl push "${APP}:${VERSION}"

# Generate SBOM
syft "${APP}:${VERSION}" -o spdx-json > "myapp-${VERSION}.sbom.json"

# Sign image and attach SBOM
cosign sign --key cosign.key "${APP}:${VERSION}"
cosign attach sbom --sbom "myapp-${VERSION}.sbom.json" "${APP}:${VERSION}"

# Get pushed image digest and update prod overlay
DIGEST=$(nerdctl image inspect "${APP}:${VERSION}" --format '{{index .RepoDigests 0}}' | cut -d'@' -f2)
pushd deploy/overlays/prod >/dev/null
kustomize edit set image "${APP}=${APP}:${VERSION}@${DIGEST}"
popd >/dev/null

# Push full kustomize tree as OCI artifact (overlay depends on ../../base)
flux push artifact "oci://${DEPLOY}:${VERSION}" \
  --path=deploy \
  --source="local" \
  --revision="${VERSION}"

# Sign the deploy artifact
cosign sign --key cosign.key "${DEPLOY}:${VERSION}"
```

---

## 5. Deployment with Kustomize

### 5.1 Compose to Kustomize Mapping

| Compose | Kubernetes | Notes |
|---|---|---|
| `services:` | Deployment (stateless) / StatefulSet (stateful) | StatefulSet for databases |
| `image:` | `spec.containers[].image` | Managed via `images:` transformer in overlay |
| `ports:` | Service | ClusterIP for internal, headless for StatefulSets |
| `environment:` | ConfigMap / Secret | Use `secretGenerator` in Kustomize |
| `volumes:` (named) | PersistentVolumeClaim | VolumeClaimTemplate for StatefulSets |
| `depends_on:` | No direct equivalent | Use init containers or readiness probes |
| Different envs | Overlays | `overlays/dev/`, `overlays/prod/` |

### 5.2 Directory Structure

```
deploy/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment-web.yaml
│   ├── statefulset-db.yaml
│   ├── service-web.yaml
│   ├── service-db.yaml
│   └── pvc-web.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    └── prod/
        ├── kustomization.yaml
        └── patches/
```

### 5.3 Base Manifests

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp

resources:
  - namespace.yaml
  - deployment-web.yaml
  - statefulset-db.yaml
  - service-web.yaml
  - service-db.yaml
  - pvc-web.yaml

secretGenerator:
  - name: web-secret
    literals:
      - APP_SECRET=changeme
  - name: db-secret
    literals:
      - POSTGRES_PASSWORD=changeme

commonLabels:
  app.kubernetes.io/name: myapp
```

`base/deployment-web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app.kubernetes.io/component: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: web
  template:
    metadata:
      labels:
        app.kubernetes.io/component: web
    spec:
      containers:
        - name: web
          image: git.sansnom.uk/martin/myapp:latest
          ports:
            - containerPort: 80
          env:
            - name: DATABASE_URL
              value: "postgres://db:5432/myapp"
            - name: APP_SECRET
              valueFrom:
                secretKeyRef:
                  name: web-secret
                  key: APP_SECRET
          volumeMounts:
            - name: app-data
              mountPath: /data
      volumes:
        - name: app-data
          persistentVolumeClaim:
            claimName: web-data
```

`base/statefulset-db.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  labels:
    app.kubernetes.io/component: db
spec:
  serviceName: db
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: db
  template:
    metadata:
      labels:
        app.kubernetes.io/component: db
    spec:
      containers:
        - name: db
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: db-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

`base/service-web.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app.kubernetes.io/component: web
```

`base/service-db.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  ports:
    - port: 5432
  selector:
    app.kubernetes.io/component: db
```

`base/pvc-web.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

### 5.4 Overlays

`overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-dev

resources:
  - ../../base

namePrefix: dev-

images:
  - name: git.sansnom.uk/martin/myapp
    newTag: dev-latest
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-prod

resources:
  - ../../base

namePrefix: prod-

images:
  - name: git.sansnom.uk/martin/myapp
    newTag: v1.0.0
    digest: sha256:abc123...
```

Production overlays pin to digest to ensure the deployed image is exactly the signed artifact.
The digest value (`sha256:...`) is illustrative; always use the actual digest from the pushed image.

### 5.5 When to Write Kustomize

The Kustomize manifests should be created when the application shape stabilises — services, ports, volumes, environment variables, and dependencies are no longer changing frequently.

**During initial development**: Use `nerdctl compose` and the compose file. Do not maintain Kustomize in parallel while the app is still being shaped.

**When the app stabilises**: Translate the compose file to base Kustomize manifests once. This is typically triggered by readiness to deploy to a shared cluster, needing others to run it, or wanting Flux to manage deployment.

**On each release**: Only the overlay changes — specifically the image tag and digest. The base rarely changes after initial creation.

---

## 6. GitOps Deployment with Flux

### 6.1 OCI-Based Deployment

Kustomize manifests are packaged and pushed to the OCI registry alongside the application image. Flux pulls from the registry, not from git.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  url: oci://git.sansnom.uk/martin/myapp-deploy
  ref:
    tag: v1.0.0
  secretRef:
    name: forgejo-registry
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp-prod
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: OCIRepository
    name: myapp
  prune: true
  wait: true
```

Registry credentials (note: `docker-registry` is the Kubernetes secret type name, not a Docker dependency):

```bash
kubectl create secret docker-registry forgejo-registry \
  --docker-server=git.sansnom.uk \
  --docker-username=martin \
  --docker-password=<token> \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 6.2 Signature Verification at Admission

The Sigstore policy-controller ensures only signed images are admitted to the cluster:

```bash
helm install policy-controller sigstore/policy-controller \
  -n cosign-system --create-namespace
```

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: forgejo-registry
spec:
  images:
    - glob: "git.sansnom.uk/**"
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          <cosign.pub contents>
          -----END PUBLIC KEY-----
```

Any pod pulling from `git.sansnom.uk` without a valid cosign signature is rejected.

---

## 7. End-to-End Flow

```
Developer Workstation              Forgejo (git.sansnom.uk)       k0s Production Cluster
────────────────────              ──────────────────────────      ──────────────────────

Git (Intent)
  │
  ├── Code + Containerfile
  ├── Kustomize base + overlays
  │
  ▼
nerdctl build
  │
  ▼
nerdctl push ──────────────────► OCI Registry
                                   ├── App image (signed)
syft (SBOM) ──► cosign attach ──► │   ├── Signature
                                   │   └── SBOM
cosign sign ────────────────────► │
                                   │
kustomize edit set image           │
  │                                │
  ▼                                │
flux push artifact ─────────────► ├── Deploy manifests (signed)
cosign sign ────────────────────► │   └── Signature
                                   │
                                   │           Flux reconciliation
                                   │                  │
                                   ◄──────────────────┘
                                                       │
                                              policy-controller
                                              verifies signatures
                                                       │
                                                       ▼
                                              etcd / API Server
                                              (Canonical Running State)
                                                       │
                                                       ▼
                                                  Pods running ✓
```

### 7.1 Troubleshooting Model

When something is wrong, ask three questions in order:

1. **Is intent correct?** → Check git (code, Containerfile, Kustomize definitions)
2. **Was it built correctly?** → Check the OCI artifact — verify signature, inspect SBOM
3. **Is it running correctly?** → Check the API server / etcd — use `crictl` on the node, `kubectl` from outside

### 7.2 Multi-Cluster Scaling

For k0rdent-managed multi-cluster environments, the same OCI artifacts are consumed by multiple clusters. Each cluster's Flux instance pulls independently from the shared registry. The registry becomes the distribution point without git fan-out complexity.

---

## 8. Tool Reference

| Tool | Purpose | Install |
|---|---|---|
| `fnm` | Node.js version management | `curl -fsSL https://fnm.vercel.app/install \| bash` |
| `bun` | JS/TS runtime and package manager | `curl -fsSL https://bun.sh/install \| bash` |
| `uv` | Python package manager | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| `containerd` | Container runtime daemon | `sudo apt install containerd` |
| `nerdctl` | CLI for containerd (Docker-compatible syntax) | Full tarball from GitHub releases |
| `BuildKit` | Container image builder | Included in nerdctl full tarball |
| `crictl` | Kubernetes CRI debug tool | `sudo apt install cri-tools` |
| `cosign` | Container image signing | `sudo apt install cosign` (from k8s-tools) |
| `syft` | SBOM generation | Installer from anchore/syft |
| `oras` | OCI artifact push/pull | `sudo apt install oras` (from k8s-tools) |
| `flux` | GitOps toolkit CLI | `sudo apt install flux` (from k8s-tools) |
| `kustomize` | Kubernetes manifest management | `kubectl` built-in or standalone binary |
| `helm` | Kubernetes package manager | `sudo apt install helm` (from k8s-tools) |

---

## 9. Key Principles

1. **Git is intent, OCI is truth, etcd is state.** Each layer has a single responsibility.
2. **Everything is an OCI artifact.** App images, deploy manifests, Helm charts, SBOMs — all versioned, signed, and stored in the same registry.
3. **Sign everything.** Both app images and deploy manifests are cosign-signed. Admission policy rejects unsigned artifacts.
4. **Pin to digest in production.** Tags are mutable. Digests are not. Production overlays always reference digests.
5. **Kustomize for your own apps, Helm for third-party.** Kustomize is real YAML with patches. Helm is for distributing reusable charts to others.
6. **Write Kustomize once, update overlays on release.** The base manifests rarely change after the app shape stabilises.
7. **containerd is the shared runtime.** nerdctl for dev, CRI for Kubernetes, same daemon, same image store.
8. **Separate build and deploy lifecycles.** Code commits trigger builds. Deploy artifacts are published when a release is ready. Flux reconciles independently.

---

## Appendix A: Deployment Without Flux

This appendix covers deploying from OCI artifacts directly using `oras`, `cosign`, and `kubectl` — no Flux or GitOps controller required. This is suitable for manual deployments, simple environments, CI/CD pipelines that push directly, or clusters where Flux is not installed.

### A.1 Prerequisites on the Target Machine

The deploying machine (operator workstation or CI runner) needs:

- `oras` — to pull deploy manifests from the OCI registry
- `cosign` — to verify signatures before applying
- `kubectl` — configured with kubeconfig for the target k0s cluster
- `kustomize` — standalone binary or via `kubectl kustomize`

Install the Kubernetes CLI tools from the `k8s-tools` repository setup in Section 1.3 (recommended via `sudo apt install kube-essential`).

### A.2 Pull and Verify

```bash
#!/usr/bin/env bash
set -euo pipefail

APP="git.sansnom.uk/martin/myapp"
DEPLOY="git.sansnom.uk/martin/myapp-deploy"
VERSION="${1:?Usage: deploy.sh <version>}"
WORKDIR=$(mktemp -d)

echo "==> Verifying app image signature"
cosign verify --key cosign.pub "${APP}:${VERSION}"

echo "==> Verifying deploy manifest signature"
cosign verify --key cosign.pub "${DEPLOY}:${VERSION}"

echo "==> Pulling deploy manifests"
cd "${WORKDIR}"
oras pull "${DEPLOY}:${VERSION}"
```

### A.3 Inspect Before Applying

Always review what will be applied:

```bash
echo "==> Rendering manifests"
kubectl kustomize "${WORKDIR}"

# Or diff against what is currently running
kubectl diff -k "${WORKDIR}"
```

`kubectl diff` shows exactly what would change without applying anything. This is the manual equivalent of Flux's dry-run reconciliation.

### A.4 Apply

```bash
echo "==> Applying to cluster"
kubectl apply -k "${WORKDIR}"

echo "==> Waiting for rollout"
kubectl rollout status deployment/prod-web -n myapp-prod --timeout=300s

echo "==> Cleaning up"
rm -rf "${WORKDIR}"
```

### A.5 Registry Credentials on the Cluster

The k0s nodes need credentials to pull the app image from the Forgejo registry. Create an image pull secret (note: `docker-registry` is the Kubernetes secret type name, not a Docker dependency):

```bash
kubectl create secret docker-registry forgejo-pull \
  --docker-server=git.sansnom.uk \
  --docker-username=martin \
  --docker-password=<token> \
  -n myapp-prod
```

Reference it in the deployment either via a patch in the prod overlay or directly in the base:

`overlays/prod/patches/imagepullsecret.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      imagePullSecrets:
        - name: forgejo-pull
```

Add to `overlays/prod/kustomization.yaml`:

```yaml
patches:
  - path: patches/imagepullsecret.yaml
```

### A.6 Full Deploy Script

```bash
#!/usr/bin/env bash
set -euo pipefail

APP="git.sansnom.uk/martin/myapp"
DEPLOY="git.sansnom.uk/martin/myapp-deploy"
VERSION="${1:?Usage: deploy.sh <version>}"
OVERLAY="${2:-prod}"
NAMESPACE="myapp-${OVERLAY}"
WORKDIR=$(mktemp -d)
trap 'rm -rf "$WORKDIR"' EXIT

# Verify signatures
echo "==> Verifying signatures"
cosign verify --key cosign.pub "${APP}:${VERSION}"
cosign verify --key cosign.pub "${DEPLOY}:${VERSION}"

# Pull deploy manifests
echo "==> Pulling deploy manifests (${DEPLOY}:${VERSION})"
cd "${WORKDIR}"
oras pull "${DEPLOY}:${VERSION}"

# Show what will change
echo "==> Diff against running state"
kubectl diff -k "${WORKDIR}" || true

# Prompt for confirmation (set AUTO_APPROVE=true for CI/non-interactive runs)
if [[ "${AUTO_APPROVE:-false}" != "true" ]]; then
  read -rp "Apply to ${NAMESPACE}? [y/N] " confirm
  if [[ "${confirm}" != [yY] ]]; then
    echo "Aborted."
    exit 0
  fi
fi

# Apply
echo "==> Applying"
kubectl apply -k "${WORKDIR}"

# Wait for rollout
echo "==> Waiting for rollout"
kubectl rollout status deployment/prod-web -n "${NAMESPACE}" --timeout=300s

echo "==> Deployed ${VERSION} to ${NAMESPACE}"
```

Usage:

```bash
# Deploy v1.0.0 to prod
./deploy.sh v1.0.0 prod

# Deploy v1.1.0-rc1 to dev
./deploy.sh v1.1.0-rc1 dev
```

### A.7 Rollback

Rollback is deploying a previous version:

```bash
# Roll back to previous known-good version
./deploy.sh v0.9.0 prod
```

Or use kubectl's built-in rollback for the deployment itself:

```bash
# Undo the last rollout (does not change the OCI state)
kubectl rollout undo deployment/prod-web -n myapp-prod
```

Note: `kubectl rollout undo` reverts the cluster state but does not change what's in the registry. The OCI artifact for the current version still exists. This is a quick fix — the next proper deploy should use the script to ensure OCI truth and cluster state are aligned.

### A.8 CI/CD Integration

For a CI pipeline (e.g. Forgejo Actions, Woodpecker, Jenkins), the deploy script becomes a pipeline step:

```yaml
# .forgejo/workflows/deploy.yaml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      version:
        required: true
      environment:
        required: true
        default: prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Install tools
        run: |
          wget -O- https://sansnom-co.github.io/k8s-tools/public_key.asc | \
            gpg --dearmor | sudo tee /usr/share/keyrings/sansnom-k8s-tools.gpg >/dev/null
          echo "deb [signed-by=/usr/share/keyrings/sansnom-k8s-tools.gpg] https://sansnom-co.github.io/k8s-tools stable main" | \
            sudo tee /etc/apt/sources.list.d/sansnom-k8s-tools.list
          sudo apt update
          sudo apt install -y kube-essential

      - name: Login to registry
        run: oras login git.sansnom.uk -u ${{ secrets.REGISTRY_USER }} -p ${{ secrets.REGISTRY_TOKEN }}

      - name: Verify and deploy
        run: |
          cosign verify --key cosign.pub git.sansnom.uk/martin/myapp:${{ inputs.version }}
          cosign verify --key cosign.pub git.sansnom.uk/martin/myapp-deploy:${{ inputs.version }}
          WORKDIR=$(mktemp -d)
          cd "${WORKDIR}"
          oras pull git.sansnom.uk/martin/myapp-deploy:${{ inputs.version }}
          kubectl apply -k .
          kubectl rollout status deployment/prod-web -n myapp-${{ inputs.environment }} --timeout=300s
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

### A.9 Comparison: Flux vs Manual

| Aspect | Flux (Section 6) | Manual (This Appendix) |
|---|---|---|
| Trigger | Automatic on OCI tag change | Explicit script invocation |
| Drift detection | Continuous reconciliation | Only at deploy time via `kubectl diff` |
| Rollback | Revert OCI tag in Flux config | Redeploy previous version or `kubectl rollout undo` |
| Audit trail | Flux events + OCI tags | Script logs + OCI tags |
| Complexity | Flux installed on cluster | Just oras + kubectl |
| Multi-cluster | Each cluster reconciles independently | Run script per cluster |
| Self-healing | Yes — Flux restores desired state if someone edits live | No — manual intervention required |

The manual approach is simpler to set up and reason about. Flux adds continuous reconciliation and drift correction. Both consume the same OCI artifacts — migrating from manual to Flux later requires no changes to the build/publish pipeline.

---

## Appendix B: Lightweight OCI Drift Checker

This appendix describes a minimal in-cluster drift detection mechanism that continuously compares the OCI source of truth against the canonical running state in etcd / the API server. It is not a full GitOps controller — it answers one question on a schedule: "does what should be running match what is running?"

### B.1 Concept

The core operation is `kubectl diff -k` — it renders the Kustomize manifests from the OCI artifact and compares them against the live cluster state. Everything else is plumbing: pulling the artifact, verifying the signature, and reporting the result.

```
                  ┌─────────────┐
                  │  CronJob     │  (runs every N minutes)
                  └──────┬──────┘
                         │
              ┌──────────┼──────────┐
              ▼                     ▼
    OCI Registry              API Server
    (Source of Truth)         (Running State)
              │                     │
              ▼                     ▼
         oras pull             kubectl get
         cosign verify
              │                     │
              └──────────┬──────────┘
                         │
                    kubectl diff
                         │
                    ┌────┴────┐
                    │         │
                 No drift   Drift detected
                    │         │
                   OK        Alert / Auto-remediate
```

The drift checker has two modes controlled by configuration:

- **Alert-only** — log the drift, send a webhook notification. Safe, non-destructive, gets 80% of Flux's value for 5% of the complexity.
- **Auto-remediate** — apply the OCI manifests to bring the cluster back in line. This is essentially what Flux does, without the event-driven architecture and retry logic.

### B.2 Drift Checker Container Image

Build a minimal container with the required tools. `Containerfile`:

```
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    wget gpg ca-certificates jq \
    && rm -rf /var/lib/apt/lists/*

RUN wget -O- https://sansnom-co.github.io/k8s-tools/public_key.asc \
    | gpg --dearmor > /usr/share/keyrings/sansnom-k8s-tools.gpg
RUN echo "deb [signed-by=/usr/share/keyrings/sansnom-k8s-tools.gpg] https://sansnom-co.github.io/k8s-tools stable main" \
    > /etc/apt/sources.list.d/sansnom-k8s-tools.list
RUN apt-get update && apt-get install -y --no-install-recommends kubectl oras cosign \
    && rm -rf /var/lib/apt/lists/*

COPY drift-check.sh /usr/local/bin/drift-check.sh
RUN chmod +x /usr/local/bin/drift-check.sh

ENTRYPOINT ["/usr/local/bin/drift-check.sh"]
```

### B.3 Drift Checker Script

`drift-check.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration via environment variables
APP_IMAGE="${APP_IMAGE:?APP_IMAGE required}"
DEPLOY_ARTIFACT="${DEPLOY_ARTIFACT:?DEPLOY_ARTIFACT required}"
NAMESPACE="${NAMESPACE:?NAMESPACE required}"
DEPLOYMENT="${DEPLOYMENT:?DEPLOYMENT required}"
COSIGN_KEY="${COSIGN_KEY:-/etc/cosign/cosign.pub}"
AUTO_REMEDIATE="${AUTO_REMEDIATE:-false}"
WEBHOOK_URL="${WEBHOOK_URL:-}"

WORKDIR=$(mktemp -d)
trap 'rm -rf "$WORKDIR"' EXIT

# Resolve the latest version tag from the registry
LATEST_TAG=$(oras repo tags "${DEPLOY_ARTIFACT}" | sort -V | tail -1)
echo "Latest OCI tag: ${LATEST_TAG}"

# Verify app image signature
echo "==> Verifying app image signature"
if ! cosign verify --key "${COSIGN_KEY}" "${APP_IMAGE}:${LATEST_TAG}" >/dev/null 2>&1; then
  echo "ALERT: Signature verification FAILED for ${APP_IMAGE}:${LATEST_TAG}"
  if [[ -n "${WEBHOOK_URL}" ]]; then
    curl -sf -X POST "${WEBHOOK_URL}" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \"🔴 Signature verification failed: ${APP_IMAGE}:${LATEST_TAG}\"}" || true
  fi
  exit 1
fi

# Verify deploy artifact signature
echo "==> Verifying deploy artifact signature"
if ! cosign verify --key "${COSIGN_KEY}" "${DEPLOY_ARTIFACT}:${LATEST_TAG}" >/dev/null 2>&1; then
  echo "ALERT: Signature verification FAILED for ${DEPLOY_ARTIFACT}:${LATEST_TAG}"
  if [[ -n "${WEBHOOK_URL}" ]]; then
    curl -sf -X POST "${WEBHOOK_URL}" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \"🔴 Signature verification failed: ${DEPLOY_ARTIFACT}:${LATEST_TAG}\"}" || true
  fi
  exit 1
fi

# Pull the deploy manifests (OCI source of truth)
echo "==> Pulling deploy manifests"
cd "${WORKDIR}"
oras pull "${DEPLOY_ARTIFACT}:${LATEST_TAG}"

# Compare OCI truth against API server state
echo "==> Comparing OCI truth to cluster state"
DIFF_OUTPUT=$(kubectl diff -k "${WORKDIR}" 2>&1) || true

if [[ -z "${DIFF_OUTPUT}" ]]; then
  echo "OK: OCI truth matches cluster state (${LATEST_TAG})"
  exit 0
fi

# Drift detected
echo "DRIFT DETECTED: OCI truth (${LATEST_TAG}) != cluster state"
echo "---"
echo "${DIFF_OUTPUT}"

# Alert
if [[ -n "${WEBHOOK_URL}" ]]; then
  # Truncate diff for webhook payload
  DIFF_SUMMARY=$(echo "${DIFF_OUTPUT}" | head -50)
  curl -sf -X POST "${WEBHOOK_URL}" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"🟡 Drift detected in ${NAMESPACE}/${DEPLOYMENT} (OCI: ${LATEST_TAG})\n\`\`\`\n${DIFF_SUMMARY}\n\`\`\`\"}" || true
fi

# Auto-remediate if enabled
if [[ "${AUTO_REMEDIATE}" == "true" ]]; then
  echo "==> Auto-remediating: applying OCI truth"
  kubectl apply -k "${WORKDIR}"
  kubectl rollout status "deployment/${DEPLOYMENT}" -n "${NAMESPACE}" --timeout=300s
  echo "==> Remediation complete"

  if [[ -n "${WEBHOOK_URL}" ]]; then
    curl -sf -X POST "${WEBHOOK_URL}" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \"🟢 Drift auto-remediated in ${NAMESPACE}/${DEPLOYMENT} (applied ${LATEST_TAG})\"}" || true
  fi
else
  echo "Auto-remediation disabled. Manual intervention required."
  exit 1
fi
```

### B.4 CronJob Manifest

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: oci-drift-checker
  namespace: myapp-prod
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 1
      activeDeadlineSeconds: 120
      template:
        spec:
          serviceAccountName: drift-checker
          containers:
            - name: checker
              image: git.sansnom.uk/martin/drift-checker:latest
              env:
                - name: APP_IMAGE
                  value: git.sansnom.uk/martin/myapp
                - name: DEPLOY_ARTIFACT
                  value: git.sansnom.uk/martin/myapp-deploy
                - name: NAMESPACE
                  value: myapp-prod
                - name: DEPLOYMENT
                  value: prod-web
                - name: AUTO_REMEDIATE
                  value: "false"
                - name: WEBHOOK_URL
                  valueFrom:
                    secretKeyRef:
                      name: drift-checker-config
                      key: webhook-url
                      optional: true
              volumeMounts:
                - name: cosign-key
                  mountPath: /etc/cosign
                  readOnly: true
                - name: registry-auth
                  # oras/cosign read auth from this path (OCI tooling convention)
                  mountPath: /root/.docker
                  readOnly: true
          volumes:
            - name: cosign-key
              secret:
                secretName: cosign-pub
            - name: registry-auth
              secret:
                secretName: forgejo-pull
                items:
                  # k8s secret key name — not a Docker dependency
                  - key: .dockerconfigjson
                    path: config.json
          restartPolicy: Never
```

### B.5 RBAC

Alert-only mode requires read access. Auto-remediate mode requires write access.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: drift-checker
  namespace: myapp-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: drift-checker
  namespace: myapp-prod
rules:
  # Read — always required
  - apiGroups: ["", "apps"]
    resources:
      - deployments
      - statefulsets
      - services
      - configmaps
      - secrets
      - persistentvolumeclaims
    verbs: ["get", "list"]
  # Write — only required if AUTO_REMEDIATE=true
  # Uncomment the following to enable auto-remediation:
  # - apiGroups: ["", "apps"]
  #   resources:
  #     - deployments
  #     - statefulsets
  #     - services
  #     - configmaps
  #     - secrets
  #     - persistentvolumeclaims
  #   verbs: ["create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: drift-checker
  namespace: myapp-prod
subjects:
  - kind: ServiceAccount
    name: drift-checker
roleRef:
  kind: Role
  name: drift-checker
  apiGroup: rbac.authorization.k8s.io
```

### B.6 Secrets Setup

```bash
# Cosign public key for signature verification
kubectl create secret generic cosign-pub \
  --from-file=cosign.pub=./cosign.pub \
  -n myapp-prod

# Registry credentials (reuse existing pull secret)
# Already created in A.5 as forgejo-pull

# Optional: webhook URL for alerts
kubectl create secret generic drift-checker-config \
  --from-literal=webhook-url="https://hooks.slack.com/services/..." \
  -n myapp-prod
```

### B.7 Operational Modes

| Mode | `AUTO_REMEDIATE` | RBAC | Behaviour |
|---|---|---|---|
| Alert-only | `false` | Read | Detects drift, logs it, sends webhook. No cluster changes. |
| Auto-remediate | `true` | Read + Write | Detects drift, applies OCI truth, confirms rollout, sends webhook. |

Start with alert-only. Promote to auto-remediate once you trust the pipeline and have verified the checker behaves correctly over a few weeks.

### B.8 Comparison to Flux

| Aspect | Drift Checker | Flux |
|---|---|---|
| Architecture | CronJob (batch) | Controller (event-driven) |
| Detection latency | Schedule-based (e.g. 5 min) | Near-realtime (watch + poll) |
| Remediation | Optional, single apply | Continuous reconciliation with retry |
| Dependencies | oras, cosign, kubectl | Flux controllers on cluster |
| Health checks | None (relies on rollout status) | Built-in health assessment |
| Multi-resource | Single `kubectl diff -k` | Per-resource tracking with status |
| Notifications | Webhook in script | Native alert provider integration |
| Complexity | ~100 lines of bash | Full controller framework |

The drift checker is a stepping stone. It validates the OCI-as-source-of-truth pattern with minimal infrastructure. If you later need Flux's capabilities — health assessment, dependency ordering, multi-tenancy — the migration path is clean because both consume the same OCI artifacts.
