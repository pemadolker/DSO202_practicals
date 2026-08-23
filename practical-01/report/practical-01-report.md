# DSO202 Practical 1 Report
### Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

**Student:** Pema Dolker    
**Student ID:** 02230294    
**Programme:** BE in Software Engineering   
**Date:** 18/8/26

---

## 1. Objective
 
A practical setup to deploy a three-node local Kubernetes cluster via
`kind` was used to deploy a static nginx webserver utilizing gradually
more complete Kubernetes resources like: namespace for multi-tenancy, a
ResourceQuota and LimitRange to control the resource consumption, a simple
Pod, a Deployment controlling a ReplicaSet of 3 pods, and two services,
ClusterIP and NodePort, to make the service accessible both from within and
from outside the cluster.
  
It covers from descriptors Unit I 1.1 (Kubernetes Architecture),
1.2.1 - 1.2.4 (Pods, ReplicaSets, Deployments, Services), 1.3.1 - 1.3.3 (kubectl
usage and troubleshooting), 1.4.1 (Workload Terminology), and 1.5.1/1.5.3 (Namespaces,
resource quotas, limit ranges).
 
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
 
### Stage 0 - Prerequisites
Docker, kind, and kubectl
 
### Stage 1 - Creating the Three-Node Cluster
The cluster was created using `kind create cluster --config
cluster/kind-cluster.yaml` and the cluster with three Docker containers were verified using `kind get clusters` and `docker ps` commands respectively. It was confirmed that kubectl is pointing to the cluster using `kubectl config current-context`.
 
![kind create cluster, get clusters, docker ps, current-context](../evidence/stage1-cluster-creation.png)
 
### Stage 2 – Inspecting the Cluster
Nodes, control-plane, and namespaces have been listed as instructed.
Verified that all the nodes – `control-plane`, `worker-node-1`,
and `worker-node-2` were `Ready`, and inspected the labels,
capacity, and allocatable resources using `kubectl describe node worker-node-1`.
 
![cluster-info, get nodes, get nodes -o wide, describe node](../evidence/stage2-cluster-inspect.png)
 
Verified that the control plane was running one copy of each component,
while `kube-proxy` and `kindnet` run one copy per node as DaemonSets.
 
![kubectl get pods -n kube-system -o wide](../evidence/stage2-kubesystem-pods.png)
 
Read one control-plane component's logs and listed namespaced vs.
cluster-scoped API resource types.
 
![kubectl logs kube-scheduler, kubectl api-resources](../evidence/stage2-scheduler-logs-apiresources.png)
 
### Stage 3 - Namespace, ResourceQuota, LimitRange
The namespace was created imperatively at first to compare (`dso202-scratch`,
after which it was deleted), and declaratively using `manifests/00-namespace.yaml`
in compliance with the guide.
 
**Discrepancy found:** The namespace is defined as `dso202-practical-01`
in the manifest, yet several commands from the guide reference it
as `dso202-practical` (without `-01`). Setting the default namespace
using the guide’s own command in step 4, and running
`kubectl describe

```
Error from server (NotFound): namespaces "dso202-practical" not found
```
 
Diagnosed by running `kubectl get namespace` to confirm the namespace that
actually existed, then correcting the context with:
 
```
kubectl config set-context --current --namespace=dso202-practical-01
```
 
![namespace create, dry-run, apply, set-context, quota apply, NotFound error, get namespaces](../evidence/stage3-namespace-mismatch-error.png)
 
Since the context was fixed, both the ResourceQuota and the LimitRange objects were checked using the `kubectl describe` command. The default values of the LimitRange were successfully inserted into a Pod which did not define any resources for itself (`limitrange-check`).

![describe resourcequota, describe limitrange, limitrange-check test](../evidence/stage3-quota-limitrange.png)
 
### Stage 4 - Pods
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
 
Label and annotation work including removal and addition of a runtime label,
and addition of an annotation to ensure that it is not selectable like a label.
 
![get pods --show-labels, label selector tests, label add/remove, annotate](../evidence/stage4-labels-selectors-annotations.png)
 
Went over `kubectl exec` commands (interactive shell and single command), `kubectl
port-forward` and `kubectl explain` for unknown parameters.

![exec into web-pod, port-forward, explain resources](../evidence/stage4-exec-portforward-explain.png)

![explain livenessProbe --recursive](../evidence/stage4-explain-livenessprobe.png)
 
### Stage 5 - Deployments
Created a manifest file of the Deployment object imperative approach for comparison
purpose, and then created `manifests/03-deployment-web.yaml` and understood its ownership
chain of Deployment -> ReplicaSet -> Pod.
 
![dry-run deployment yaml, apply, rollout status, ownership chain](../evidence/stage5-deployment-create-chain.png)
 
Verified ownership directly through the `ownerReferences`
field, verified that the scheduler distributed replicas to both worker
nodes, and showed self-healing by deleting one Pod and observing the
ReplicaSet create another replica almost immediately. Also scaled up
imperatively to five replicas as a comparison against the stated number of three replicas.
 
![ownerReferences, node placement, delete pod, self-healing, scale to 5](../evidence/stage5-selfheal-scaling.png)
 
Watched a rolling update from `nginx:1.30-alpine` to `nginx:1.31-alpine`, seeing
how the old ReplicaSet's Pods shut down only when new Pods came online.
 
![rollout watch: old ReplicaSet terminating as new one becomes ready](../evidence/stage5-rollout-watch.png)
 
Checked the revision history, examined the image in an older revision, and rolled back using
`kubectl rollout undo`.

![replicaset after update, rollout history, rollout undo](../evidence/stage5-rollout-history-rollback.png)
 
Successfully forced a failed rollout using the non-existent image version `nginx:9.99-does-not-exist`, and observed how the three live replicas weren't
deleted while the new Pod was stuck in `ImagePullBackOff` mode — proof
that `maxUnavailable: 0` protected against downtime during the failed
rollout. Rolled back.

 
![failed rollout: ImagePullBackOff alongside 3 healthy Pods, rollback](../evidence/stage5-failed-rollout.png)
 
**Second filename discrepancy found:** : The guide's Step 8 text (back
to the declared replica count) mentions `manifests/06-deployment-web.yaml`,
which does not exist under that name — Listing 5 is called
`03-deployment-web.yaml` according to Listing numbering of the companion
file. Fixed using the correct filename.

 
```
error: the path "manifests/06-deployment-web.yaml" does not exist
```
 
![06-deployment-web.yaml missing, corrected to 03-deployment-web.yaml](../evidence/stage5-filename-mismatch.png)
 
Finally, verified the exact match between the cluster and repository with
`kubectl diff`, which output nothing before the confirmation echo.
 
![kubectl diff printing nothing, "cluster matches manifest"](../evidence/stage5-diff-matches.png)
 
### Stage 6 - Services
Applied the ClusterIP Service (`manifests/04-service-clusterip.yaml`) and
the client Pod. The guide's Step 3 text mentions
`manifests/09-pod-client.yaml`, which does not exist under that name -
Listing 8 is called `06-pod-client.yaml`; fixed using the correct
filename.
 
![09-pod-client.yaml missing, corrected to 06-pod-client.yaml](../evidence/stage6-filename-mismatch.png)
 
Verified the Service could respond to HTTP requests properly within the
cluster through `client-pod`.
 
Two genuine problems were found and resolved in this stage.
 
**1. A Pod running separately was interfering with the Service's
endpoints.** The load balancing test (writing a different hostname to each
Pod and making nine requests) resulted in a split into four different Pod
names, instead of three — one of them was `web-pod`, the separate Pod from
Stage 4, not a Deployment replica. Cause: `manifests/02-pod-web.yaml` and
`manifests/03-deployment-web.yaml` use identical Pod labeling: `app:
web, tier: frontend` (as can be seen from the screenshot with Stage 4
labels shown above), while the ClusterIP Service filters by these two
criteria only, leaving no way to distinguish an unmanaged Pod from the
Deployment replica. Fixed by deleting `web-pod`, since its educational role
in Stage 4 was completed. The load balancing test was performed again after
this action and verified a clear, even distribution across three Deployment
Pods.
 
![clean load balancing results across exactly 3 Deployment Pods](../evidence/stage6-clean-loadbalance-final.png)
 
**2. Missing `readinessProbe`.** The test of readiness-based traffic
filtering (by deleting `index.html` from one Deployment Pod) produced no
effect initially — the Pod kept on reporting `1/1 Running` and still fully
present in the Service's EndpointSlice, because `manifests/03-deployment-web.yaml`
distributed with the assignment has neither `readinessProbe` nor
`livenessProbe`, contrary to the description in the guide's narrative
regarding readiness-based Pod removal. Added: 

 
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
 
Using the Deployment again triggered a redeployment (a change in Pod template
results in a change in the template hash, and therefore in a new
ReplicaSet). Testing the readiness of the new Pods revealed that the target
Pod was now dropped to `0/1 READY`. Looking into the per-address
`conditions` of the EndpointSlice (instead of simply looking at the number
of addresses in `-o wide`) revealed the truth behind this behavior: the
current `discovery.k8s.io/v1` API keeps the unready Pod's address in the
list, but marks it as `ready: false` (unlike in the older Endpoints API
which the guide's sample output is based upon, where the unready Pod's
address is simply removed). 

Testing for load balancing once more revealed
that the unready Pod did indeed receive none out of nine requests - traffic
was correctly excluded despite the presence of the address. After restoring
the file, it was confirmed that all three addresses now have `ready: true`.
The very same set of tests proved the guide's broken selector example,
`broken-service`, by producing an empty EndpointSlice exactly as it should
have done.
 
![readiness test: ready=false, load-balance excluding the broken Pod, restore, broken-service demo](../evidence/stage6-readiness-gating.png)
 
A NodePort service (`manifests/05-service-nodeport.yaml`) and a temporary
`lb-demo` Service were also tested: `curl http://localhost:30080` from the
host machine and from a worker container both accessed the application
correctly, while the `LoadBalancer`-type Service remained `<pending>` (since
there is no cloud provider in `kind`).
 

### Stage 7 - Cleanup
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
 
**What was difficult.** The most difficult aspect of this practical was not
any particular concept in Kubernetes, but the necessity not to believe what
the guide says at face value. The guide and its accompanying manifest file
conflict about a few details - namespace name and three manifest files' -
and the only way to identify these conflicts early on was to rely solely on
the index of files in the manifest file, and run all commands by
verifying them against the contents of that file first. It paid off
directly: the namespace-mismatch (Stages 3 and 4) and the manifest filename
errors (Stages 5 and 6) were fixed immediately due to the checking
approach, because the diagnostics for both classes of errors were simple
every time: read the exact error message, and check whether something
actually exists in the Kubernetes cluster/repository in contradiction with
the error message.

**Which error was met and how it was diagnosed, and why it mattered.**
 The
most important outcome of the practical work was a discovery of the lack
of the `readinessProbe` in `manifests/03-deployment-web.yaml`. The thing
that made it stand out among naming errors is that it was not a typo; it
was an actual difference between the claims of the guide in Stage 6 and
what the manifest could make actually happen (no pods could disappear from
the service's routable set upon failing the readiness probe since there were no probes at all). It was detected through a negative test result -
deleting `index.html` did nothing: Pod continued working and receiving
traffic after being put in state where readiness check should've failed. A
negative result like that can easily be mistaken for "I made a mistake,
doing this step," which is why a deliberate effort had to be made to
exclude this hypothesis and realize that the manifest was actually
incomplete, and only then the `readinessProbe` and `livenessProbe` were added to it.

The fix led to a second, unexpected discovery, which was potentially even more valuable than the first one: after implementing a working `readinessProbe` and ensuring the Pod fails the probe, the EndpointSlice shows the address of this Pod in its overview, suggesting that the fix didn't work. However, viewing the conditions for the Pod's IP address directly demonstrates the real state of affairs: the address is included in the EndpointSlice but marked `ready: false`. This is inconsistent with the deletion of address in the older sample output used by the guide. It means that the right way to test whether the fix works is not to "count the addresses" but to "check the `ready` condition on each IP address in the EndpointSlice, and conduct a load balancing test", which is more accurate reflection of Kubernetes Services' functionality and should translate seamlessly into future practicals with Services, readiness checks and rolling updates.
  
**What would be done differently.** The overlap between `web-pod` / Deployment labels in Stage 6 was an easily predictable error. The labels used for `web-pod` (Stage 4) and the Pod template for the Deployment (Stage 5) are exactly the same (`app: web, tier: frontend`) by definition: there are no distinguishing factors anywhere in either manifest, and any Service targeting those labels will always include both of them. This was clear upon re-reading listings 4 and 5 side by side and could have been predicted before attempting Stage 6.

 
## 6. References

- DSO202 Practical 1 Guide (HackMD, sarojsanyasi)
- DSO202 Practical 1 Companion Manifest File (HackMD, sarojsanyasi)
- Kubernetes official documentation - EndpointSlices 
- Kubernetes official documentation - Configure Liveness, Readiness and Startup Probes 
- kind documentation - https://kind.sigs.k8s.io/