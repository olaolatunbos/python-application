# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A minimal Flask web service with two endpoints:
- `GET /api/v1/info` — returns current time, hostname, and a hello-world message
- `GET /api/v1/healthz` — returns `{"status": "up"}` (used as the liveness/readiness probe)

The app runs on port 5000. It is deployed to Kubernetes via a GitOps pipeline: GitHub Actions builds and pushes the Docker image to Amazon ECR Public, then updates `charts/python-application/values.yaml` with the new image tag, and ArgoCD syncs the change to the cluster.

## Running locally

```bash
pip install -r requirements.txt
python src/app.py
```

## Docker

```bash
docker build -t python-application .
docker run -p 5000:5000 python-application
```

## CI/CD pipeline

The workflow in [.github/workflows/cicd.yaml](.github/workflows/cicd.yaml) triggers on pushes to `main` that touch files under `src/`:

1. **CI job** — builds and pushes the image to `public.ecr.aws/r1j8z0t4/idp/python-application:<short-sha>` using AWS credentials from repository secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).
2. **CD job** (runs on a self-hosted runner with cluster access) — updates `image.tag` in `charts/python-application/values.yaml` using `yq`, commits the change, then uses the ArgoCD CLI to sync the `python-application` ArgoCD app. The ArgoCD password is stored in the `ARGOCD_PASSWORD` secret.

The CD job auto-creates the ArgoCD app if it does not exist, pointing at `charts/python-application` on the `main` branch, deploying into the `prod` namespace.

## Kubernetes / Helm

- Raw manifests are in [k8s/](k8s/) (not actively used by the pipeline — the Helm chart is the canonical deployment).
- The Helm chart lives in [charts/python-application/](charts/python-application/). The image tag in `values.yaml` is the only value the pipeline mutates; everything else is stable.
- ArgoCD itself is configured via [charts/argocd/values-argo.yaml](charts/argocd/values-argo.yaml) (HA disabled, single replicas, nginx ingress at `argocd.test.com`).
- The app is exposed at `app.olaolat.com` via an nginx ingress class.

## Backstage / TechDocs

`catalog-info.yaml` registers this service in Backstage as a `Component` of type `service`. `mkdocs.yaml` + `docs/index.md` provide the TechDocs source (`backstage.io/techdocs-ref: dir:.`).
