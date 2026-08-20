# DSO202 Practical 1 Report
### Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

**Student:** Pema Dolker    
**Student ID:** 02230294    
**Programme:** BE in Software Engineering   
**Date:** 18/8/26

---

## 1. Objective
 
This practical set up a three-node local Kubernetes cluster using `kind`
and deployed a static nginx web server through progressively more complete
Kubernetes objects: a Namespace for multi-tenancy, a ResourceQuota and
LimitRange to govern resource consumption, a bare Pod, a Deployment managing
a ReplicaSet of three Pods, and two Services (ClusterIP and NodePort) to
expose the application both inside and outside the cluster.
 
It covers descriptor sections Unit I 1.1 (Kubernetes architecture), 1.2.1–
1.2.4 (Pods, ReplicaSets, Deployments, Services), 1.3.1–1.3.3 (kubectl
operation and troubleshooting), 1.4.1 (workload terminology), and 1.5.1/1.5.3
(namespaces, resource quotas, limit ranges) — meeting Learning Outcomes LO1,
LO2, LO3, and the multi-tenancy half of LO5.
 
## 2. Environment
 
| Item | Value |
|---|---|
| Operating system | Kali Linux, kernel 6.18.12+kali-amd64, Debian 13 (trixie) |
| Docker version |  29.3.1 |
| kind version | v0.32.0 |
| kubectl version | v1.36.0 |
| Cluster Kubernetes version | v1.36.1 |
| Container runtime | containerd 2.3.1 |
 
 
## 3. Procedure and Observations
 
### Stage 0 — Prerequisites
Docker, kind, and kubectl
 
### Stage 1 — Creating the Three-Node Cluster
Created the cluster from `cluster/kind-cluster.yaml` with `kind create
cluster --config cluster/kind-cluster.yaml`. Confirmed the cluster and its
three Docker containers with `kind get clusters` and `docker ps`, and
confirmed kubectl was pointed at the new cluster with `kubectl config
current-context`.
 
![kind create cluster, get clusters, docker ps, current-context](../evidence/stage1-cluster-creation.png)
 
### Stage 2 — Inspecting the Cluster
Listed nodes, control-plane components, and namespaces as directed.
Confirmed all three nodes (`control-plane`, `worker-node-1`,
`worker-node-2`) reported `Ready`, and read `kubectl describe node
worker-node-1` to inspect its labels, capacity, and allocatable resources.
 
![cluster-info, get nodes, get nodes -o wide, describe node](../evidence/stage2-cluster-inspect.png)
 
Confirmed the control-plane components appeared once each on the
control-plane node, while `kube-proxy` and `kindnet` appeared once per node
as DaemonSets.
 
![kubectl get pods -n kube-system -o wide](../evidence/stage2-kubesystem-pods.png)
 
Read one control-plane component's logs and listed namespaced vs.
cluster-scoped API resource types.
 
![kubectl logs kube-scheduler, kubectl api-resources](../evidence/stage2-scheduler-logs-apiresources.png)
 
### Stage 3 — Namespace, ResourceQuota, LimitRange
Created the namespace imperatively first for comparison (`dso202-scratch`,
then deleted), then declaratively from `manifests/00-namespace.yaml`.
 
**Discrepancy found:** the manifest defines the namespace as
`dso202-practical-01`, but several guide commands reference
`dso202-practical` (without the `-01` suffix). Setting the kubectl context
default namespace per the guide's own Step 4 command, then running
`kubectl describe resourcequota dso202-quota`, produced:
 
```
Error from server (NotFound): namespaces "dso202-practical" not found
```
 
Diagnosed by running `kubectl get namespace` to confirm the namespace that
actually existed, then correcting the context with:
 
```
kubectl config set-context --current --namespace=dso202-practical-01
```
 
![namespace create, dry-run, apply, set-context, quota apply, NotFound error, get namespaces](../evidence/stage3-namespace-mismatch-error.png)
 
With the context corrected, the ResourceQuota and LimitRange were verified
with `kubectl describe`. The LimitRange's default values were confirmed to
be injected into a Pod that declared no resources of its own
(`limitrange-check`), exactly as the guide describes.
 
![describe resourcequota, describe limitrange, limitrange-check test](../evidence/stage3-quota-limitrange.png)
 
### Stage 4 — Pods
Created a Pod both imperatively (`kubectl run web-imperative ...`) and
declaratively (`manifests/02-pod-web.yaml`), and compared the two.
 
Applying `manifests/02-pod-web.yaml` initially failed with the same
namespace mismatch as Stage 3:
 
```
Error from server (NotFound): error when creating "manifests/02-pod-web.yaml": namespaces "dso202-practical" not found
```
 
This time the cause was inside the manifest itself — Listing 4 as
distributed hardcodes `namespace: dso202-practical`. Corrected by editing
the manifest to `dso202-practical-01`.
 
![manifest showing corrected namespace field](../evidence/stage4-manifest-content-fix.png)
 
![apply failing then succeeding after the fix](../evidence/stage4-pod-apply-fix.png)
 
Worked through labels, annotations, and label selectors — including
adding and removing a runtime label, and adding an annotation to confirm it
cannot be selected on the way a label can.
 
![get pods --show-labels, label selector tests, label add/remove, annotate](../evidence/stage4-labels-selectors-annotations.png)
 
Also worked through `kubectl exec` (interactive shell and single command),
`kubectl port-forward`, and `kubectl explain` for unfamiliar fields.
 
![exec into web-pod, port-forward, explain resources](../evidence/stage4-exec-portforward-explain.png)

![explain livenessProbe --recursive](../evidence/stage4-explain-livenessprobe.png)
 
### Stage 5 — Deployments
Generated a Deployment manifest imperatively for comparison, then applied
`manifests/03-deployment-web.yaml` and observed the Deployment → ReplicaSet
→ Pod ownership chain.
 
![dry-run deployment yaml, apply, rollout status, ownership chain](../evidence/stage5-deployment-create-chain.png)
 
Confirmed the ownership relationship directly via `ownerReferences`,
confirmed the scheduler spread replicas across both worker nodes, and
demonstrated self-healing by deleting one Pod and watching the ReplicaSet
recreate a replacement within seconds. Also scaled imperatively to 5
replicas as a comparison against the declared state of 3.
 
![ownerReferences, node placement, delete pod, self-healing, scale to 5](../evidence/stage5-selfheal-scaling.png)
 
Watched a rolling update from `nginx:1.30-alpine` to `nginx:1.31-alpine` in
real time, observing the old ReplicaSet's Pods terminate only as new ones
became ready.
 
![rollout watch: old ReplicaSet terminating as new one becomes ready](../evidence/stage5-rollout-watch.png)
 
Read the revision history, inspected an earlier revision's image, and
rolled back with `kubectl rollout undo`.
 
![replicaset after update, rollout history, rollout undo](../evidence/stage5-rollout-history-rollback.png)
 
Deliberately triggered a failed rollout using the non-existent image tag
`nginx:9.99-does-not-exist`, and confirmed the three healthy replicas were
never removed while the new Pod sat in `ImagePullBackOff` — direct evidence
that `maxUnavailable: 0` prevented an outage during the stalled rollout.
Rolled back to recover.
 
![failed rollout: ImagePullBackOff alongside 3 healthy Pods, rollback](../evidence/stage5-failed-rollout.png)
 
**Second filename discrepancy found:** the guide's Step 8 text (returning
to the declared replica count) references `manifests/06-deployment-web.yaml`,
which does not exist under that name — Listing 5 is saved as
`03-deployment-web.yaml` per the companion file's Listing index. Corrected
by using the actual filename.
 
```
error: the path "manifests/06-deployment-web.yaml" does not exist
```
 
![06-deployment-web.yaml missing, corrected to 03-deployment-web.yaml](../evidence/stage5-filename-mismatch.png)
 
Finally, confirmed the cluster matched the repository exactly with
`kubectl diff`, which printed nothing before the confirmation echo.
 
![kubectl diff printing nothing, "cluster matches manifest"](../evidence/stage5-diff-matches.png)
 
### Stage 6 — Services
Applied the ClusterIP Service (`manifests/04-service-clusterip.yaml`) and
the client Pod. The guide's Step 3 text references
`manifests/09-pod-client.yaml`, which does not exist under that name —
Listing 8 is saved as `06-pod-client.yaml`; corrected by using the actual
filename.
 
![09-pod-client.yaml missing, corrected to 06-pod-client.yaml](../evidence/stage6-filename-mismatch.png)
 
Confirmed the Service answered HTTP requests correctly from inside the
cluster via `client-pod`.
 
Two genuine problems were found and resolved in this stage.
 
**1. A standalone Pod was polluting the Service's endpoints.** The
load-balancing test (writing a distinct hostname to each Pod and sending
nine requests) returned results split across four distinct Pod names
instead of three — one of them was `web-pod`, the standalone Pod from
Stage 4, not a Deployment replica. Cause: `manifests/02-pod-web.yaml` and
`manifests/03-deployment-web.yaml` both label their Pods `app: web,
tier: frontend` (visible directly in the Stage 4 labels screenshot above),
and the ClusterIP Service selects on exactly those two keys with no way to
distinguish an unmanaged Pod from a Deployment-owned replica. Resolved by
deleting `web-pod`, since its teaching purpose from Stage 4 was already
fulfilled. Re-ran the load-balancing test afterward and confirmed a clean,
even split across exactly the three Deployment replicas.
 
![clean load-balancing result across exactly 3 Deployment Pods](../evidence/stage6-clean-loadbalance-final.png)
 
**2. Missing `readinessProbe`.** Testing readiness-based traffic gating (by
deleting `index.html` from one Deployment Pod) initially showed no change —
the Pod continued to report `1/1 Running` and remained fully in the
Service's EndpointSlice, because `manifests/03-deployment-web.yaml` as
distributed had no `readinessProbe` or `livenessProbe` at all, despite the
guide's narrative describing readiness-based removal. Added:
 
```yaml
readinessProbe:
  httpGet:
    path: /index.html
    port: http
  initialDelaySeconds: 2
  periodSeconds: 5
  failureThreshold: 2
livenessProbe:
  tcpSocket:
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
```
 
Re-applying the Deployment triggered a new rollout (a changed Pod template
produces a new template hash, and therefore a new ReplicaSet). Repeating
the readiness test on the new Pods showed the target Pod correctly drop to
`0/1 READY`. Reading the EndpointSlice's per-address `conditions` (rather
than just the address count in `-o wide`) showed the true mechanism: the
current `discovery.k8s.io/v1` API retains the unready Pod's address in the
list but marks it `ready: false`, rather than removing the address outright
as in the older Endpoints API the guide's sample output was written
against. A repeat load-balancing test confirmed the unready Pod received
zero of nine requests — traffic was correctly excluded despite the address
still being listed. Restored the file and confirmed all three addresses
returned to `ready: true`. The same evidence run also demonstrated the
guide's broken-selector example (`broken-service`), which produced an empty
EndpointSlice exactly as expected.
 
![readiness test: ready=false, load-balance excluding the broken Pod, restore, broken-service demo](../evidence/stage6-readiness-gating.png)
 
Both the NodePort Service (`manifests/05-service-nodeport.yaml`) and a
temporary `lb-demo` Service were also tested: `curl http://localhost:30080`
from the host and from inside a worker container both reached the
application correctly, and a `LoadBalancer`-type Service correctly stayed
`<pending>`, since `kind` has no cloud provider to fulfil the request.
 
![NodePort apply, curl from host, curl from worker container, LoadBalancer pending](../evidence/stage6-nodeport-loadbalancer.png)
 
### Stage 7 — Cleanup
Captured final evidence to text files (`evidence/final-state-all.txt`,
`evidence/final-state-nodes.txt`, `evidence/final-state-events.txt`) via
`kubectl get all -o wide`, `resourcequota,limitrange,endpointslice -o wide`,
`get nodes -o wide`, and `get events --sort-by=.lastTimestamp`. Deleted
workload objects declaratively via `kubectl delete -f manifests/<file>` for
each manifest actually present in the repository.
 
![evidence capture to files, deleting workload objects](../evidence/stage7-evidence-capture-delete.png)
 
Confirmed the namespace held no workloads afterward (`kubectl get all`
returned nothing), then rebuilt everything from the repository with
`kubectl apply -f manifests/` to demonstrate reproducibility, and reset the
kubectl default namespace.
 
![empty namespace, rebuild from manifests/, resources recreated](../evidence/stage7-rebuild-reproducibility.png)
 
Finally deleted the cluster with `kind delete cluster --name dso202` and
confirmed with `kind get clusters` that no clusters remained.
 
![kind delete cluster, kind get clusters showing none remain](../evidence/stage7-cluster-deleted.png)
 
## 4. Analysis

## 5. Reflection
 
**What was difficult.** The hardest part of this practical was not any
single Kubernetes concept in isolation, but the discipline of not trusting
the guide's text at face value. The guide and its companion manifest file
disagreed with each other in several places — a namespace name, and three
separate manifest filenames — and the only way to catch these before they
wasted time was to treat the companion file's Listing index as the single
source of truth and check every command against it before running it. That
habit paid off directly: the namespace-mismatch errors (Stages 3 and 4) and
the filename errors (Stages 5 and 6) were each resolved quickly once the
checking habit was in place, because the diagnostic step was always the
same — read the error message literally, then verify what actually exists
in the cluster or repository rather than assuming the guide's prose was
correct.
 
**Which error was met and how it was diagnosed, and why it mattered.** The
most significant finding of the practical was the missing `readinessProbe`
in `manifests/03-deployment-web.yaml`. What made it different from the
naming mismatches is that it was not a typo — it was a real gap between
what the guide's Stage 6 narrative claimed would happen (a Pod disappearing
from the Service's routable set once its readiness check fails) and what
the distributed manifest actually made possible (nothing, because no
readiness check existed to fail). It surfaced as a negative result: after
deleting `index.html` from a running Pod, nothing changed — the Pod stayed
`1/1 Running` and kept receiving traffic indefinitely. A negative result
like that is easy to misread as "I did the step wrong," so the diagnosis
had to rule that out deliberately before concluding the manifest itself was
incomplete, and only then was a `readinessProbe`/`livenessProbe` pair
written and applied.
 
Fixing it produced a second, unplanned finding that was arguably more
valuable than the first: once a working `readinessProbe` was added and the
Pod correctly failed it, the EndpointSlice still listed the Pod's address
in the summary view, which looked at first like the fix hadn't worked.
Reading the object's per-address `conditions` directly showed the true
behaviour — the address is retained but tagged `ready: false` — which
differs from the address-removal behaviour implied by the guide's
older-style sample output. This meant the correct way to verify the fix
was not "count the addresses" but "read the `ready` condition on each
address, and confirm the routing behaviour directly with a load test,"
which is a more accurate mental model of how Kubernetes Services actually
work than the guide's simplified description, and one that should transfer
directly to later practicals involving Services, readiness, and rolling
updates.
 
**What would be done differently.** The `web-pod` / Deployment label
overlap in Stage 6 could have been anticipated rather than discovered by
surprise. `web-pod` (Stage 4) and the Deployment's Pod template (Stage 5)
share identical `app: web, tier: frontend` labels by construction — nothing
in either manifest distinguishes "a standalone teaching Pod" from "a
Deployment-managed replica," so any Service selecting on those two keys was
always going to catch both. Recognising this before starting Stage 6 —
simply by re-reading the label sections of Listings 4 and 5 side by side —
would have avoided a confusing intermediate load-balancing result. More
generally, the lesson is that a label selector's correctness should be
checked against every Pod in the namespace that could possibly match it,
not just the Pods it was written with in mind.
 
 
## 6. References
 
- DSO202 Practical 1 Guide (HackMD, sarojsanyasi)
- DSO202 Practical 1 Companion Manifest File (HackMD, sarojsanyasi)
- Kubernetes official documentation — EndpointSlices 
- Kubernetes official documentation — Configure Liveness, Readiness and Startup Probes 
- kind documentation — https://kind.sigs.k8s.io/ 