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
(namespaces, resource quotas, limit ranges) - meeting Learning Outcomes LO1,
LO2, LO3, and the multi-tenancy half of LO5.

## 2. Environment

| Item | Value |
|---|---|
| Operating system | Kali Linux, kernel 6.18.12+kali-amd64, Debian 13 (trixie) |
| Docker version | [ fill in — run `docker info --format '{{.ServerVersion}}'` ] |
| kind version | v0.32.0 |
| kubectl version | v1.36.0 |
| Cluster Kubernetes version | v1.36.1 |
| Container runtime | containerd 2.3.1 |



## 3. Procedure and Observations

### Stage 0 — Prerequisites
[ Fill in from your own memory: confirm Docker, kind, and kubectl were
verified before cluster creation. Only include detail here that you can
personally stand behind if asked. ]

### Stage 1 — Creating the Three-Node Cluster
[ Fill in: confirm whether cluster creation succeeded on the first attempt
or required troubleshooting. Only record what genuinely happened. ]

📷 *[Insert screenshot: `kind create cluster` output, `kind get clusters`,
`kubectl config current-context`]*



### Stage 2 — Inspecting the Cluster
Listed nodes, control-plane components, and namespaces as directed.
Confirmed all three nodes (`control-plane`, `worker-node-1`,
`worker-node-2`) reported `Ready`, and that `etcd`, `kube-apiserver`,
`kube-controller-manager`, and `kube-scheduler` each appeared once on the
control-plane node, while `kube-proxy` and `kindnet` appeared once per node
as DaemonSets.

📷 *[Insert screenshot: `kubectl get nodes -o wide`]*
📷 *[Insert screenshot: `kubectl get pods -n kube-system -o wide`]*

### Stage 3 — Namespace, ResourceQuota, LimitRange
Created the namespace declaratively from `manifests/00-namespace.yaml`.

**Discrepancy found:** the manifest defines the namespace as
`dso202-practical-01`, but several guide commands reference
`dso202-practical` (without the `-01` suffix). Running `kubectl describe
resourcequota dso202-quota` with the kubectl context defaulting to the
wrong name produced:

```
Error from server (NotFound): namespaces "dso202-practical" not found
```

Diagnosed by running `kubectl get namespace` to confirm the namespace that
actually existed, then correcting the context with:

```
kubectl config set-context --current --namespace=dso202-practical-01
```

📷 *[Insert screenshot: the `NotFound` error]*
📷 *[Insert screenshot: `kubectl get namespace` showing `dso202-practical-01`]*


The ResourceQuota and LimitRange were then applied and verified with
`kubectl describe`. The LimitRange's default values were confirmed to be
injected into a Pod that declared no resources of its own, exactly as the
guide describes.

📷 *[Insert screenshot: `kubectl describe resourcequota dso202-quota`]*
📷 *[Insert screenshot: `kubectl describe limitrange dso202-limits`]*

### Stage 4 — Pods
Created a Pod both imperatively (`kubectl run web-imperative ...`) and
declaratively (`manifests/02-pod-web.yaml`), and compared the two.

Applying `manifests/02-pod-web.yaml` initially failed with the same
namespace mismatch as Stage 3:

```
Error from server (NotFound): namespaces "dso202-practical" not found
```

This time the cause was inside the manifest itself — Listing 4 as
distributed hardcodes `namespace: dso202-practical`. Corrected by editing
the manifest to `dso202-practical-01`, after which `kubectl apply`
succeeded. Worked through labels, annotations, `kubectl exec`, `kubectl
logs`, and `kubectl port-forward` without further issues.

📷 *[Insert screenshot: corrected `apply` succeeding — `pod/web-pod created`]*
📷 *[Insert screenshot: `kubectl get pod web-pod -o wide --show-labels`]*

### Stage 5 — Deployments
Applied `manifests/03-deployment-web.yaml` (note: the guide's Step 8 text
references `manifests/06-deployment-web.yaml`, which does not exist under
that name in the companion file's Listing index — Listing 5 is saved as
`03-deployment-web.yaml`; corrected by using the actual filename).

Observed the Deployment → ReplicaSet → Pod ownership chain, demonstrated
self-healing by deleting a Pod and watching the ReplicaSet recreate it
within seconds, scaled imperatively to 5 replicas and then returned to the
declared state of 3 by re-applying the manifest, performed a rolling update
from `nginx:1.30-alpine` to `nginx:1.31-alpine`, rolled back with `kubectl
rollout undo`, and deliberately triggered a failed rollout using the
non-existent image tag `nginx:9.99-does-not-exist`.

📷 *[Insert screenshot: `kubectl get deployment,replicaset,pod -l app=web`]*
📷 *[Insert screenshot: Pod list immediately after deleting one Pod,
showing a replacement was created]*

Confirmed via `kubectl get pods -l app=web` that the three healthy replicas
were never removed during the stalled rollout — direct evidence that
`maxUnavailable: 0` prevented an outage.

📷 *[Insert screenshot: `ImagePullBackOff` Pod alongside 3 still-`Running` Pods]*

`kubectl rollout undo` produced a warning that the `last-applied-configuration`
annotation would go stale (a consequence of mixing `kubectl set image` with
`kubectl apply`); this resolved itself once the manifest was re-applied in
the final step, which `kubectl diff` then confirmed matched the live object
exactly.

📷 *[Insert screenshot: `kubectl diff -f manifests/03-deployment-web.yaml`
printing nothing, followed by `echo "cluster matches manifest"`]*

### Stage 6 — Services
Applied the ClusterIP Service (`manifests/04-service-clusterip.yaml`) and
the client Pod. The guide's Step 3 text references
`manifests/09-pod-client.yaml`, which does not exist under that name —
Listing 8 is saved as `06-pod-client.yaml`; corrected by using the actual
filename. Confirmed DNS resolution of the Service name from `client-pod`
and confirmed the Service answered HTTP requests correctly.

📷 *[Insert screenshot: `kubectl exec client-pod -- nslookup web-clusterip`]*
📷 *[Insert screenshot: `kubectl exec client-pod -- wget -qO- http://web-clusterip`]*

Two genuine problems were found and resolved in this stage.

**1. A standalone Pod was polluting the Service's endpoints.** The
load-balancing test (writing a distinct hostname to each Pod and sending
nine requests) returned results split across four distinct Pod names
instead of three — one of them was `web-pod`, the standalone Pod from
Stage 4, not a Deployment replica. Cause: `manifests/02-pod-web.yaml` and
`manifests/03-deployment-web.yaml` both label their Pods `app: web,
tier: frontend`, and the ClusterIP Service selects on exactly those two
keys with no way to distinguish an unmanaged Pod from a Deployment-owned
replica. Confirmed with:

```
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip -o wide
kubectl get pod web-pod --show-labels
```

which showed four endpoints and an exact label match.

📷 *[Insert screenshot: EndpointSlice with 4 addresses]*
📷 *[Insert screenshot: `web-pod --show-labels` matching the Service selector]*
📷 *[Insert screenshot: load-balancing test showing `web-pod` receiving traffic]*

Resolved by deleting `web-pod` via `kubectl delete -f
manifests/02-pod-web.yaml`, since its teaching purpose from Stage 4 was
already fulfilled. Re-ran the load-balancing test afterward and confirmed a
clean split across exactly the three Deployment replicas.

📷 *[Insert screenshot: clean load-balancing result across 3 Pods only]*

**2. Missing `readinessProbe`.** Testing readiness-based traffic gating (by
deleting `index.html` from one Deployment Pod) showed no change: the Pod
continued to report `1/1 Running` and remained fully in the Service's
EndpointSlice. Checked directly with:

```
grep -i probe manifests/03-deployment-web.yaml
```

which returned nothing — confirming the container specification in
`manifests/03-deployment-web.yaml` (Listing 5) had no `readinessProbe` or
`livenessProbe` defined at all, despite the guide's narrative describing
readiness-based removal.

📷 *[Insert screenshot: empty `grep -i probe` output]*
📷 *[Insert screenshot: Pod still `1/1` and still fully in EndpointSlice
after `index.html` was deleted]*

Added:

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

Re-applying `manifests/03-deployment-web.yaml` triggered a new rollout (a
changed Pod template produces a new template hash, and therefore a new
ReplicaSet). Repeating the readiness test on the new Pods showed the Pod
correctly drop to `0/1 READY` with a recorded `Warning Unhealthy: Readiness
probe failed: HTTP probe failed with statuscode: 404` event.

📷 *[Insert screenshot: `kubectl describe pod` showing the `Unhealthy` events]*

Checking the EndpointSlice as YAML (`kubectl get endpointslice ... -o
yaml`) rather than the summary table (`-o wide`) showed the true mechanism:
the current `discovery.k8s.io/v1` API retains the unready Pod's address in
the list but marks it with a per-address condition `ready: false`, rather
than removing the address outright as in the older Endpoints API the
guide's sample output was written against.

📷 *[Insert screenshot: full EndpointSlice YAML showing `ready: false` on the
broken Pod's entry]*

A repeat load-balancing test confirmed the unready Pod received zero of
nine requests, proving traffic was correctly excluded despite the address
still being listed.

📷 *[Insert screenshot: load-balancing test with 0 requests reaching the
unready Pod]*

Both the NodePort Service (`manifests/05-service-nodeport.yaml`) and a
temporary `broken-service`/`lb-demo` were also tested: `curl
http://localhost:30080` from the host and from inside a worker container
both reached the application correctly, the broken-selector demo produced
an empty EndpointSlice exactly as expected, and a `LoadBalancer`-type
Service correctly stayed `<pending>`, since `kind` has no cloud provider to
fulfil the request.

📷 *[Insert screenshot: `curl http://localhost:30080` from the host]*
📷 *[Insert screenshot: `docker exec dso202-worker curl http://localhost:30080`]*
📷 *[Insert screenshot: `broken-service` with an empty EndpointSlice]*
📷 *[Insert screenshot: `lb-demo` showing `EXTERNAL-IP <pending>`]*

### Stage 7 — Cleanup
Captured final evidence (`kubectl get all -o wide`, `resourcequota`,
`limitrange`, `endpointslice`, `nodes`). Final state confirmed: 3 healthy
Deployment Pods plus `client-pod` (4 total, matching
`resourcequota/dso202-quota`'s reported `pods: 4/20`), 2 Services
(`web-clusterip`, `web-nodeport`, matching `count/services: 2/5`), and 4
ReplicaSets retained under `revisionHistoryLimit: 5` — one active at 3/3/3
and three scaled to 0/0/0, preserved for rollback.

📷 *[Insert screenshot: final `kubectl get all -o wide`]*
📷 *[Insert screenshot: final `resourcequota` describe output]*

Deleted workload objects declaratively via `kubectl delete -f
manifests/<file>` for each manifest actually present in the repository,
confirmed the namespace held only the ResourceQuota and LimitRange
afterward, then rebuilt everything from the repository with `kubectl apply
-f manifests/` to demonstrate reproducibility, before deleting the cluster
with `kind delete cluster --name dso202`.

📷 *[Insert screenshot: `kubectl apply -f manifests/` rebuild output]*
📷 *[Insert screenshot: `kind delete cluster` and `kind get clusters` showing
no clusters remain]*

## 4. Analysis

[ Insert the review questions from guide section 21 here, with your own
answers. The official question set was not available in the materials used
for this session — confirm with the lecturer or course LMS before
submission, since this section should answer the actual assigned questions
rather than a self-set substitute. ]

## 5. Reflection

**What was difficult.** The hardest part of this practical was not any
single Kubernetes concept in isolation, but the discipline of not trusting
the guide's text at face value. The guide and its companion manifest file
disagreed with each other in several places — a namespace name, several
manifest filenames — and the only way to catch these before they wasted
time was to treat the companion file's Listing index as the single source
of truth and check every command against it before running it. That habit
paid off directly: the two namespace-mismatch errors and the two filename
errors were each resolved in under a minute once the checking habit was in
place, because the diagnostic step was always the same — read the error
message literally, then verify what actually exists in the cluster or
repository rather than assuming the guide's prose was correct.

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
had to rule that out deliberately: confirming the delete succeeded, waiting
longer than the guide's suggested interval in case of a delay, and only
then checking the manifest itself with `grep -i probe`, which returned
nothing. That confirmed the gap was in the manifest, not in the procedure.

Fixing it produced a second, unplanned finding that was arguably more
valuable than the first: once a working `readinessProbe` was added and the
Pod correctly failed it, the EndpointSlice still listed the Pod's address
in `kubectl get endpointslice -o wide`, which looked at first like the fix
hadn't worked. Inspecting the object as full YAML rather than the summary
table showed the address was retained but tagged `ready: false` under a
per-address `conditions` block — the current `discovery.k8s.io/v1`
EndpointSlice API's actual behaviour, which differs from the address-removal
behaviour implied by the guide's older-style sample output. This meant the
correct way to verify the fix was not "count the addresses" but "read the
`ready` condition on each address, and confirm the routing behaviour
directly with a load test" — which is a more accurate mental model of how
Kubernetes Services actually work than the guide's simplified description,
and one that will transfer directly to later practicals involving Services,
readiness, and rolling updates.

**What would be done differently.** The `web-pod` / Deployment label
overlap in Stage 6 could have been anticipated rather than discovered by
surprise. `web-pod` (Stage 4) and the Deployment's Pod template (Stage 5)
share identical `app: web, tier: frontend` labels by construction — nothing
in either manifest distinguishes "a standalone teaching Pod" from "a
Deployment-managed replica," so any Service selecting on those two keys was
always going to catch both. Recognising this before starting Stage 6 —
simply by re-reading the label sections of Listings 4 and 5 side by side —
would have avoided a confusing intermediate load-balancing result and saved
the time spent re-diagnosing it. More generally, the lesson is that a label
selector's correctness should be checked against every Pod in the
namespace that could possibly match it, not just the Pods it was written
with in mind.

**What remains unclear.** [ Insert one genuine thing you're still unsure
about, in your own words — for example, the exact timing relationship
between a readiness probe's `failureThreshold`/`periodSeconds` and how
quickly kube-proxy on each node actually stops forwarding to a
newly-unready address, since the EndpointSlice condition update and
kube-proxy's own local rule update are two separate steps that this
practical didn't give a way to measure directly. ]

## 6. References

- DSO202 Practical 1 Guide (HackMD, sarojsanyasi)
- DSO202 Practical 1 Companion Manifest File (HackMD, sarojsanyasi)
- Kubernetes official documentation — EndpointSlices — [ add specific URL and access date if consulted ]
- Kubernetes official documentation — Configure Liveness, Readiness and Startup Probes — [ add specific URL and access date if consulted ]
- kind documentation — https://kind.sigs.k8s.io/ — [ add access date ]