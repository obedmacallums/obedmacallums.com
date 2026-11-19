---
author: Obed Macallums
pubDatetime: 2026-11-19T12:35:30Z
title: "Deploying a FastAPI Application to Kubernetes with GitHub Actions"
slug: fastapi-kubernetes-github-actions
featured: false
draft: false
tags:
  - fastapi
  - kubernetes
  - docker
  - cicd
  - devops
  - github
description: Build an end-to-end deployment pipeline for a FastAPI service. Containerize the app with a lean Dockerfile, write production-ready Kubernetes manifests with health probes, resource limits and ConfigMaps, and automate testing, image publishing to GitHub Container Registry and rolling updates with GitHub Actions.
---

Writing a FastAPI service is the easy part. Getting every commit on `main` tested, packaged into an immutable image and rolled out to a Kubernetes cluster without downtime is where most projects start accumulating shell scripts and tribal knowledge. In this article we build that path end to end: a small **FastAPI** app, a **Docker** image, a set of **Kubernetes** manifests with health probes and resource limits, and a **GitHub Actions** pipeline that tests, builds, pushes and rolls out each change automatically.

## Table of Contents

## What We Are Building

The previous posts on this blog covered each piece in isolation: [FastAPI basics](/posts/introduction-to-fastapi/), [Kubernetes core concepts](/posts/introduction-to-kubernetes/), [GitHub Actions workflows](/posts/github-actions-cicd/) and [local orchestration with Docker Compose](/posts/docker-compose-guide/). Here we connect them into a single delivery pipeline:

1. A developer pushes to `main` (or opens a pull request).
2. GitHub Actions runs the test suite with `pytest`.
3. If the tests pass on `main`, the workflow builds a Docker image and pushes it to **GitHub Container Registry (GHCR)**, tagged with the commit SHA.
4. The workflow updates the Kubernetes Deployment to that exact image and waits for the rollout to finish.
5. Kubernetes performs a rolling update, only sending traffic to new Pods once their readiness probe passes.

The final repository layout looks like this:

```bash
fastapi-k8s-demo/
├── app/
│   ├── __init__.py
│   └── main.py
├── tests/
│   └── test_main.py
├── k8s/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── .github/
│   └── workflows/
│       └── deploy.yaml
├── .dockerignore
├── Dockerfile
├── requirements.txt
└── requirements-dev.txt
```

## The FastAPI Application

The application is intentionally small, but it includes the two things Kubernetes needs from any service: a **liveness** endpoint ("is the process alive?") and a **readiness** endpoint ("can it accept traffic right now?"). It also reads its configuration from environment variables so the same image can run in any environment.

```python
# app/main.py
import os

from fastapi import FastAPI
from pydantic import BaseModel

APP_VERSION = os.getenv("APP_VERSION", "dev")
GREETING = os.getenv("GREETING", "Hello")

app = FastAPI(title="fastapi-k8s-demo", version=APP_VERSION)


class Item(BaseModel):
    name: str
    price: float


@app.get("/")
def root():
    return {"message": f"{GREETING} from Kubernetes!", "version": APP_VERSION}


@app.get("/healthz")
def healthz():
    """Liveness probe: the process is up and the event loop responds."""
    return {"status": "ok"}


@app.get("/readyz")
def readyz():
    """Readiness probe: dependencies (DB, cache, models) are ready.

    In a real service, check your connections here and return a 503
    when the app cannot serve requests yet.
    """
    return {"status": "ready"}


@app.post("/items")
def create_item(item: Item):
    return {"name": item.name, "price_with_tax": round(item.price * 1.19, 2)}
```

Keeping liveness and readiness separate matters. If the database goes down, you want the readiness probe to fail so Kubernetes stops routing traffic to the Pod, but you do **not** want the liveness probe to fail, because restarting the container would not fix the database.

### Dependencies

Split runtime and development dependencies so the production image stays small:

```bash
# requirements.txt
fastapi>=0.110,<1.0
uvicorn[standard]>=0.29,<1.0
```

```bash
# requirements-dev.txt
-r requirements.txt
pytest>=8.0
httpx>=0.27
```

`httpx` is required by FastAPI's `TestClient`.

### Tests

The pipeline will refuse to deploy anything that fails these tests:

```python
# tests/test_main.py
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_root():
    response = client.get("/")
    assert response.status_code == 200
    assert "message" in response.json()


def test_health_endpoints():
    assert client.get("/healthz").status_code == 200
    assert client.get("/readyz").status_code == 200


def test_create_item():
    response = client.post("/items", json={"name": "sensor", "price": 100})
    assert response.status_code == 200
    assert response.json() == {"name": "sensor", "price_with_tax": 119.0}
```

Run them locally before going any further:

```bash
pip install -r requirements-dev.txt
pytest -q
uvicorn app.main:app --reload
```

## Containerizing the Service

A good Dockerfile for Kubernetes has three properties: it is small, it runs as a non-root user, and it starts the server directly (so signals like `SIGTERM` reach Uvicorn and it can shut down gracefully during rolling updates).

```dockerfile
# Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Install dependencies first to take advantage of layer caching
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy the application code
COPY app ./app

# Create and switch to an unprivileged user
RUN useradd --create-home --uid 10001 appuser
USER 10001

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

A few deliberate choices:

- **One Uvicorn process per container.** In Kubernetes, horizontal scaling is handled by adding Pods, not by adding workers inside a container. This keeps memory usage predictable and makes resource limits meaningful.
- **Exec-form `CMD`.** Using the JSON array form means Uvicorn runs as PID 1 and receives termination signals directly.
- **Numeric `USER`.** Using a UID lets Kubernetes verify the `runAsNonRoot` security setting we will add later.

Add a `.dockerignore` so tests, Git history and local virtual environments never end up in the image:

```bash
# .dockerignore
.git
.github
.venv
__pycache__
*.pyc
tests
k8s
```

Build and run it locally to make sure it works:

```bash
docker build -t fastapi-k8s-demo:local .
docker run --rm -p 8000:8000 -e GREETING=Hola fastapi-k8s-demo:local
curl http://localhost:8000/
```

## Writing the Kubernetes Manifests

We will keep the manifests as plain YAML in the `k8s/` directory. Tools like Helm or Kustomize are worth adopting later, but plain manifests make every moving part visible.

### Namespace

Isolating the application in its own namespace makes permissions and cleanup simpler:

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fastapi-demo
```

### ConfigMap

Configuration that is not secret lives in a ConfigMap and is injected as environment variables:

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fastapi-demo-config
  namespace: fastapi-demo
data:
  GREETING: "Hello"
```

For sensitive values (database passwords, API keys), use a `Secret` instead and reference it the same way with `secretRef`.

### Deployment

The Deployment is where most of the production-readiness lives:

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-demo
  namespace: fastapi-demo
  labels:
    app: fastapi-demo
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: fastapi-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: fastapi-demo
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
        - name: api
          image: ghcr.io/your-user/fastapi-k8s-demo:latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8000
          envFrom:
            - configMapRef:
                name: fastapi-demo-config
          env:
            - name: APP_VERSION
              value: "manual"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
```

What each block does:

- **`strategy.rollingUpdate`**: with `maxUnavailable: 0` and `maxSurge: 1`, Kubernetes creates one new Pod, waits for it to become ready, and only then terminates an old one. Capacity never drops below three replicas during a deploy.
- **`readinessProbe`**: a Pod is added to the Service endpoints only after `/readyz` returns a success status. This is what makes the rolling update zero-downtime.
- **`livenessProbe`**: if `/healthz` fails three times in a row, the kubelet restarts the container.
- **`resources`**: requests tell the scheduler how much capacity to reserve; the memory limit protects the node from a runaway process. Leaving the CPU limit unset avoids CPU throttling, a common and reasonable choice for latency-sensitive APIs.
- **`securityContext`**: the container runs as a non-root user with a read-only filesystem. Since our app writes nothing to disk (`PYTHONDONTWRITEBYTECODE=1`), this works out of the box.
- **`image`**: the `latest` tag here is only a placeholder for the first apply. The pipeline will always replace it with an immutable, SHA-tagged image.

### Service

The Service gives the Pods a stable DNS name and load-balances across ready replicas:

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-demo
  namespace: fastapi-demo
spec:
  type: ClusterIP
  selector:
    app: fastapi-demo
  ports:
    - name: http
      port: 80
      targetPort: http
```

`ClusterIP` keeps the service internal. To expose it publicly, either change the type to `LoadBalancer` on a cloud provider or put an Ingress in front of it.

## Testing the Manifests Locally with kind

Before wiring up CI, validate the manifests against a real cluster. [kind](https://kind.sigs.k8s.io/) runs Kubernetes inside Docker containers and is ideal for this:

```bash
# Create a local cluster
kind create cluster --name fastapi-demo

# Load the locally built image into the cluster nodes
docker build -t fastapi-k8s-demo:local .
kind load docker-image fastapi-k8s-demo:local --name fastapi-demo

# Apply the manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/

# Point the Deployment at the local image and watch the rollout
kubectl -n fastapi-demo set image deployment/fastapi-demo api=fastapi-k8s-demo:local
kubectl -n fastapi-demo rollout status deployment/fastapi-demo

# Reach the service through a port-forward
kubectl -n fastapi-demo port-forward service/fastapi-demo 8080:80
curl http://localhost:8080/
```

The namespace is applied first because the other manifests reference it. If a Pod is stuck, `kubectl -n fastapi-demo describe pod <name>` and `kubectl -n fastapi-demo logs <name>` usually reveal the cause: a failing probe, a wrong port or an image that cannot be pulled.

## Automating the Pipeline with GitHub Actions

Now for the part that ties everything together. The workflow has three jobs that run in sequence: `test`, `build`, and `deploy`.

### Preparing Cluster Access

The deploy job needs credentials for your cluster. The simplest portable approach is to store a kubeconfig file as an encrypted repository secret:

1. Create a dedicated ServiceAccount in the cluster with permissions limited to the `fastapi-demo` namespace (via a `Role` and `RoleBinding`).
2. Generate a kubeconfig that uses that ServiceAccount's token.
3. Base64-encode it and save it in **Settings → Secrets and variables → Actions** as `KUBE_CONFIG`.

```bash
base64 -w0 deploy-kubeconfig.yaml
```

If you use a managed service such as EKS, GKE or AKS, prefer the provider's official login action with OpenID Connect (OIDC) federation instead of long-lived credentials. The rest of the workflow stays the same.

Also make sure the image in GHCR can be pulled by the cluster: either make the package public, or create an `imagePullSecret` in the namespace and reference it from the Deployment.

### The Workflow File

```yaml
# .github/workflows/deploy.yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Run tests
        run: pytest -q

  build:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.image.outputs.ref }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute image reference
        id: image
        run: |
          IMAGE="${REGISTRY}/${IMAGE_NAME,,}"
          echo "ref=${IMAGE}:${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
          echo "latest=${IMAGE}:latest" >> "$GITHUB_OUTPUT"

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ steps.image.outputs.ref }}
            ${{ steps.image.outputs.latest }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: production-deploy
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure cluster access
        run: |
          mkdir -p "$HOME/.kube"
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > "$HOME/.kube/config"
          chmod 600 "$HOME/.kube/config"

      - name: Render image and version into the manifest
        env:
          IMAGE: ${{ needs.build.outputs.image }}
        run: |
          sed -i "s|image: ghcr.io/your-user/fastapi-k8s-demo:latest|image: ${IMAGE}|" k8s/deployment.yaml
          sed -i "s|value: \"manual\"|value: \"${GITHUB_SHA::7}\"|" k8s/deployment.yaml
          grep -E "image:|value:" k8s/deployment.yaml

      - name: Apply manifests
        run: |
          kubectl apply -f k8s/namespace.yaml
          kubectl apply -f k8s/

      - name: Wait for rollout
        run: |
          if ! kubectl -n fastapi-demo rollout status deployment/fastapi-demo --timeout=180s; then
            echo "Rollout failed, rolling back"
            kubectl -n fastapi-demo rollout undo deployment/fastapi-demo
            exit 1
          fi
```

### How the Workflow Works

**Triggers and gating.** Pull requests only run the `test` job. The `build` job has an `if` condition that restricts it to pushes on `main`, and `deploy` depends on `build`, so nothing reaches the cluster from a feature branch.

**Least-privilege permissions.** The `build` job declares `packages: write` so the automatically provided `GITHUB_TOKEN` can push to GHCR. No personal access token is needed.

**Immutable image tags.** Every image is tagged with the full commit SHA. Registry paths must be lowercase, so the step uses Bash's `${VAR,,}` expansion to lowercase `owner/repo`. The `latest` tag is pushed too, but the cluster always runs the SHA tag, which means you can always tell exactly which commit is live and roll back to a known image.

**Build caching.** `cache-from` and `cache-to` with `type=gha` store Docker layers in the GitHub Actions cache. As long as `requirements.txt` does not change, the dependency layer is reused and builds finish much faster.

**Protected environment.** `environment: production` links the job to a GitHub environment. In the repository settings you can require manual approval from a reviewer, restrict which branches may deploy, and scope the `KUBE_CONFIG` secret to that environment only.

**Serialized deploys.** The `concurrency` group ensures two pushes in quick succession never run `kubectl` against the cluster at the same time. With `cancel-in-progress: false`, a running deploy finishes before the next one starts.

**Rendering the manifest.** Before applying anything, two `sed` commands replace the placeholder image with the SHA-tagged image and set `APP_VERSION` to the short commit hash. The change only exists in the runner's checkout; the repository keeps the placeholder. Because the final image is already in the file, a single `kubectl apply` triggers exactly one rolling update. If you run `kubectl apply` first and `kubectl set image` afterwards, you can get two rollouts, the first one briefly reverting to the placeholder image.

**Rollout and rollback.** `kubectl apply` keeps the Namespace, ConfigMap, Service and Deployment in sync with the repository. A new image or `APP_VERSION` changes the Pod template, which triggers the rolling update. `rollout status` blocks until all new Pods are ready or the timeout expires; on failure, `rollout undo` returns the Deployment to the previous revision and the job is marked as failed.

> Tip! `sed` is fine for a single image, but as the number of manifests grows, Kustomize is a cleaner option: declare the image in a `kustomization.yaml`, update it with `kustomize edit set image` (standalone `kustomize` CLI) in the workflow, and apply with `kubectl apply -k k8s/`, which has Kustomize built in.

## Verifying and Operating the Deployment

Once the pipeline is green, a few commands help you confirm what is running:

```bash
# Which image is the Deployment using?
kubectl -n fastapi-demo get deployment fastapi-demo \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Pod status and readiness
kubectl -n fastapi-demo get pods -l app=fastapi-demo

# Revision history of the Deployment
kubectl -n fastapi-demo rollout history deployment/fastapi-demo

# Stream logs from all replicas
kubectl -n fastapi-demo logs -l app=fastapi-demo -f --prefix
```

Rolling back manually is also a single command, and you can target a specific revision from the history:

```bash
kubectl -n fastapi-demo rollout undo deployment/fastapi-demo
kubectl -n fastapi-demo rollout undo deployment/fastapi-demo --to-revision=3
```

Remember that the next push to `main` will deploy again. If a commit is truly broken, the cleanest fix is to revert it in Git so the repository and the cluster agree again.

### Scaling Automatically

With CPU requests defined, adding a HorizontalPodAutoscaler is straightforward. It requires the Kubernetes Metrics Server to be installed in the cluster:

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-demo
  namespace: fastapi-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-demo
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

If you adopt an HPA, remove `replicas` from the Deployment manifest. Otherwise every `kubectl apply` resets the replica count to three, fighting against the autoscaler.

## Best Practices and Next Steps

- **Never deploy `latest`.** Mutable tags make rollbacks ambiguous and can leave different Pods running different code. Always deploy by SHA or by a semantic version tag.
- **Keep probes cheap.** Probes run every few seconds on every Pod. They should not run heavy queries or call slow external services.
- **Handle shutdown gracefully.** Kubernetes sends `SIGTERM` and waits `terminationGracePeriodSeconds` (30 seconds by default) before killing the container. Uvicorn finishes in-flight requests on `SIGTERM`, so make sure long requests fit inside that window.
- **Scan your images.** Add a vulnerability scanner such as Trivy to the `build` job, as shown in the [GitHub Actions post](/posts/github-actions-cicd/), and fail the build on critical findings.
- **Separate environments.** Add a `staging` namespace and environment that deploys automatically, and keep `production` behind a manual approval.
- **Consider GitOps.** As the number of services grows, tools like Argo CD or Flux invert the model: CI only pushes images and updates manifests in Git, and an in-cluster agent applies them. This removes cluster credentials from CI entirely.

## Conclusion

We took a FastAPI service from source code to a running, self-healing Kubernetes Deployment with a fully automated pipeline. The Dockerfile produces a small, non-root image; the manifests add health probes, resource requests, security settings and a zero-downtime rolling update strategy; and the GitHub Actions workflow ties it all together by testing every change, publishing immutable images to GHCR and rolling them out with an automatic rollback on failure.

None of the individual pieces is complicated, but together they give you a deployment process that is repeatable, auditable and safe to run many times a day. From here, the natural next steps are templating with Kustomize or Helm, adding a staging environment, and eventually moving to a GitOps workflow as your platform grows.

## Resources

- [FastAPI Documentation: FastAPI in Containers](https://fastapi.tiangolo.com/deployment/docker/)
- [Kubernetes Documentation: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Documentation: Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [GitHub Docs: Publishing Docker images](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)
- [GitHub Docs: Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [kind: Kubernetes in Docker](https://kind.sigs.k8s.io/)
