# DSO202 — Practical 1: Local Kubernetes Cluster with kind

## Purpose

This repository sets up a three-node local Kubernetes cluster using `kind`
(Kubernetes IN Docker) and deploys a static nginx web server through several
stages of Kubernetes objects: a Namespace, a ResourceQuota/LimitRange pair, a
standalone Pod, a Deployment (with rolling updates and rollback), and two
Services (ClusterIP and NodePort). It fulfils Practical 1 of 10 for DSO202 —
Scaling, Orchestration, Monitoring & Observability, covering descriptor
sections Unit I 1.1–1.5 and Learning Outcomes LO1, LO2, LO3, and the
multi-tenancy half of LO5.

## Software Versions Used

| Software | Version |
|---|---|
| OS | Kali Linux, kernel 6.18.12+kali-amd64, Debian 13 (trixie) base |
| Docker | [ fill in: `docker info --format '{{.ServerVersion}}'` ] |
| kind | v0.32.0 |
| kubectl | v1.36.0 |
| Kubernetes (cluster) | v1.36.1 |
| Container runtime | containerd 2.3.1 |

## Repository Structure

```
practical-01/
├── README.md
├── cluster/
│   └── kind-cluster.yaml
├── manifests/
│   ├── 00-namespace.yaml
│   ├── 01-quota-and-limits.yaml
│   ├── 02-pod-web.yaml
│   ├── 03-deployment-web.yaml
│   ├── 04-service-clusterip.yaml
│   ├── 05-service-nodeport.yaml
│   └── 06-pod-client.yaml
├── evidence/
└── report/
    └── practical-01-report.md
```

## Prerequisites

- Docker Engine or Docker Desktop ≥ 24.0
- kind ≥ v0.32.0
- kubectl v1.35–v1.37 (within one minor version of the cluster)

Verify before starting:

```bash
docker info --format '{{.ServerVersion}} {{.OperatingSystem}}'
kind version
kubectl version --client
```

## How to Rebuild This Practical From an Empty Machine

1. **Create the cluster.**
   ```bash
   kind create cluster --config cluster/kind-cluster.yaml
   ```
   First run downloads a ~1 GB node image; later runs take roughly a minute.

2. **Confirm the cluster is up.**
   ```bash
   kind get clusters
   kubectl get nodes
   kubectl get pods -n kube-system -o wide
   ```
   All three nodes should report `Ready`, and every kube-system Pod should be
   `Running` with `1/1` containers ready.

3. **Set the working namespace.**
   ```bash
   kubectl apply -f manifests/00-namespace.yaml
   kubectl config set-context --current --namespace=dso202-practical-01
   ```

4. **Apply the ResourceQuota and LimitRange.**
   ```bash
   kubectl apply -f manifests/01-quota-and-limits.yaml
   ```

5. **Apply the remaining manifests** (or apply the whole directory at once —
   `kubectl apply -f` processes a directory in lexical filename order, which
   is why the files are numbered):
   ```bash
   kubectl apply -f manifests/
   ```

6. **Verify everything is running.**
   ```bash
   kubectl get all
   kubectl get endpointslice
   ```

7. **Test the application.**
   ```bash
   # From inside the cluster
   kubectl exec client-pod -- wget -qO- http://web-clusterip

   # From the host, via NodePort
   curl -s http://localhost:30080
   ```

8. **Confirm the cluster matches the repository** (should print nothing but
   the confirmation line):
   ```bash
   kubectl diff -f manifests/ && echo "cluster matches repository"
   ```

## Cleanup

```bash
kubectl delete -f manifests/
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202
```

## Known Issues Encountered and Fixed in This Repository

The manifests and commands in the officially distributed HackMD guide
contained several inconsistencies against the actual companion manifest
file. These were identified and corrected during the practical; full detail
is in `report/practical-01-report.md` under Reflection. Summary:

1. **Namespace name mismatch** — some guide commands and one manifest file
   referenced `dso202-practical`, while the actual Namespace object (per the
   companion file) is `dso202-practical-01`. All files/commands in this repo
   use `dso202-practical-01` consistently.
2. **File-numbering mismatch** — some guide steps referenced manifest
   filenames (e.g. `06-deployment-web.yaml`, `09-pod-client.yaml`) that don't
   match the actual Listing index. This repo uses the filenames given in the
   companion file's Listing index (`00`–`06`).
3. **Missing readinessProbe** — `manifests/03-deployment-web.yaml` as
   originally distributed had no `readinessProbe`, which broke the
   readiness-gating demonstration. A `readinessProbe` and `livenessProbe`
   were added (HTTP GET on `/index.html`, port `http`,
   `periodSeconds: 5`, `failureThreshold: 2`).
4. **Standalone Pod polluting Service endpoints** — `manifests/02-pod-web.yaml`
   (Stage 4) and `manifests/03-deployment-web.yaml` (Stage 5) both label
   their Pods `app: web, tier: frontend`, so `web-clusterip`'s selector
   matched `web-pod` in addition to the Deployment's three replicas. This
   was only noticed when a load-balancing test returned four distinct Pod
   names instead of three. Resolved by deleting `web-pod` once its Stage 4
   teaching purpose was fulfilled.
