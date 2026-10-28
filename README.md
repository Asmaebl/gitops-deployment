# gitops-deployment

GitOps repository for the **k3s + GitOps internship project**.

This repository contains the Kubernetes desired state of the
[`asmae-bouhlal`](https://github.com/Asmaebl/gitops-application) application and
the configuration required to reconcile it with either **Argo CD** or **Flux CD**.

The project evaluates both GitOps controllers against the same application and
the same Kubernetes manifests.

---

## Purpose

The repository acts as the **source of truth** for the Kubernetes deployment.

Application CI does not deploy directly to the cluster. Instead, after a
successful test and container build, GitHub Actions updates the image reference
in this repository.

The active GitOps controller detects the Git change and reconciles the k3s
cluster with the desired state stored here.

```text
Application repository
        |
        | push to main
        v
GitHub Actions
        |
        | test
        | build
        | push image
        v
GitHub Container Registry
        |
        | update image tag
        v
GitOps repository
gitops-deployment
        |
        +--------------------+
        |                    |
        v                    v
     Argo CD              Flux CD
        |                    |
        +---------+----------+
                  |
                  v
              k3s cluster
                  |
                  v
          asmae-bouhlal
```

Only one GitOps controller should actively reconcile the application at a time
during the comparison.

---

## Related repository

Application source code, tests, Dockerfile and GitHub Actions workflow:

https://github.com/Asmaebl/gitops-application

Container images are published to:

```text
ghcr.io/asmaebl/gitops-application:<git-sha>
```

---

## Repository structure

```text
.
├── argocd/
│   └── application.yaml
│
├── base/
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
│
└── clusters/
    └── my-cluster/
        ├── apps.yaml
        └── flux-system/
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

### Directories

| Path | Purpose |
| --- | --- |
| `base/` | Kubernetes manifests shared by both GitOps controllers |
| `argocd/` | Argo CD `Application` definition |
| `clusters/my-cluster/` | Flux cluster and application configuration |
| `clusters/my-cluster/flux-system/` | Flux bootstrap manifests |

Both Argo CD and Flux CD reconcile the same manifests under `base/`, which keeps
the comparison focused on the GitOps controllers rather than on different
application configurations.

---

## Kubernetes manifests

The application is deployed using Kustomize-compatible Kubernetes manifests.

### Deployment

`base/deployment.yaml` defines the application Deployment.

The main configuration includes:

- 2 replicas
- container port `3000`
- immutable GHCR image tag based on the application Git SHA
- liveness probe on `/healthz`
- readiness probe on `/readyz`
- CPU and memory requests and limits
- non-root container execution
- disabled privilege escalation
- Kubernetes Downward API environment variables
- private GHCR authentication through `imagePullSecrets`

The Deployment uses:

```yaml
imagePullSecrets:
  - name: ghcr-secret
```

The registry credential itself is deliberately **not stored in Git**.

### Service

`base/service.yaml` exposes the application internally using a Kubernetes
`ClusterIP` Service:

```text
Service port: 80
        |
        v
Container port: 3000
```

### Kustomization

`base/kustomization.yaml` groups the application Kubernetes resources so the
same base can be consumed by the GitOps controllers.

---

## Prerequisites

To reproduce the deployment, the following tools are required depending on the
selected GitOps controller:

- Git
- a running k3s cluster
- kubectl / k3s kubectl
- access to this Git repository
- access to the private GHCR package
- Argo CD for the Argo CD scenario
- Flux CLI for the Flux CD scenario

Verify the cluster before installing a GitOps controller:

```sh
sudo k3s kubectl get nodes
```

The node or nodes used for the deployment should report:

```text
Ready
```

---

## Clone the repository

```sh
git clone git@github.com:Asmaebl/gitops-deployment.git
cd gitops-deployment
```

---

## Private GHCR access

The application image is stored in GitHub Container Registry.

Because the package may be private, Kubernetes requires an image pull Secret in
the target namespace.

Create the Secret locally using your own GitHub credentials:

```sh
sudo k3s kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<github-token> \
  --namespace=default
```

The GitHub token must have the permissions required to read the private package.

Verify that the Secret exists:

```sh
sudo k3s kubectl get secret ghcr-secret -n default
```

The Secret must remain inside Kubernetes and must **not** be committed to this
repository.

---

# Argo CD deployment

## Install Argo CD

Create the namespace:

```sh
sudo k3s kubectl create namespace argocd
```

Install Argo CD using the official installation manifest:

```sh
sudo k3s kubectl apply \
  -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until the Argo CD components are ready:

```sh
sudo k3s kubectl get pods -n argocd
```

All required Argo CD components should eventually report `Running`.

---

## Private Git repository access with Argo CD

This GitOps repository is private, so Argo CD requires credentials allowing its
repository service to read it.

The credential must be configured as an Argo CD repository credential or
repository Secret.

SSH private keys and other credentials must **not** be stored in the GitOps
repository.

After repository access has been configured, the application can be created
from:

```text
argocd/application.yaml
```

---

## Argo CD Application

The Argo CD configuration points to:

```text
Repository: Asmaebl/gitops-deployment
Branch:     main
Path:       base
Namespace:  default
```

Apply it with:

```sh
sudo k3s kubectl apply -f argocd/application.yaml
```

The Application enables:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

`prune` allows Argo CD to remove managed resources that are removed from Git.

`selfHeal` allows Argo CD to correct supported live-cluster drift and restore
the desired state stored in Git.

---

## Verify Argo CD

Check the Application:

```sh
sudo k3s kubectl get applications -n argocd
```

The expected application state is:

```text
Synced
Healthy
```

Check the workload:

```sh
sudo k3s kubectl get deployment asmae-bouhlal
sudo k3s kubectl get pods
sudo k3s kubectl get service asmae-bouhlal
```

---

## Argo CD self-healing test

The desired Deployment contains:

```yaml
replicas: 2
```

To introduce temporary drift:

```sh
sudo k3s kubectl scale deployment asmae-bouhlal --replicas=1
```

The live cluster temporarily differs from Git:

```text
Git desired state:     2 replicas
Live cluster state:    1 replica
```

With automated self-healing enabled, Argo CD detects the drift and restores the
Git-defined replica count.

Observe the Deployment:

```sh
sudo k3s kubectl get deployment asmae-bouhlal -w
```

The expected final state is again:

```text
2 desired replicas
2 available replicas
```

---

# Switching from Argo CD to Flux CD

The comparison uses the same application manifests, but Argo CD and Flux CD
should not simultaneously manage the same application resources.

Before testing Flux CD, remove the Argo CD `Application` while preserving the
application resources.

The objective is to transfer reconciliation responsibility to Flux without
requiring a different application configuration.

Verify that no Argo CD Application is actively reconciling the workload before
enabling the Flux application Kustomization.

---

# Flux CD deployment

## Install the Flux CLI

Install the Flux CLI using the official Flux installation instructions.

Verify the installation:

```sh
flux --version
```

Check that the cluster satisfies the Flux prerequisites:

```sh
sudo flux check --pre \
  --kubeconfig=/etc/rancher/k3s/k3s.yaml
```

---

## Bootstrap Flux

Bootstrap Flux against the GitOps repository:

```sh
flux bootstrap git \
  --url=ssh://git@ssh.github.com:443/Asmaebl/gitops-deployment.git \
  --branch=main \
  --private-key-file=<path-to-private-ssh-key> \
  --path=clusters/my-cluster
```

`<path-to-private-ssh-key>` is a placeholder. Each user must provide the path to
their own SSH private key with access to the GitOps repository.

The SSH private key itself must never be committed to Git.

Depending on the local k3s kubeconfig permissions, Flux commands may require an
explicit kubeconfig path.

Flux stores its generated cluster configuration under:

```text
clusters/my-cluster/flux-system/
```

The resulting repository contains:

```text
gotk-components.yaml
gotk-sync.yaml
kustomization.yaml
```

`gotk-components.yaml` contains the Flux controllers and CRDs generated during
bootstrap.

`gotk-sync.yaml` defines the Git source and the Flux system Kustomization.

---

## Git access over SSH

Flux reads this private repository using SSH.

The configured Git source is:

```text
ssh://git@ssh.github.com:443/Asmaebl/gitops-deployment.git
```

Port `443` is used for SSH access in this environment because direct SSH access
to GitHub over port `22` was unavailable.

The Flux SSH Secret contains the credentials required by Flux to access the
private repository.

The Secret remains inside the cluster and is **not committed to Git**.

---

## Application Kustomization

The application reconciliation configuration is stored in:

```text
clusters/my-cluster/apps.yaml
```

It defines a Flux `Kustomization` named:

```text
asmae-bouhlal
```

The Kustomization uses:

```yaml
path: ./base
interval: 1m
prune: true
targetNamespace: default
```

Flux therefore checks the Git source and ensures that the Kubernetes resources
under `base/` are applied to the `default` namespace.

---

## Verify Flux

Check the Flux installation:

```sh
sudo flux check \
  --kubeconfig=/etc/rancher/k3s/k3s.yaml
```

Check all Flux resources:

```sh
sudo flux get all -A \
  --kubeconfig=/etc/rancher/k3s/k3s.yaml
```

The Git source and both Kustomizations should report:

```text
READY=True
```

Check the Flux controllers:

```sh
sudo k3s kubectl get pods -n flux-system
```

The bootstrap used in this project runs the following controllers:

```text
helm-controller
kustomize-controller
notification-controller
source-controller
```

---

## Flux reconciliation

Flux periodically compares the desired state obtained from Git with the state
of the Kubernetes resources it manages.

When Git changes, Flux applies the new desired state.

When supported live configuration drifts from the Git-defined state, Flux
reconciliation restores the desired configuration.

A reconciliation can also be requested manually when immediate synchronization
is useful:

```sh
sudo flux reconcile source git flux-system \
  --kubeconfig=/etc/rancher/k3s/k3s.yaml
```

Application reconciliation can be requested with:

```sh
sudo flux reconcile kustomization asmae-bouhlal \
  --with-source \
  --kubeconfig=/etc/rancher/k3s/k3s.yaml
```

Manual reconciliation is not required during normal operation; Flux reconciles
automatically according to the configured intervals.

---

## Flux self-healing test

Introduce temporary drift:

```sh
sudo k3s kubectl scale deployment asmae-bouhlal --replicas=1
```

Observe the Deployment:

```sh
sudo k3s kubectl get deployment asmae-bouhlal -w
```

Because Git declares two replicas, Flux reconciliation restores:

```text
2 replicas
```

This validates Git as the source of truth.

---

# Verify the deployed application

Check the Deployment:

```sh
sudo k3s kubectl get deployment asmae-bouhlal
```

Check the Pods:

```sh
sudo k3s kubectl get pods -l app=asmae-bouhlal
```

Check the Service:

```sh
sudo k3s kubectl get service asmae-bouhlal
```

To access the ClusterIP Service from the local machine, use port forwarding:

```sh
sudo k3s kubectl port-forward service/asmae-bouhlal 8080:80
```

Then, from another terminal:

```sh
curl http://127.0.0.1:8080/
curl http://127.0.0.1:8080/healthz
curl http://127.0.0.1:8080/readyz
curl http://127.0.0.1:8080/version
```

Expected health responses:

```text
/healthz -> ok
/readyz  -> ready
```

`/version` reports the application version and the Git commit embedded in the
container image.

This provides traceability from the running application back to the application
repository commit.

---

# Application delivery workflow

A normal application change follows this path:

```text
1. Developer pushes application code
                |
                v
2. GitHub Actions runs tests
                |
                v
3. CI builds the Docker image
                |
                v
4. Image is pushed to GHCR
   ghcr.io/asmaebl/gitops-application:<git-sha>
                |
                v
5. CI updates base/deployment.yaml
                |
                v
6. CI commits the GitOps change
                |
                v
7. Argo CD or Flux CD detects the new Git state
                |
                v
8. GitOps reconciliation updates the k3s cluster
                |
                v
9. New application version becomes available
```

The CI pipeline does not use cluster credentials.

---

# Rollback

Rollback follows the GitOps model and is initiated from the application
repository using Git history.

For example:

```sh
git revert <application-commit>
git push origin main
```

This creates a new application commit that reverses the selected change without
rewriting Git history.

The normal delivery pipeline then:

```text
Revert commit
      |
      v
GitHub Actions
      |
      v
Tests
      |
      v
New GHCR image
      |
      v
GitOps image update
      |
      v
Argo CD / Flux CD reconciliation
      |
      v
Reverted application state
```

A controlled rollback was validated during the project by deploying a temporary
application endpoint and subsequently reverting the commit.

After reconciliation:

- the temporary endpoint was no longer available;
- `/healthz` remained healthy;
- `/version` reported the new revert commit.

This demonstrates rollback while preserving Git as the source of truth.

---

# Security considerations

The project follows several separation and credential-management principles:

- Kubernetes cluster credentials are not stored in the application CI pipeline.
- GitHub tokens are not committed to Git.
- SSH private keys are not committed to Git.
- GHCR credentials are stored as a Kubernetes Secret.
- Git repository credentials used by the GitOps controllers remain inside the cluster.
- the application container runs as a non-root user;
- privilege escalation is disabled;
- resource requests and limits are configured;
- container images are referenced using immutable Git SHA tags.

---

# Argo CD vs Flux CD

Both controllers were tested against the same Kubernetes application manifests.

| Criterion | Argo CD | Flux CD |
| --- | --- | --- |
| GitOps deployment | Validated | Validated |
| Automatic synchronization | Validated | Validated |
| Self-healing | Validated | Validated |
| Git-based rollback workflow | Supported by the GitOps model | Validated during final rollback test |
| Application-oriented Web UI | Integrated | No equivalent primary built-in application UI |
| Kubernetes-native CRD workflow | Yes | Strongly controller/CRD-oriented |
| Controllers observed in this project | 7 pods | 4 pods |
| Observed memory during measurement | ~224 MiB | ~122 MiB |
| Observed CPU during measurement | ~35m | ~34m |

The resource measurements above are point-in-time observations from the project
environment. They are not universal performance characteristics of either
product.

Argo CD provides strong application-oriented visibility and an integrated
graphical interface.

Flux CD provides a lightweight, Kubernetes-native controller model that
integrates naturally with declarative Kubernetes workflows.

Based on the observations made in this project environment, Flux CD was selected
for the final deployment configuration. It provided the required reconciliation
and self-healing capabilities while using fewer controller pods and less memory
during the recorded measurements.

Argo CD remains particularly useful when graphical application visibility and
interactive operational monitoring are priorities.

---

# Reproduction checklist

A colleague reproducing the project should be able to follow this sequence:

```text
1. Prepare a k3s cluster
2. Verify that the cluster node(s) are Ready
3. Clone the application repository
4. Run the application tests
5. Clone this GitOps repository
6. Configure private GHCR access
7. Choose one GitOps controller
      ├── Argo CD
      └── Flux CD
8. Configure private Git repository access
9. Reconcile the base manifests
10. Verify the Deployment and Pods
11. Verify /healthz and /readyz
12. Verify /version
13. Push an application change
14. Observe CI build and GitOps repository update
15. Observe automatic deployment
16. Introduce temporary cluster drift
17. Verify self-healing
18. Perform a Git revert
19. Verify the rollback through GitOps
```

---

## Important operational note

Argo CD and Flux CD configurations are both kept in this repository for the
comparison, but they should not simultaneously reconcile the same application
resources during the experiment.

Use one controller as the active reconciler for the application at a time.
