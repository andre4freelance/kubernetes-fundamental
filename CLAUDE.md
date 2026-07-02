# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A hands-on learning collection of standalone Kubernetes manifests, one directory per resource kind (`pod/`, `deployment/`, `service/`, `configmap/`, `secret/`, `pvc/`, `hpa/`, `ingress/`, `replicaset/`, `resourcequota/`, `limitrange/`). There is no application source, build system, or test suite — the "code" is the YAML, and the workflow is applying it to a cluster with `kubectl` and observing behavior.

Fork of `ajidiyantoro/kubernetes-fundamental` (git remote `upstream`).

## Applying manifests

```
kubectl apply  -n learning -f deployment/deployment.yaml
kubectl delete -n learning -f deployment/deployment.yaml
```

Manifests declare **no `namespace`** — they land in whatever namespace is current or passed with `-n`. Scope all learning work to a dedicated namespace (this setup uses **`learning`**), never a shared/system one.

## Cluster access & safety (IMPORTANT)

- The learning cluster is a **Rancher / RKE2** cluster reached through an **isolated kubeconfig** kept **out of git** (context `local`). The kubeconfig path and any cluster endpoints are machine-local — see local memory, not this file.
- **Always run `kubectl config current-context` before any mutating command** (`apply`/`delete`/`scale`/…). Never point learning applies at a production context. Contexts can change between machines and sessions.
- A `kubernetes` MCP server may be configured in **non-destructive mode** (delete operations disabled) — use `kubectl` directly for deletes.
- **Never commit secrets:** kubeconfigs, tokens, certs, and this project's real cluster identifiers stay in `.gitignore` (the repo is a public fork).

## Conventions

- Most workloads are named `nginx-demo`, labeled `app: nginx`, image `nginx:1.25`. Exceptions that matter: `deployment/deployment-secret-1.yaml` is `nginx-deploy`; `pod/pod-with-cm.yaml` and `pod/pod-with-secret.yaml` are `demo-app`.
- Because Deployment, ReplicaSet, bare Pod, Service, Ingress, and HPA all share the `nginx-demo` name and `app: nginx` selector, apply **one workload type at a time** — applying several together creates overlapping objects fighting over the same pods.
- Service is `ClusterIP` on port `8080` → `targetPort 80`; Ingress and HPA reference the `nginx-demo` Service/Deployment.
- Many files include a **commented-out alternative** demonstrating a second technique (e.g. `env`/`*KeyRef` shown with `envFrom`/`*Ref` commented; `pod/pod-with-probe.yaml` has readiness active, liveness commented). Uncomment to explore the variant.
- When creating new practice cases, prefer common **production-style English names** (e.g. `frontend`, `api`, `app-config`, `db-credentials`).

## Cross-file dependencies (apply the referenced object first)

- `deployment/deployment-configmap-*.yaml` → needs `configmap/configmap-*.yaml`.
- `deployment/deployment-secret-*.yaml` → needs a matching Secret. **Note the referenced name may not match the provided files** (e.g. `deployment-secret-1.yaml` references `app-secret`, but the secret manifests are `app-secret-1`..`app-secret-4`) — verify/adjust names before applying.
- `pod/pod-with-cm.yaml` → ConfigMap `app-config-2`; `pod/pod-with-secret.yaml` → Secret `app-secret-1`.
- `pod/pod-with-pvc.yaml` → PVC `app-storage` (`pvc/pvc.yaml`); requires a default StorageClass to bind.
- `hpa/hpa.yaml` → Deployment `nginx-demo` + metrics-server installed.
- `ingress/ingress.yaml` → Service `nginx-demo:8080` + an ingress controller.

## Gotchas

- `pod/pod-with-probe.yaml` is actually a **Deployment**, not a Pod, despite its path.
- `pod/pod-with-pvc.yaml` uses `nginx:latest`; everything else pins `nginx:1.25`.

## Learning progress

See `LEARNING.md` for the structured learning log (portable across machines).
