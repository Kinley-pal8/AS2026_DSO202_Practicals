# DSO202 — Practical 1

## Companion File: YAML Definitions

---

## How to Use This File

This file contains every configuration file referenced by `DSO202_Practical1_Guide.md`. Each listing states the exact path it must be saved to, the descriptor section it implements, and the command that applies it.

Three rules apply to all listings:

1. **Copy the comments as well as the configuration.** The comment blocks are part of the deliverable. *Code Comments* is worth 2 of the 15 practical-work marks, and a manifest submitted without comments cannot earn them.
2. **Use two spaces for indentation and never a tab character.** YAML forbids tabs for indentation. Configure the editor accordingly before starting.
3. **Do not edit these files in a word processor.** Curly quotation marks and non-breaking spaces are invalid YAML and produce errors that are hard to diagnose.

### Listing index

| Listing | Saved as | Object kinds |
| --- | --- | --- |
| 1 | `cluster/kind-cluster.yaml` | kind `Cluster` |
| 1B | `cluster/kind-cluster-fallback.yaml` | kind `Cluster` |
| 2 | `manifests/00-namespace.yaml` | `Namespace` |
| 3 | `manifests/01-quota-and-limits.yaml` | `ResourceQuota`, `LimitRange` |
| 4 | `manifests/02-pod-web.yaml` | `Pod` |
| 5 | `manifests/03-deployment-web.yaml` | `Deployment` |
| 6 | `manifests/04-service-clusterip.yaml` | `Service` |
| 7 | `manifests/05-service-nodeport.yaml` | `Service` |
| 8 | `manifests/06-pod-client.yaml` | `Pod` |

### Container images used

| Image | Role | Reason for the choice |
| --- | --- | --- |
| `kindest/node:v1.36.1` | Every cluster node | The default node image for kind v0.32.0, pinned by digest so that the whole class runs an identical cluster |
| `nginx:1.30-alpine` | Web server | The nginx stable branch. Alpine-based, therefore small and quick to pull. Its shell is `sh`, not `bash` |
| `nginx:1.31-alpine` | Rolling-update target | The nginx mainline branch, used only to demonstrate a version change in Stage 5 (9.4) |
| `busybox:1.37` | Content writer, client | A single small binary providing `sh`, `wget`, `nslookup`, and the standard file utilities |

### Label scheme used throughout

Consistent labels are assessed under *Code Organisation & Readability*. Every object in this practical carries the same four keys.

| Label key | Values used | Purpose |
| --- | --- | --- |
| `app` | `web`, `client` | The application a Pod belongs to. This is the key Services select on |
| `tier` | `frontend`, `tooling` | The architectural layer |
| `dso202/practical` | `01` | Which practical produced the object, so that later practicals do not collide |
| `dso202/managed-by` | `declarative` | Records that the object came from a committed manifest rather than a terminal command |

---

## Listing 1 — kind cluster configuration

**Save as:** `cluster/kind-cluster.yaml`
**Descriptor section:** Unit I 1.1
**Apply with:** `kind create cluster --config cluster/kind-cluster.yaml`

```yaml
# =============================================================================
# File : cluster/kind-cluster.yaml
# Practical : DSO202 Practical 1
# Apply with : kind create cluster --config cluster/kind-cluster.yaml
# Delete with : kind delete cluster --name dso202
# =============================================================================

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

# The cluster name. It determines two things:
# - the Docker container names: dso202-control-plane, dso202-worker,
# dso202-worker2
# - the kubectl context name: kind-dso202
name: dso202

networking:
 # Publish the API server on the loopback interface only. On a shared or
 # public network, binding to all interfaces would expose the cluster.
 apiServerAddress: "127.0.0.1"

 # Pod IP range. Every Pod receives an address from this range. It is stated
 # explicitly rather than left to the default so that the addresses seen in
 # 'kubectl get pods -o wide' can be recognised as Pod addresses.
 podSubnet: "10.244.0.0/16"

 # Service IP range. Every ClusterIP is allocated from this range, which is
 # why the in-cluster DNS service is reachable at 10.96.0.10.
 serviceSubnet: "10.96.0.0/16"

nodes:
 # ---------------------------------------------------------------------------
 # Node 1 of 3: the control-plane node.
 # Runs kube-apiserver, etcd, kube-scheduler, kube-controller-manager, and
 # also the kubelet and kube-proxy that every node runs.
 # By default kind taints this node so that application Pods are not
 # scheduled onto it. That taint is left in place.
 # ---------------------------------------------------------------------------
 - role: control-plane

 # The node image is pinned by digest so that every student in the class
 # runs exactly Kubernetes v1.36.1 and sees the output shown in the guide.
 # If the installed kind release is newer than v0.32.0, delete this line and
 # let kind choose its own default image instead - node images are not
 # guaranteed to be compatible across kind releases.
 image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5

 # kind builds a kubeadm configuration for each node. A patch merges extra
 # fields into it. On the first control-plane node the relevant kubeadm
 # object is InitConfiguration, because kind runs 'kubeadm init' here.
 #
 # 'nodeRegistration.name' is the name the kubelet registers with the API
 # server, which is the name that 'kubectl get nodes' displays. It is
 # independent of the Docker container name, which kind always derives from
 # the cluster name and cannot be configured.
 #
 # No 'apiVersion' is given inside the patch on purpose. kind translates an
 # unversioned patch to whichever kubeadm API version the target Kubernetes
 # release requires - v1beta4 for Kubernetes 1.36 and later. Adding an
 # explicit apiVersion would tie this file to one Kubernetes release.
 kubeadmConfigPatches:
 - |
 kind: InitConfiguration
 nodeRegistration:
 name: control-plane

 # Publish a host port into this node's container so that the NodePort
 # Service in Listing 5 can be reached from the host with
 # 'curl http://localhost:30080'.
 #
 # A NodePort is opened on EVERY node, so mapping it on any one node is
 # sufficient; the control-plane node is used here because it is the node
 # that is certain to exist in every cluster.
 #
 # containerPort must equal the 'nodePort' value in Listing 7.
 extraPortMappings:
 - containerPort: 30080
 hostPort: 30080
 listenAddress: "127.0.0.1"
 protocol: TCP

 # ---------------------------------------------------------------------------
 # Node 2 of 3: first worker.
 # Runs the kubelet, kube-proxy, the CNI plugin, and application Pods.
 # The kubeadm object here is JoinConfiguration, because kind runs
 # 'kubeadm join' on every node after the first control-plane node.
 # ---------------------------------------------------------------------------
 - role: worker
 image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
 kubeadmConfigPatches:
 - |
 kind: JoinConfiguration
 nodeRegistration:
 name: worker-node-1
 kubeletExtraArgs:
 node-labels: "dso202/node-role=worker,dso202/node-index=1"

 # ---------------------------------------------------------------------------
 # Node 3 of 3: second worker.
 # A second worker exists so that Pod placement by the scheduler is visible.
 # ---------------------------------------------------------------------------
 - role: worker
 image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
 kubeadmConfigPatches:
 - |
 kind: JoinConfiguration
 nodeRegistration:
 name: worker-node-2
 kubeletExtraArgs:
 node-labels: "dso202/node-role=worker,dso202/node-index=2"
```

### Notes on Listing 1

- The two node-name patches are the only non-standard part of this file. kind has no dedicated field for naming nodes; the name is set by patching the kubeadm configuration that kind generates.
- The `node-labels` values are custom labels of the form `<prefix>/<name>=<value>`. A kubelet is permitted to set labels under a custom prefix such as `dso202/`, but is deliberately prevented from setting most labels under `kubernetes.io/`. These labels are used in Extension 1.
- The `image:` lines may be removed if a newer kind release is installed. Every other line should be left unchanged, because the guide's expected output depends on it.

---

## Listing 1B — fallback cluster configuration

**Save as:** `cluster/kind-cluster-fallback.yaml`
**Use only if:** Listing 1 fails during cluster creation with an error mentioning `nodeRegistration`, or `kubectl get nodes` continues to show the Docker container names after a clean recreation.

This configuration is identical to Listing 1 except that it omits the two node-name patches. The Kubernetes Node objects then take kind's default names — `dso202-control-plane`, `dso202-worker`, `dso202-worker2` — and every command in the guide works unchanged apart from those names. If this fallback is used, state so in the report's Reflection section and substitute the names consistently throughout the evidence.

```yaml
# =============================================================================
# File : cluster/kind-cluster-fallback.yaml
# Practical : DSO202 Practical 1
# Descriptor : Unit I 1.1 - Kubernetes architecture and components
# Purpose : Same three-node cluster as Listing 1, without the node-name
# patches. Node objects keep kind's default names.
# Apply with : kind create cluster --config cluster/kind-cluster-fallback.yaml
# =============================================================================
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

name: dso202

networking:
 apiServerAddress: "127.0.0.1"
 podSubnet: "10.244.0.0/16"
 serviceSubnet: "10.96.0.0/16"

nodes:
 # Node object name will be: dso202-control-plane
 - role: control-plane
 extraPortMappings:
 - containerPort: 30080
 hostPort: 30080
 listenAddress: "127.0.0.1"
 protocol: TCP

 # Node object name will be: dso202-worker
 - role: worker
 kubeadmConfigPatches:
 - |
 kind: JoinConfiguration
 nodeRegistration:
 kubeletExtraArgs:
 node-labels: "dso202/node-role=worker,dso202/node-index=1"

 # Node object name will be: dso202-worker2
 - role: worker
 kubeadmConfigPatches:
 - |
 kind: JoinConfiguration
 nodeRegistration:
 kubeletExtraArgs:
 node-labels: "dso202/node-role=worker,dso202/node-index=2"
```

---

## Listing 2 — Namespace

**Save as:** `manifests/00-namespace.yaml`
**Descriptor section:** Unit I 1.5.1
**Apply with:** `kubectl apply -f manifests/00-namespace.yaml`

```yaml
# =============================================================================
# File : manifests/00-namespace.yaml
# Practical : DSO202 Practical 1
# Apply with : kubectl apply -f manifests/00-namespace.yaml
# Prefix 00- : applied first; every later object depends on it existing.
# =============================================================================
apiVersion: v1
kind: Namespace
metadata:
 # Namespaces are cluster-scoped, so this name must be unique across the
 # whole cluster, not just within a namespace.
 name: dso202-practical-01

 # A namespace is labelled for the same reason a Pod is: so that it can be
 # selected. 'kubectl get namespace -l dso202/practical=01' finds it without
 # relying on the name.
 labels:
 dso202/practical: "01"
 dso202/managed-by: declarative

 annotations:
 # Annotations carry information for humans and tools. They are never used
 # as selectors, which is the difference between an annotation and a label.
 dso202.cst.edu.bt/description: "Namespace for DSO202 Practical 1 workloads"
 dso202.cst.edu.bt/owner: "student_name"
 dso202.cst.edu.bt/date: "2026-08-11"
```

---

## Listing 3 — ResourceQuota and LimitRange

**Save as:** `manifests/01-quota-and-limits.yaml`
**Descriptor section:** Unit I 1.5.3
**Apply with:** `kubectl apply -f manifests/01-quota-and-limits.yaml`

This is the only file in the practical that contains two objects. They are kept together because neither is useful without the other: the quota makes resource declarations mandatory, and the limit range supplies them when a quick imperative command does not.

```yaml
# =============================================================================
# File : manifests/01-quota-and-limits.yaml
# Practical : DSO202 Practical 1
# Apply with : kubectl apply -f manifests/01-quota-and-limits.yaml
# Contains : two objects, separated by the YAML document separator '---'
# =============================================================================

apiVersion: v1
kind: ResourceQuota
metadata:
 name: dso202-quota
 namespace: dso202-practical-01
 labels:
 dso202/practical: "01"
 dso202/managed-by: declarative
spec:
 # 'hard' values are absolute. The API server rejects any request that would
 # take the namespace past one of them.
 #
 # Setting a compute cap has a second effect that is easy to miss: once
 # 'requests.cpu' is capped, EVERY container in the namespace must declare
 # requests.cpu, or its Pod is rejected. That is why the LimitRange below is
 # applied at the same time.
 hard:
 # Total guaranteed CPU across all Pods. 2 means two full CPU cores.
 # Chosen because the workloads in this practical request about 500m in
 # total, leaving room for the extension tasks without exhausting a
 # typical laptop.
 requests.cpu: "2"

 # Total guaranteed memory across all Pods.
 requests.memory: 2Gi

 # Total maximum CPU. Higher than the request total because limits are
 # allowed to be over-committed: containers rarely reach their limits at
 # the same moment.
 limits.cpu: "4"
 limits.memory: 4Gi

 # An object-count cap. It stops runaway creation from filling the cluster
 # even when the objects themselves consume no CPU or memory.
 # 20 comfortably accomodate Pods this practical creates.
 pods: "20"

 # Object-count caps may be written for any resource with 'count/<resource>'.
 # This practical creates 2 Services; 5 leaves room for the extensions.
 count/services: "5"

 # ConfigMaps and Secrets are introduced in a later practical. The caps are
 # stated here so that the pattern is visible.
 count/configmaps: "10"
 count/secrets: "10"

---

apiVersion: v1
kind: LimitRange
metadata:
 name: dso202-limits
 namespace: dso202-practical-01
 labels:
 dso202/practical: "01"
 dso202/managed-by: declarative
spec:
 limits:
 # 'type: Container' means these rules are evaluated per container, not per
 # Pod. A LimitRange may also target Pod or PersistentVolumeClaim.
 - type: Container

 # Applied as 'limits' when a container declares none. Without this, the
 # ResourceQuota above would reject any container that omitted limits.
 default:
 cpu: 200m
 memory: 128Mi

 # Applied as 'requests' when a container declares none.
 # Deliberately lower than 'default': a request is what is guaranteed and
 # is what the scheduler reserves, while a limit is a ceiling. Requesting
 # less than the ceiling allows more Pods per node.
 defaultRequest:
 cpu: 50m
 memory: 64Mi

 # The smallest values a container may declare. Below these, a container
 # is so constrained that it will fail in ways that look like bugs.
 min:
 cpu: 10m
 memory: 16Mi

 # The largest values a container may declare. This is what makes the
 # 600Mi request in Extension 2 fail. Note that a LimitRange rejects the
 # single container, whereas a ResourceQuota rejects based on the
 # namespace total - two different mechanisms with similar messages.
 max:
 cpu: "1"
 memory: 512Mi
```

### Unit notation used in Listings 3 to 8

| Notation | Meaning |
| --- | --- |
| `1` for CPU | One CPU core |
| `500m` for CPU | 500 millicores, that is half a core. `1000m` equals `1` |
| `Mi`, `Gi` | Binary multiples: `1Mi` is 1048576 bytes, `1Gi` is 1024 `Mi` |
| `M`, `G` | Decimal multiples: `1M` is 1000000 bytes. Not used in this practical, because binary units are conventional for memory |

---

## Listing 4 — Pod with labels and resource allocation

**Save as:** `manifests/02-pod-web.yaml`
**Descriptor section:** Unit I 1.2.1
**Apply with:** `kubectl apply -f manifests/02-pod-web.yaml`

```yaml
# =============================================================================
# File : manifests/02-pod-web.yaml
# Practical : DSO202 Practical 1
# Inspect with: kubectl get pod web-pod -o wide --show-labels
# Delete with : kubectl delete -f manifests/02-pod-web.yaml
#
# NOTE: A bare Pod has no controller behind it. If its node fails, nothing
# recreates it. Listing 8 replaces this with a Deployment, which is what
# real workloads use. This Pod exists to make the Pod specification
# itself visible without a controller in the way.
# =============================================================================
apiVersion: v1
kind: Pod
metadata:
 name: web-pod
 namespace: dso202-practical

 # Labels are the only mechanism by which other objects find this Pod.
 # A Service, a ReplicaSet, and a NetworkPolicy all locate Pods by label.
 labels:
 app: web # selected on by the Services in Listings 9 and 10
 tier: frontend # architectural layer
 dso202/practical: "01" # which practical created this object
 dso202/managed-by: declarative # created from a committed file, not a command

 annotations:
 dso202.rub.edu.bt/description: "Single-container Pod for DSO202 Practical 1 Stage 4"

spec:
 # 'Always' is the default and is stated explicitly so that the field is
 # visible. It means the kubelet restarts a container whenever it exits,
 # regardless of the exit code. 'OnFailure' and 'Never' suit batch work.
 restartPolicy: Always

 # How long the kubelet waits, after sending the termination signal, before
 # killing a container. 30 is the default; it is stated so that the shutdown
 # path is not invisible.
 terminationGracePeriodSeconds: 30

 containers:
 # The container name must be unique within the Pod. It is the value passed
 # to 'kubectl logs -c' and 'kubectl exec -c'.
 - name: web

 # The tag is pinned to a specific nginx branch. 'nginx:latest' would
 # make this Pod non-reproducible: the same manifest could produce a
 # different application tomorrow.
 image: nginx:1.30-alpine

 # IfNotPresent uses a locally cached image when one exists. Correct for
 # a pinned tag. 'Always' would be required for a mutable tag, and is
 # what Kubernetes defaults to when the tag is 'latest'.
 imagePullPolicy: IfNotPresent

 ports:
 # Declaring a port does not open or publish anything. It is metadata:
 # it documents the port for readers, and it gives the port a NAME that
 # a Service's targetPort can refer to instead of a number.
 - name: http
 containerPort: 80
 protocol: TCP

 resources:
 # requests = what is guaranteed. The scheduler subtracts the request
 # from a node's allocatable capacity when deciding whether the Pod
 # fits. An under-stated request risks eviction; an over-stated one
 # wastes capacity.
 #
 # 50m and 64Mi were chosen by observing that an idle nginx worker uses
 # a few millicores and under 16Mi, leaving a safety margin.
 requests:
 cpu: 50m
 memory: 64Mi

 # limits = the ceiling. Exceeding the memory limit terminates the
 # container with reason OOMKilled. Exceeding the CPU limit only
 # throttles it, because CPU is compressible and memory is not.
 #
 # 200m allows a burst of four times the request to absorb start-up and
 # traffic spikes without permitting one container to starve a node.
 limits:
 cpu: 200m
 memory: 128Mi
```

---

## Listing 5 — Deployment

**Save as:** `manifests/03-deployment-web.yaml`
**Descriptor section:** Unit I 1.2.2 (ReplicaSets), 1.2.3 (Deployments)
**Apply with:** `kubectl apply -f manifests/03-deployment-web.yaml`

```yaml
# =============================================================================
# File : manifests/03-deployment-web.yaml
# Apply with : kubectl apply -f manifests/03-deployment-web.yaml
# Watch with : kubectl rollout status deployment/web-deployment
# Update with : kubectl set image deployment/web-deployment web=nginx:1.31-alpine
# Undo with : kubectl rollout undo deployment/web-deployment
# Verify with : kubectl diff -f manifests/03-deployment-web.yaml
#
# OWNERSHIP CHAIN: Deployment -> ReplicaSet -> Pods
# The Deployment holds the update strategy and the revision history.
# The ReplicaSet it creates holds the replica count and does the counting.
# Only Pods actually run containers.
# =============================================================================
apiVersion: apps/v1 # NOT 'v1'. Deployments live in the 'apps' API group.
kind: Deployment
metadata:
 name: web-deployment
 namespace: dso202-practical-01

 # Labels on the DEPLOYMENT object. These are for finding the Deployment
 # itself and are separate from the labels in spec.template.metadata.labels,
 # which land on the Pods.
 labels:
 app: web
 tier: frontend
 dso202/practical: "01"
 dso202/managed-by: declarative

spec:
 # Desired number of Pods. The ReplicaSet's control loop counts matching Pods
 # continuously and creates or deletes Pods until this number is reached.
 replicas: 3

 # How the Deployment finds the Pods it owns.
 #
 # IMMUTABLE: this field cannot be changed after creation. Changing it
 # requires deleting and recreating the Deployment.
 #
 # It must also MATCH spec.template.metadata.labels below, or the API server
 # rejects the object with 'selector does not match template labels'.
 selector:
 matchLabels:
 app: web
 tier: frontend

 strategy:
 # RollingUpdate replaces Pods incrementally. The alternative, 'Recreate',
 # deletes every old Pod before creating any new one, which causes downtime.
 type: RollingUpdate
 rollingUpdate:
 # How many replicas may be MISSING during an update.
 # 0 means the ready count never drops below 3. The default is 25%, which
 # would permit serving with fewer replicas mid-update.
 #
 # This single field is what makes the failed rollout in Stage 7 Step 15
 # harmless: because no old Pod may be removed until a new one is ready,
 # a new Pod that never becomes ready stalls the rollout instead of
 # causing an outage.
 maxUnavailable: 0

 # How many replicas may exist ABOVE the desired count during an update.
 # 1 gives the sequence: add one new Pod, wait for it to become ready,
 # remove one old Pod, repeat.
 maxSurge: 1

 # How long a new Pod must remain ready before it counts as available. Without
 # it, a Pod that passes one readiness check and then crashes would still
 # advance the rollout. 5 also makes the rollout slow enough to watch.
 minReadySeconds: 5

 # How many old ReplicaSets to keep, scaled to zero, for rollback. Each one
 # retains a full Pod template, so the value trades disk against how far back
 # a rollback can reach. The default is 10.
 revisionHistoryLimit: 5

 # How long a rollout may make no progress before it is marked as failed.
 progressDeadlineSeconds: 300

 # ---------------------------------------------------------------------------
 # The Pod template. Everything below this line is a Pod specification, and
 # is identical in structure to Listing 4. Changing ANY field here produces a
 # new template hash, therefore a new ReplicaSet, therefore a new rollout.
 # ---------------------------------------------------------------------------
 template:
 metadata:
 # Labels applied to every Pod the ReplicaSet creates. These must include
 # everything in spec.selector.matchLabels above.
 labels:
 app: web
 tier: frontend
 dso202/practical: "01"
 dso202/managed-by: declarative
 spec:
 containers:
 - name: web
 image: nginx:1.30-alpine
 imagePullPolicy: IfNotPresent

 ports:
 - name: http
 containerPort: 80
 protocol: TCP

 resources:
 # 3 replicas x 50m = 150m of the namespace's 2-core request quota.
 # During a rolling update, maxSurge allows a fourth Pod, so the
 # peak is 200m. Both figures were checked against Listing 3.
 requests:
 cpu: 50m
 memory: 64Mi
 limits:
 cpu: 200m
 memory: 128Mi

```

---

## Listing 6 — ClusterIP Service

**Save as:** `manifests/04-service-clusterip.yaml`
**Descriptor section:** Unit I 1.2.4
**Apply with:** `kubectl apply -f manifests/04-service-clusterip.yaml`

```yaml
# =============================================================================
# File : manifests/04-service-clusterip.yaml
# Practical : DSO202 Practical 1
# Apply with : kubectl apply -f manifests/04-service-clusterip.yaml
# Verify with : kubectl get endpointslice \
# -l kubernetes.io/service-name=web-clusterip
# kubectl exec client-pod -- wget -qO- http://web-clusterip
#
# DNS NAME CREATED: web-clusterip.dso202-practical.svc.cluster.local
# Within this namespace the short name 'web-clusterip' is enough.
# From another namespace, 'web-clusterip.dso202-practical' is required.
# =============================================================================
apiVersion: v1
kind: Service
metadata:
 name: web-clusterip
 namespace: dso202-practical-01
 labels:
 app: web
 tier: frontend
 dso202/practical: "01"
 dso202/managed-by: declarative
spec:
 # ClusterIP is the default type and is reachable only from inside the
 # cluster. It is stated explicitly so that the choice is visible.
 type: ClusterIP

 # Which Pods receive the traffic. Note the difference from a Deployment's
 # selector: this is a plain map, with no 'matchLabels' wrapper, because the
 # Service API predates the newer selector syntax.
 #
 # A Pod is sent traffic only if it BOTH matches this selector AND is
 # currently passing its readiness probe. An empty EndpointSlice is the
 # signature of a selector that matches nothing.
 selector:
 app: web
 tier: frontend

 ports:
 - name: http

 # The port on the SERVICE. Clients connect here.
 port: 80

 # The port on the POD that traffic is forwarded to. Given as the NAME of
 # the container port declared in Listing 5 rather than the number 80.
 # Using the name means the container port could change without editing
 # this file, and it documents the intent.
 targetPort: http

 protocol: TCP

 # How long a client is pinned to the same Pod. 'None' means every connection
 # is balanced independently, which is what the request-distribution test in
 # Stage 6 Step 6 relies on. 'ClientIP' would pin by source address.
 sessionAffinity: None
```

---

## Listing 7 — NodePort Service

**Save as:** `manifests/05-service-nodeport.yaml`
**Descriptor section:** Unit I 1.2.4
**Apply with:** `kubectl apply -f manifests/05-service-nodeport.yaml`

```yaml
# =============================================================================
# File : manifests/05-service-nodeport.yaml
# Apply with : kubectl apply -f manifests/05-service-nodeport.yaml
# Verify with : curl -s http://localhost:30080
# docker exec dso202-worker curl -s http://localhost:30080
#
# DEPENDS ON LISTING 1: nodePort below must equal the 'containerPort' in the
# control-plane node's extraPortMappings. Change one and the other must change.
#
# REQUEST PATH:
# host :30080
# -> control-plane container :30080 (published by Listing 1)
# -> kube-proxy on that node (rewrites to a Pod address)
# -> a ready Pod on a worker node, port 80
# =============================================================================
apiVersion: v1
kind: Service
metadata:
 name: web-nodeport
 namespace: dso202-practical-01
 labels:
 app: web
 tier: frontend
 dso202/practical: "01"
 dso202/managed-by: declarative
spec:
 # NodePort does everything ClusterIP does - it still allocates a cluster IP
 # and a DNS name - and additionally opens 'nodePort' on EVERY node.
 type: NodePort

 selector:
 app: web
 tier: frontend

 ports:
 - name: http
 port: 80 # port on the Service, for in-cluster clients
 targetPort: http # named container port on the Pod

 # The port opened on every node. Valid range is 30000 to 32767; the API
 # server rejects anything outside it.
 #
 # Omitting this field is usually better practice, because the cluster
 # then allocates a free port automatically. It is fixed here only
 # because Listing 1 had to publish a specific host port in advance, and
 # the two must agree.
 nodePort: 30080

 protocol: TCP
```

---

## Listing 8 — Client Pod

**Save as:** `manifests/06-pod-client.yaml`
**Descriptor section:** Unit I 1.3.3
**Apply with:** `kubectl apply -f manifests/06-pod-client.yaml`

```yaml
# =============================================================================
# File : manifests/06-pod-client.yaml
# Apply with : kubectl apply -f manifests/06-pod-client.yaml
# Use with : kubectl exec client-pod -- nslookup web-clusterip
# kubectl exec client-pod -- wget -qO- http://web-clusterip
# kubectl exec -it client-pod -- sh
#
# WHY 'sleep infinity': a container whose main process exits is treated as
# having crashed, and with restartPolicy Always it enters CrashLoopBackOff.
# A diagnostic Pod must therefore keep a process alive deliberately.
# =============================================================================
apiVersion: v1
kind: Pod
metadata:
 name: client-pod
 namespace: dso202-practical-01
 labels:
 app: client
 tier: tooling
 dso202/practical: "01"
 dso202/managed-by: declarative
spec:
 # No Service selects on 'app: client', so this Pod receives no traffic. It
 # only sends requests.
 restartPolicy: Always

 # Shorter than the default 30 seconds, because there is nothing to shut down
 # gracefully and a fast delete is convenient during cleanup.
 terminationGracePeriodSeconds: 5

 containers:
 - name: client
 image: busybox:1.37
 imagePullPolicy: IfNotPresent

 # busybox provides sh, wget, nslookup, ping, and the standard file
 # utilities in a single small binary, which is everything needed here.
 command: ["/bin/sh", "-c", "sleep infinity"]

 resources:
 requests:
 cpu: 25m
 memory: 32Mi
 limits:
 cpu: 100m
 memory: 64Mi
```

---

## Appendix A — Imperative Equivalents

Every object in this practical could have been created with a single command. The table below gives the closest imperative equivalent for each listing, together with what that command would have omitted. The gap in the third column is the reason declarative manifests are the graded artefact.

| Listing | Closest imperative command | What the command cannot express |
| --- | --- | --- |
| 2 | `kubectl create namespace dso202-practical` | Labels and annotations |
| 3 | `kubectl create quota dso202-quota --hard=pods=20,requests.cpu=2,requests.memory=2Gi,limits.cpu=4,limits.memory=4Gi` | Object-count caps, and the LimitRange, which has no `kubectl create` verb at all |
| 4 | `kubectl run web-pod --image=nginx:1.30-alpine --restart=Never --port=80 --labels='app=web,tier=frontend'` | Resource requests and limits, named ports, `imagePullPolicy`, annotations |
| 5 | No single command exists | Probes cannot be created imperatively at all |
| 6 | `kubectl run liveness-failure-pod --image=busybox:1.37 --restart=Always -- /bin/sh -c '...'` | The liveness probe |
| 7 | No single command exists | Multiple containers, volumes, and volume mounts cannot be created imperatively |
| 8 | `kubectl create deployment web-deployment --image=nginx:1.30-alpine --replicas=3` | Rollout strategy, `minReadySeconds`, `revisionHistoryLimit`, resources, all three probes, named ports, labels beyond `app` |
| 9 | `kubectl expose deployment web-deployment --name=web-clusterip --port=80 --target-port=80` | A `targetPort` given by name, `sessionAffinity`, a selector chosen independently of the Deployment's labels |
| 10 | `kubectl expose deployment web-deployment --name=web-nodeport --type=NodePort --port=80` | A fixed `nodePort`, which the command allocates at random instead |
| 11 | `kubectl run client-pod --image=busybox:1.37 -- sleep infinity` | Resources, labels beyond `run=client-pod`, `terminationGracePeriodSeconds` |

The pattern is consistent: imperative commands can create objects but cannot express the fields that make an object production-ready. Use them to explore, and to generate a first draft with `--dry-run=client -o yaml`; then commit the result.

---

## Appendix B — Applying and Removing Everything

### Apply in order

Because the filenames are numbered, a single command applies them in the correct sequence. `kubectl apply -f` accepts a directory and processes its files in lexical order.

```bash
kubectl apply -f manifests/
```

### Verify the cluster matches the repository

`kubectl diff` compares each manifest against the live object and prints nothing when they agree. Run it before claiming that the repository reflects the running state.

```bash
kubectl diff -f manifests/ && echo "cluster matches repository"
```

### Remove everything

```bash
kubectl delete -f manifests/
```

Deleting the namespace would be faster but would also delete the ResourceQuota and LimitRange, and would not prove that every object came from a committed file:

```bash
# Faster, but weaker evidence. Prefer the command above.
kubectl delete namespace dso202-practical
```

### Remove the cluster

```bash
kind delete cluster --name dso202
```

---

## Appendix C — YAML Rules That Cause Most First-Session Errors

| Rule | Why it matters | Symptom when broken |
| --- | --- | --- |
| Indentation uses spaces only | YAML forbids tab characters for indentation | `error converting YAML to JSON: yaml: line N: found character that cannot start any token` |
| Indentation is significant and must be consistent | Two spaces per level throughout; a field indented one level too far belongs to a different parent | `unknown field "..."`, or a field that is silently ignored |
| A hyphen introduces a list item | `containers:` holds a list, so each container begins with `- name:` | `cannot unmarshal object into Go value of type []v1.Container` |
| Quote values that could be read as another type | `"01"`, `"1"`, `"true"` are strings; `01`, `1`, `true` are a number, a number, and a boolean. Label values must be strings | `cannot unmarshal number into Go value of type string` |
| Memory units are case-sensitive | `Mi` and `Gi` are binary multiples; `mi` is invalid | `quantities must match the regular expression ...` |
| CPU `m` means milli | `100m` is a tenth of a core; `100` is one hundred cores | The Pod stays `Pending` with `Insufficient cpu` |
| `---` separates documents in one file | Two objects in one file must be separated by it | The second object is never created |
| `\|` keeps line breaks, `>` folds them into spaces | Shell scripts embedded in `args` need `\|` | The script runs as one long line and behaves unexpectedly |
| Comments start with `#` and run to end of line | There is no block-comment syntax in YAML | A stray `/* */` becomes invalid content |
| Object names are lowercase | Names must be lowercase alphanumeric characters, `-`, or `.`, at most 253 characters | `a lowercase RFC 1123 subdomain must consist of ...` |

When a field name is uncertain, ask the cluster rather than guessing:

```bash
kubectl explain pod.spec.containers.resources
kubectl explain deployment.spec.strategy --recursive
kubectl explain service.spec.ports
```

`kubectl explain` reads the schema from the running API server, so its answer is always correct for the version in use — which is more reliable than any reference table, including this one.

---

*Source: [Manifest file - HackMD](https://hackmd.io/@sarojsanyasi/dso202-practical1-manifest-files)*

*End of companion file. The procedure, expected output, troubleshooting, and assessment criteria are in `DSO202_Practical1_Guide.md`.*