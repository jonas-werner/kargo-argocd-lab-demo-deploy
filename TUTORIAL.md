# Kargo + ArgoCD lab — tutorial

This tutorial is a hands-on lab for learning **automated promotion** with **Kargo** and **ArgoCD**, and how that fits together with **release automation** (Release Please).

It intentionally covers **two distinct but complementary flows**:

- **Part 1 (Git-as-Freight)**: Kargo promotes **git commits**.
- **Part 2 (Artifact-as-Freight)**: Release automation produces **versioned artifacts** (images/charts) and Kargo promotes those by updating the deploy repo.

---

## Components and how they interact

### ArgoCD (reconciler)
ArgoCD continuously applies the desired state from git into the Kubernetes cluster.

In this lab, ArgoCD runs **three Applications** (this mirrors real-world “app-of-apps” layouts):

- **`demo-argocd-apps`** (app-of-apps): syncs the deploy repo’s `argocd/` folder and creates/updates other ArgoCD Applications.
- **`demo-prod`** (workload): syncs `apps/demo/overlays/prod` to the `demo` namespace (Deployment/Service).
- **`demo-kargo`** (delivery config): syncs `kargo/` to the `kargo-demo` namespace (Warehouse/Stage/ProjectConfig).

### Kargo (delivery controller)
Kargo detects new “versions” (called **Freight**) and runs a **Promotion** to advance them to a Stage.

What “a new version” means depends on the flow:

- **Part 1**: a new **git commit** under a watched path.
- **Part 2**: a new **artifact version** (image tag or chart version).

### GitHub (source of truth + automation)
- Git repos store desired state (deploy repo) and optionally app code (app repo).
- GitHub Actions can be used to automate releases (Release Please) and artifact publishing.

---

## Repo layout

## If you’re following this tutorial in your own GitHub account

This lab is easiest if you **fork both repos** into your own GitHub user/org and update the few hard-coded repo/image references.

### 1) Fork and clone

- Fork the **deploy repo**
- Fork the **app repo** (only required for Part 2)

Clone your forks locally:

```bash
git clone https://github.com/<YOU>/<DEPLOY_REPO>.git
git clone https://github.com/<YOU>/<APP_REPO>.git
```

### 2) Update repo URLs in the deploy repo

Replace `jonas-werner/kargo-argocd-lab-demo-deploy` with your fork in these files:

- `argocd/application-demo-argocd-apps.yaml`
- `argocd/application-demo-prod.yaml`
- `argocd/application-demo-kargo.yaml`
- `kargo/warehouse-demo.yaml`
- `kargo/stage-demo-prod.yaml` (also contains the `commitFrom("https://github.com/...")` string)

#### Fast path: bulk replace the references

From the deploy repo root:

```bash
export GH_OWNER="<YOUR_GITHUB_USERNAME_OR_ORG>"

# Preview what will change:
grep -RInE "jonas-werner|ghcr\\.io/jonas-werner" .

# macOS (BSD sed)
LC_ALL=C find . -type f \
  \( -path "./.git/*" -o -path "./.github/*" \) -prune -false -o -print0 \
| xargs -0 sed -i '' \
  -e "s#https://github.com/jonas-werner/kargo-argocd-lab-demo-deploy\\.git#https://github.com/${GH_OWNER}/kargo-argocd-lab-demo-deploy.git#g" \
  -e "s#ghcr\\.io/jonas-werner/kargo-argocd-lab-demo-app#ghcr.io/${GH_OWNER}/kargo-argocd-lab-demo-app#g"

# If you're on Linux, use this instead (GNU sed):
# find . -type f \( -path "./.git/*" -o -path "./.github/*" \) -prune -false -o -print0 \
# | xargs -0 sed -i \
#   -e "s#https://github.com/jonas-werner/kargo-argocd-lab-demo-deploy\\.git#https://github.com/${GH_OWNER}/kargo-argocd-lab-demo-deploy.git#g" \
#   -e "s#ghcr\\.io/jonas-werner/kargo-argocd-lab-demo-app#ghcr.io/${GH_OWNER}/kargo-argocd-lab-demo-app#g"

# Verify:
grep -RInE "jonas-werner|ghcr\\.io/jonas-werner" . || true
git diff
```

### 3) Update the image name (only needed for Part 2)

If you plan to run Part 2 (artifact-as-freight), update the image repo to your own:

- `apps/demo/base/deployment.yaml`
- `apps/demo/overlays/*/kustomization.yaml`

Use:

- `ghcr.io/<YOU>/<APP_REPO>`

### 4) GitHub Actions permissions (only needed for Part 2)

In the **app repo** on GitHub:

- Settings → Actions → General → Workflow permissions
  - Enable **Read and write permissions**
  - Enable **Allow GitHub Actions to create and approve pull requests**

If you publish to GHCR and get `write_package` errors, you may need to explicitly grant the repo permission to publish the package (or use a PAT with `write:packages` as a secret).

### Deploy repo (this repo)
`jonas-werner/kargo-argocd-lab-demo-deploy`

- **`argocd/`**: ArgoCD Applications (app-of-apps + workload + Kargo config)
- **`kargo/`**: Kargo CRs (Warehouse/Stage/ProjectConfig)
- **`apps/demo/`**: Kubernetes manifests (Kustomize base + overlays)

### App repo (optional for Part 2)
`jonas-werner/kargo-argocd-lab-demo-app`

- Demo app code / container build context
- Release Please config and GitHub Actions workflow(s)

---

## Setup (Orbstack + Kubernetes + ArgoCD + Kargo)

This section describes a repeatable local environment on Orbstack. If you already have ArgoCD/Kargo running, you can skip to “Install the ArgoCD bootstrap app”.

### Prerequisites
- Orbstack Kubernetes enabled
- `kubectl`, `helm`
- Optional: `argocd` CLI (helpful for troubleshooting)

### Create namespaces

```bash
kubectl create ns argocd || true
kubectl create ns kargo || true
kubectl create ns kargo-demo || true
kubectl create ns demo || true
```

### Create a Git credential Secret for Kargo (required for Part 1)

Part 1 promotions **commit and push back** to the deploy repo. For that, Kargo needs Git credentials.

Create a GitHub Personal Access Token (PAT) that can write to your **deploy repo** (for a public repo, `public_repo` is sufficient; for a private repo, use `repo`).

Then create a Secret in the project namespace (`kargo-demo`) with a well-known label so Kargo can discover it:

```bash
export GITOPS_REPO_URL="https://github.com/<YOU>/kargo-argocd-lab-demo-deploy.git"
export GITHUB_USERNAME="<YOU>"
export GITHUB_PAT="<YOUR_PAT>"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo
  labels:
    kargo.akuity.io/cred-type: git
stringData:
  repoURL: ${GITOPS_REPO_URL}
  # For GitHub HTTPS auth with PATs, the username should be the GitHub handle
  # of the user who created the token.
  username: ${GITHUB_USERNAME}
  password: ${GITHUB_PAT}
EOF
```

If you previously created other git credential Secrets in the same namespace, remove them or ensure they target different `repoURL` values. Having multiple Secrets that all match the same repo can make credential selection ambiguous.

### Install ArgoCD
Install ArgoCD using your preferred method (Helm, manifests, etc.). For Helm-based installs, follow the official ArgoCD Helm chart docs.

After install, verify:

```bash
kubectl -n argocd get pods
```

### Install Kargo
Install Kargo using Helm and verify:

```bash
kubectl -n kargo get pods
```

### (Optional) Set / rotate the Kargo admin password
If you need to rotate the Kargo admin password in-cluster, the lab installation typically stores it in the Secret `kargo/kargo-api` as `ADMIN_ACCOUNT_PASSWORD_HASH`.

---

## Install the ArgoCD bootstrap app (app-of-apps)

This lab uses `demo-argocd-apps` as the bootstrap entrypoint:

```bash
kubectl -n argocd apply -f argocd/application-demo-argocd-apps.yaml
```

Wait until it is Synced/Healthy in the ArgoCD UI:
- `demo-argocd-apps`
- `demo-prod`
- `demo-kargo`

If `demo-kargo` is OutOfSync, sync it (or wait for ArgoCD automated sync).

---

## Part 1 — Git-as-Freight

### Goal
Demonstrate the Git-as-Freight pattern:

- **Warehouse** detects new commits from a repo/path.
- **Promotion** pins an ArgoCD app’s `targetRevision` to a commit SHA.
- ArgoCD syncs that revision.

### How this lab is configured

- Warehouse: `kargo-demo/warehouse.demo` watches this deploy repo’s `main` branch under `apps/demo/**`
- Stage: `kargo-demo/stage.demo-prod` promotes directly from that Warehouse
- ProjectConfig: auto-promotion enabled for `demo-prod`

### What you change to trigger the flow
Make any commit that changes something under:

- `apps/demo/**`

Examples:
- increase replicas in `apps/demo/base/deployment.yaml`
- change an environment variable
- change resources or labels

### What you should observe

1) A new Freight appears:

```bash
kubectl -n kargo-demo get freight --sort-by=.metadata.creationTimestamp
```

2) A Promotion is created automatically and succeeds:

```bash
kubectl -n kargo-demo get promotions --sort-by=.metadata.creationTimestamp
```

3) The `demo-prod` ArgoCD Application gets its `spec.source.targetRevision` updated to a commit SHA:

```bash
kubectl -n argocd get application demo-prod -o jsonpath='{.spec.source.targetRevision}{"\n"}'
```

4) The `demo` namespace workload updates:

```bash
kubectl -n demo get deploy,po
```

---

## Part 2 — Artifact-as-Freight (Release Please → publish → promote)

### Goal
Demonstrate the “artifact pipeline” used by repos like `sa-prime`:

- Release automation (Release Please) creates version bumps/tags/releases
- CI publishes artifacts (images or charts)
- Kargo detects new artifact versions and promotes them by committing an update in the deploy repo
- ArgoCD syncs the updated desired state

### The key difference from Part 1
In Part 2, **the “new version” is an artifact version** (e.g. `0.4.0` image tag), not a git commit.

That means:
- the Warehouse subscription becomes `image:` (or `helm:`), and
- the Stage promotion updates the deploy repo to reference that new version (e.g. update `kustomization.yaml` image tag).

### Recommended tutorial approach
To keep the tutorial dependable, do Part 1 first (no registry friction). Then introduce Part 2 as an “add-on”:

1) Ensure the app repo can publish images (or use a public image repo).
2) Update Warehouse to watch the image tags.
3) Update Stage to change `apps/demo/overlays/prod/kustomization.yaml` image tag.

### GitHub Actions settings you’ll likely need (for Part 2)
In the **app repo**:
- Actions “Workflow permissions”: **Read and write permissions**
- Enable: **Allow GitHub Actions to create and approve pull requests** (for Release Please PRs)
- For GHCR publishing, ensure the package/repo linkage allows publishing (or use a PAT with `write:packages`)

---

## Troubleshooting quick hits

### ArgoCD apps exist but Kargo doesn’t react
Make sure:
- `demo-kargo` exists and is Synced (so `kargo/` manifests are actually applied)
- Warehouse subscription matches what you’re changing (git path vs image tags)

### Kargo is installed but you don’t see any Freight
Check the Warehouse:

```bash
kubectl -n kargo-demo get warehouse demo -o yaml | sed -n '1,220p'
```

### Promotions are failing
Inspect the most recent Promotion:

```bash
kubectl -n kargo-demo get promotions --sort-by=.metadata.creationTimestamp
kubectl -n kargo-demo describe promotion <name>
```


