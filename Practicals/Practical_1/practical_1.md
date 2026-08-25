# DSO202 — Practical 1

## Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

## Practical Information

| Field | Detail |
| --- | --- |
| Module | DSO202 — Scaling, Orchestration, Monitoring & Observability |
| Programme | BE in Software Engineering |
| Practical number | 1 of 10 |
| Descriptor entry | *List of Practicals*, item 1: "Set up a local Kubernetes cluster using Minikube or Kind; core kubectl operations." |
| Selected tool | **kind** (Kubernetes IN Docker). |
| Subject matter covered | Unit I — 1.1, 1.2.1, 1.2.2, 1.2.3, 1.2.4, 1.3.1, 1.3.2, 1.3.3, 1.4.1, 1.5.1, 1.5.3 |
| Learning Outcomes addressed | LO1, LO2, LO3, and the multi-tenancy half of LO5 |
| Contact time | 2 hours (allocated weekly practical class) |
| Independent study | Approximately 2 hours to complete the report and the extension tasks |
| Weighting | Part of *Practical Work & Report* (20% of the module): report out of 5 marks, practical work out of 15 marks |
| Companion file | `DSO202_Practical1_Manifests.md` — contains every YAML listing referenced below |
| Tool versions verified against | kind v0.32.0, Kubernetes v1.36.1, kubectl v1.36 |

---

## 1. Purpose of This Practical

This practical builds the environment that every remaining practical in DSO202 depends on. By the end of the session a three-node Kubernetes cluster will be running on the student's own machine, and five categories of Kubernetes object will have been created, inspected, broken on purpose, and repaired.

The practical is deliberately built around a single narrow application — a static web server — so that attention stays on the Kubernetes objects rather than on application code.

---

## 2. Learning Outcomes Addressed

| Descriptor LO | Statement | Where it is met in this practical |
| --- | --- | --- |
| LO1 | Understand the core concepts and architecture of Kubernetes | Stage 2 |
| LO2 | Deploy and manage applications on a Kubernetes cluster using various resource types | Stages 4, 5, 6, 7, 8 |
| LO3 | Operate Kubernetes CLI (kubectl) for cluster management and troubleshooting | Every stage; concentrated in Stages 2, 4, and the Troubleshooting section |
| LO5 (part) | Apply namespace-based multi-tenancy in Kubernetes environments | Stage 3 |

Persistent storage (LO4) is covered in Practical 2. The service-registry half of LO5 is covered in Practical 6.

---

## 3. Repository Structure

All work is to be submitted to the personal version-control repository (Github) of students. Create the following structure before starting Stage 1. Marks for *Code Organisation & Readability* are awarded against it.

```text
dso202-practical-01/
├── README.md # what this practical does and how to run it
├── cluster/
│ └── kind-cluster.yaml # Listing 1
├── manifests/
│ ├── 00-namespace.yaml # Listing 2
│ ├── 01-quota-and-limits.yaml # Listing 3
│ ├── 02-pod-web.yaml # Listing 4
│ ├── 03-deployment-web.yaml # Listing 5
│ ├── 04-service-clusterip.yaml # Listing 6
│ ├── 05-service-nodeport.yaml # Listing 7
│ └── 06-pod-client.yaml # Listing 8
├── evidence/ # command output captured as .txt, screenshots as .png
└── report/
 └── practical-01-report.md # the assessed report
```

---

## 4. Stage 0 — Prerequisites and Verification

**Why this stage matters.** Roughly half of all first-session failures are environment problems, not Kubernetes problems. Verifying each tool separately means any failure can be attributed to a single cause.

### 4.1 Required software

| Software | Minimum version | Purpose |
| --- | --- | --- |
| Docker Engine or Docker Desktop | 24.0 | Provides the container runtime in which kind runs each node |
| kind | 0.32.0 | Creates and deletes the cluster |
| kubectl | 1.35 or 1.36 | Talks to the cluster |

kubectl is supported within one minor version of the cluster it addresses. Because the cluster created here runs Kubernetes v1.36.1, kubectl v1.35, v1.36, or v1.37 will all work.

### 4.2 Installation

Installation commands change frequently and are not reproduced here. Use the official pages:

- Docker: `https://docs.docker.com/engine/install/`
- kind: `https://kind.sigs.k8s.io/docs/user/quick-start/#installation`
- kubectl: `https://kubernetes.io/docs/tasks/tools/`

Windows users must enable the WSL 2 backend in Docker Desktop and run all commands from inside a WSL 2 Linux shell. Running kind from PowerShell is possible but is not supported in this practical.

### 4.3 Verification steps

**Step 1.** Confirm the Docker daemon is running and reachable.

```bash
docker info --format '{{.ServerVersion}} {{.OperatingSystem}}'
```

Expected output, with values that will differ by machine:

```text
28.1.1 Docker Desktop
```

An error containing `Cannot connect to the Docker daemon` means Docker is installed but not started. Start Docker Desktop, or on Linux run `sudo systemctl start docker`, then repeat the step.

**Step 2.** Confirm kind is installed.

```bash
kind version
```

```text
kind v0.32.0 go1.24.4 linux/amd64
```

**Step 3.** Confirm kubectl is installed. The `--client` flag is used because no cluster exists yet.

```bash
kubectl version --client
```

```text
Client Version: v1.36.0
Kustomize Version: v5.7.1
```

**Step 4.** Confirm at least 4 GB of memory and 2 CPUs are available to Docker, and at least 15 GB of free disk. A three-node cluster will start on less, but Pods will be evicted unpredictably during Stage 5.

**Checkpoint.** All three commands print a version. Do not continue until they do.

---

## 5. Stage 1 — Creating the Three-Node Cluster

**Why this stage matters.** A single-node cluster hides the most important idea in Kubernetes: that a workload is placed onto a node by the scheduler rather than started by hand on a particular machine. Two worker nodes make placement visible.

### 5.1 What the cluster will look like

```text
 host machine (one laptop)
 ┌───────────────────────────────────────────────────────────────┐
 │ Docker │
 │ │
 │ ┌───────────────────────┐ │
 │ │ container: │ Kubernetes Node object name: │
 │ │ dso202-control-plane │ control-plane │
 │ │ │ runs kube-apiserver, etcd, │
 │ │ │ kube-scheduler, │
 │ │ │ kube-controller-manager, │
 │ │ │ kubelet, kube-proxy │
 │ └───────────────────────┘ │
 │ │
 │ ┌───────────────────────┐ ┌───────────────────────┐ │
 │ │ container: │ │ container: │ │
 │ │ dso202-worker │ │ dso202-worker2 │ │
 │ │ Node object name: │ │ Node object name: │ │
 │ │ worker-node-1 │ │ worker-node-2 │ │
 │ │ runs kubelet, │ │ runs kubelet, │ │
 │ │ kube-proxy, │ │ kube-proxy, │ │
 │ │ application Pods │ │ application Pods │ │
 │ └───────────────────────┘ └───────────────────────┘ │
 │ │
 │ host port 30080 ──► control-plane container port 30080 │
 └───────────────────────────────────────────────────────────────┘
```

Two names appear for each node, and the difference is examinable:

- The **Docker container name** is chosen by kind and always follows the pattern `<cluster-name>-control-plane`, `<cluster-name>-worker`, `<cluster-name>-worker2`. It is visible to `docker ps` and to `kind get nodes`. kind does not offer a configuration field to change it.
- The **Kubernetes Node object name** is the name the kubelet registers with the API server. It is visible to `kubectl get nodes`, and it *can* be set, by patching the kubeadm configuration that kind generates. Listing 1 does exactly that, which is why `kubectl get nodes` will show `control-plane`, `worker-node-1`, and `worker-node-2`.

### 5.2 Steps

**Step 1.** Copy **Listing 1** from `DSO202_Practical1_Manifests.md` into `cluster/kind-cluster.yaml`.

**Step 2.** Create the cluster. The first run downloads a node image of roughly 1 GB and may take several minutes; later runs take about one minute.

```bash
kind create cluster --config cluster/kind-cluster.yaml
```

Expected output. kind decorates each progress line with a small icon, which is omitted here:

```text
Creating cluster "dso202" ...
 - Ensuring node image (kindest/node:v1.36.1)
 - Preparing nodes
 - Writing configuration
 - Starting control-plane
 - Installing CNI
 - Installing StorageClass
 - Joining worker nodes
Set kubectl context to "kind-dso202"
```

kind then prints a suggested `kubectl cluster-info --context kind-dso202` command and a closing message.

**Step 3.** Confirm kind considers the cluster to exist, and list the Docker containers behind it.

```bash
kind get clusters
kind get nodes --name dso202
```

```text
dso202
dso202-control-plane
dso202-worker
dso202-worker2
```

**Step 4.** Confirm the same three nodes as Docker containers.

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

```text
NAMES IMAGE STATUS
dso202-worker2 kindest/node:v1.36.1 Up 2 minutes
dso202-worker kindest/node:v1.36.1 Up 2 minutes
dso202-control-plane kindest/node:v1.36.1 Up 2 minutes
```

**Step 5.** Confirm that kubectl is pointing at the new cluster. kind adds a context named `kind-<cluster-name>` and selects it automatically.

```bash
kubectl config current-context
```

```text
kind-dso202
```

**Checkpoint.** `kind get clusters` prints `dso202`, and `kubectl config current-context` prints `kind-dso202`.

## 6. Stage 2 — Inspecting the Cluster and Its Components

**Why this stage matters.** Unit I 1.1 lists the control-plane and node components. This stage shows each of those components as an object that can be listed, described, and read, which converts a diagram from the lecture into something verifiable.

### 6.1 Steps

**Step 1.** Ask the cluster where its control plane is.

```bash
kubectl cluster-info
```

```text
Kubernetes control plane is running at https://127.0.0.1:38945
CoreDNS is running at https://127.0.0.1:38945/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

The port number is chosen at random by kind on each creation and will differ. The address is on `127.0.0.1` because the API server's port is published from the control-plane container to the host.

**Step 2.** List the nodes. The renamed Node objects appear here.

```bash
kubectl get nodes
```

```text
NAME STATUS ROLES AGE VERSION
control-plane Ready control-plane 3m21s v1.36.1
worker-node-1 Ready <none> 3m2s v1.36.1
worker-node-2 Ready <none> 3m2s v1.36.1
```

`ROLES` is derived from node labels, not from anything intrinsic. Worker nodes show `<none>` because kind assigns them no role label.

**Step 3.** Add columns to the same query. `-o wide` is the fastest way to get more detail without switching to full YAML output.

```bash
kubectl get nodes -o wide
```

```text
NAME STATUS ROLES AGE VERSION INTERNAL-IP OS-IMAGE KERNEL-VERSION CONTAINER-RUNTIME
control-plane Ready control-plane 4m v1.36.1 172.18.0.4 Debian GNU/Linux 12 (bookworm) 6.10.14 containerd://2.1.4
worker-node-1 Ready <none> 3m v1.36.1 172.18.0.2 Debian GNU/Linux 12 (bookworm) 6.10.14 containerd://2.1.4
worker-node-2 Ready <none> 3m v1.36.1 172.18.0.3 Debian GNU/Linux 12 (bookworm) 6.10.14 containerd://2.1.4
```

The `INTERNAL-IP` values are Docker network addresses. `CONTAINER-RUNTIME` confirms containerd, as stated in the vocabulary section.

**Step 4.** Read one node in detail and locate the labels applied by Listing 1.

```bash
kubectl describe node worker-node-1
```

The output is long. Three regions of it are worth reading now:

| Region | What it shows |
| --- | --- |
| `Labels:` | Includes `dso202/node-role=worker` and `dso202/node-index=1`, both set by Listing 1 |
| `Capacity:` and `Allocatable:` | Total CPU and memory the node reports, and how much of it Kubernetes will hand out |
| `Non-terminated Pods:` | Every Pod currently placed on this node, with its CPU and memory requests |

To retrieve only the labels, use a JSONPath expression rather than reading the whole description:

```bash
kubectl get node worker-node-1 -o jsonpath='{.metadata.labels}' | tr ',' '\n'
```

**Step 5.** List the namespaces that exist before any work is done.

```bash
kubectl get namespaces
```

```text
NAME STATUS AGE
default Active 5m
kube-node-lease Active 5m
kube-public Active 5m
kube-system Active 5m
local-path-storage Active 5m
```

| Namespace | Purpose |
| --- | --- |
| `default` | Where objects go when no namespace is specified. Production clusters avoid using it. |
| `kube-system` | Control-plane components, kube-proxy, CoreDNS, and the CNI plugin |
| `kube-public` | Readable by unauthenticated clients; holds cluster information |
| `kube-node-lease` | Holds one lease object per node, used for fast node-failure detection |
| `local-path-storage` | Added by kind; provides the default StorageClass used in Practical 2 |

**Step 6.** List the control-plane components. They run as Pods in `kube-system`.

```bash
kubectl get pods -n kube-system -o wide
```

```text
NAME READY STATUS RESTARTS AGE IP NODE
coredns-7db6d8ff4d-8xq2n 1/1 Running 0 5m 10.244.0.3 control-plane
coredns-7db6d8ff4d-vc4kt 1/1 Running 0 5m 10.244.0.2 control-plane
etcd-control-plane 1/1 Running 0 5m 172.18.0.4 control-plane
kindnet-4jd7p 1/1 Running 0 5m 172.18.0.2 worker-node-1
kindnet-9mhzq 1/1 Running 0 5m 172.18.0.4 control-plane
kindnet-hd6vx 1/1 Running 0 5m 172.18.0.3 worker-node-2
kube-apiserver-control-plane 1/1 Running 0 5m 172.18.0.4 control-plane
kube-controller-manager-control-plane 1/1 Running 0 5m 172.18.0.4 control-plane
kube-proxy-2kw8f 1/1 Running 0 5m 172.18.0.2 worker-node-1
kube-proxy-p7rzt 1/1 Running 0 5m 172.18.0.4 control-plane
kube-proxy-vqn5s 1/1 Running 0 5m 172.18.0.3 worker-node-2
kube-scheduler-control-plane 1/1 Running 0 5m 172.18.0.4 control-plane
```

Three observations to record in the report:

1. `etcd`, `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` appear **once**, all on `control-plane`. These are the control-plane components from Unit I 1.1.1.
2. `kube-proxy` and `kindnet` appear **three times**, once per node. Components that must run everywhere are deployed as DaemonSets, which are covered later in the module.
3. `coredns` appears twice, both on the control-plane node, because this cluster's control-plane taint is tolerated by CoreDNS but not by ordinary Pods.

**Step 7.** Read the log of one control-plane component. This is the same mechanism used for application logs in Stage 4.

```bash
kubectl logs -n kube-system kube-scheduler-control-plane --tail=10
```

**Step 8.** List every kind of object the cluster knows about, and note which are namespaced.

```bash
kubectl api-resources --namespaced=true -o name | head -20
kubectl api-resources --namespaced=false -o name | head -10
```

Nodes, Namespaces, PersistentVolumes, and StorageClasses are cluster-scoped; Pods, Services, Deployments, ResourceQuotas, and LimitRanges are namespaced. Knowing which is which prevents the common error of adding `-n` to a command that ignores it.

**Checkpoint.** All three nodes report `Ready`, and all Pods in `kube-system` report `Running` with `1/1` containers ready.

### 6.2 Common pitfalls in this stage

- **Forgetting `-n kube-system`.** `kubectl get pods` with no namespace flag reads the `default` namespace, which is empty, and prints `No resources found in default namespace.` This is not an error.
- **Reading the API server port from the lecture notes.** It is randomised per cluster; always read it from `kubectl cluster-info`.
- **Expecting `describe` output to be machine-readable.** `kubectl describe` is formatted for humans. For scripting, use `-o json`, `-o yaml`, or `-o jsonpath`.

---

## 7. Stage 3 — Namespaces, Resource Quotas, and Limit Ranges

**Why this stage matters.** Descriptor sections 1.5.1 and 1.5.3. A namespace is how a single cluster is shared between teams, environments, or students without name collisions, and a ResourceQuota is what stops one namespace consuming the whole cluster. Both are the foundation of the multi-tenancy half of LO5.

### 7.1 Definitions

**Namespace.** A named partition of the cluster. Object names must be unique within a namespace only, so `pod/web` may exist in three namespaces simultaneously as three unrelated Pods. Namespaces partition names, quotas, and access-control rules. They do **not** partition the network by default; a Pod in one namespace can reach a Pod in another unless a NetworkPolicy forbids it.

**ResourceQuota.** A namespace-scoped object that caps aggregate consumption in that namespace. Two kinds of cap are used in Listing 3: compute caps such as `requests.cpu`, and object-count caps such as `count/services`. When a compute cap for a resource is set, the API server *rejects* any Pod that does not declare a request and limit for that resource.

**LimitRange.** A namespace-scoped object that supplies default `requests` and `limits` to containers that omit them, and rejects containers whose values fall outside a permitted band. It is the companion to a ResourceQuota: the quota makes declarations mandatory, and the LimitRange supplies them when a quick imperative command does not.

### 7.2 Steps

**Step 1.** Create the namespace imperatively first, purely to see the imperative form, then delete it. The declarative version in Listing 2 is the graded artefact.

```bash
kubectl create namespace dso202-scratch
kubectl get namespace dso202-scratch
kubectl delete namespace dso202-scratch
```

```text
namespace/dso202-scratch created
NAME STATUS AGE
dso202-scratch Active 2s
namespace "dso202-scratch" deleted
```

**Step 2.** Generate a manifest from an imperative command without executing it. Compare the result with Listing 2 and note that Listing 2 adds labels, which the generated version does not.

```bash
kubectl create namespace dso202-practical-01 --dry-run=client -o yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
 creationTimestamp: null
 name: dso202-practical
spec: {}
status: {}
```

**Step 3.** Copy **Listing 2** into `manifests/00-namespace.yaml` and apply it.

```bash
kubectl apply -f manifests/00-namespace.yaml
```

```text
namespace/dso202-practical created
```

**Step 4.** Set the namespace as the default for the current context. Every subsequent command in this practical then omits `-n`.

```bash
kubectl config set-context --current --namespace=dso202-practical
kubectl config view --minify --output 'jsonpath={..namespace}'
```

```text
Context "kind-dso202" modified.
dso202-practical
```

**Step 5.** Copy **Listing 3** into `manifests/01-quota-and-limits.yaml` and apply it. The file contains two objects separated by `---`.

```bash
kubectl apply -f manifests/01-quota-and-limits.yaml
```

```text
resourcequota/dso202-quota created
limitrange/dso202-limits created
```

**Step 6.** Read the quota. The `Used` column is what makes a quota useful during troubleshooting.

```bash
kubectl describe resourcequota dso202-quota
```

```text
Name: dso202-quota
Namespace: dso202-practical
Resource Used Hard
-------- ---- ----
count/configmaps 1 10
count/secrets 0 10
count/services 0 5
limits.cpu 0 4
limits.memory 0 4Gi
pods 0 20
requests.cpu 0 2
requests.memory 0 2Gi
```

`count/configmaps` already reports `1`. Every namespace is given a ConfigMap named `kube-root-ca.crt`, holding the cluster's certificate authority certificate, so that Pods can verify the API server. A quota's `Used` column counts everything in the namespace, including objects the cluster created itself.

**Step 7.** Read the limit range.

```bash
kubectl describe limitrange dso202-limits
```

```text
Name: dso202-limits
Namespace: dso202-practical
Type Resource Min Max Default Request Default Limit Max Limit/Request Ratio
---- -------- --- --- --------------- ------------- -----------------------
Container cpu 10m 1 50m 200m -
Container memory 16Mi 512Mi 64Mi 128Mi -
```

**Step 8.** Prove that the LimitRange is doing its work. Start a Pod imperatively with no resource declaration at all, then read back what the cluster actually stored.

```bash
kubectl run limitrange-check --image=nginx:1.30-alpine --restart=Never
kubectl get pod limitrange-check -o jsonpath='{.spec.containers[0].resources}'
echo
```

```text
pod/limitrange-check created
{"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
```

The command supplied no resources, yet the stored object has all four values. They came from the LimitRange. Without the LimitRange, this Pod would have been rejected by the ResourceQuota.

**Step 9.** Delete the check Pod, since it is not part of the deliverable.

```bash
kubectl delete pod limitrange-check
```

**Checkpoint.** `kubectl get resourcequota,limitrange` lists both objects, and the current context's default namespace is `dso202-practical`.

### 7.3 Common pitfalls in this stage

- **`Error from server (Forbidden): ... exceeded quota`.** The namespace has hit a cap. Run `kubectl describe resourcequota dso202-quota`, compare `Used` against `Hard`, and delete unneeded objects rather than raising the quota.
- **`must specify limits.cpu` on apply.** The ResourceQuota caps `limits.cpu`, so every container must declare it. Either add the declaration to the manifest, which is the correct answer for graded work, or rely on the LimitRange default.
- **Deleting a namespace to clean up one Pod.** Deleting a namespace deletes every object inside it, including the quota and limit range. Delete individual objects instead.
- **Assuming the namespace flag persists.** `-n` applies to one command only. The default namespace set in Step 4 persists in the kubeconfig context until changed.

---

## 8. Stage 4 — Pods

### 8.1 Definition

**Pod.** The smallest deployable object in Kubernetes. A Pod is a group of one or more containers that share:

- **A network namespace.** All containers in a Pod share one IP address and one port range. They reach each other on `localhost`, and no two containers in the same Pod may bind the same port.
- **Storage volumes.** Any volume declared in the Pod specification may be mounted by any of its containers.
- **A lifecycle.** The Pod is scheduled once, onto one node, and stays there. A Pod is never moved; it is deleted and a replacement is created elsewhere.

A Pod created directly, as in this stage, is not managed by any controller. If its node fails, nothing recreates it. That limitation is precisely what the Deployment in Stage 7 removes.

**Pod phase.** The `STATUS` column reports the Pod's phase, together with more specific container reasons:

| Value | Meaning |
| --- | --- |
| `Pending` | Accepted by the API server, but not yet running. Either no node has been selected, or images are still being pulled. |
| `Running` | Bound to a node, and all containers have been created. At least one container is running. |
| `Succeeded` | All containers exited with status 0 and will not be restarted. |
| `Failed` | All containers have terminated and at least one exited non-zero. |
| `ContainerCreating` | A container reason, not a phase: the runtime is setting the container up. |
| `ImagePullBackOff` | A container reason: the image could not be pulled, and retries are being spaced out. |
| `CrashLoopBackOff` | A container reason: the container keeps exiting and restarts are being spaced out. |

### 8.2 The imperative route

**Step 1.** Create a Pod with one command, then inspect it.

```bash
kubectl run web-imperative \
 --image=nginx:1.30-alpine \
 --restart=Never \
 --port=80 \
 --labels='app=web,tier=frontend,managed-by=imperative'
```

```text
pod/web-imperative created
```

Three flags deserve comment:

| Flag | Effect |
| --- | --- |
| `--restart=Never` | Produces a bare Pod. Without it, older kubectl versions produced other object types; stating it removes all ambiguity. |
| `--port=80` | Records the container port in the Pod specification. It does not open anything on the host. |
| `--labels=` | Attaches labels at creation time. Labels added later with `kubectl label` are equivalent. |

**Step 2.** Watch the Pod reach `Running`. Press `Ctrl+C` to stop watching.

```bash
kubectl get pod web-imperative --watch
```

```text
NAME READY STATUS RESTARTS AGE
web-imperative 0/1 ContainerCreating 0 2s
web-imperative 1/1 Running 0 9s
```

**Step 3.** Capture the imperative Pod as a manifest. This is the bridge between the two styles.

```bash
kubectl get pod web-imperative -o yaml > evidence/web-imperative-as-stored.yaml
```

Open the file. Note how much the cluster added: a `status` block, a `nodeName`, a service account, a default `terminationGracePeriodSeconds`, tolerations, and the resources injected by the LimitRange. A manifest committed to version control must contain only what the author intends to declare, which is why the retrieved YAML is evidence and Listing 4 is the deliverable.

**Step 4.** Delete the imperative Pod.

```bash
kubectl delete pod web-imperative
```

### 8.3 The declarative route

**Step 5.** Copy **Listing 4** into `manifests/02-pod-web.yaml` and read the comments in it before applying. Then apply it.

```bash
kubectl apply -f manifests/02-pod-web.yaml
```

```text
pod/web-pod created
```

**Step 6.** Apply the same file a second time without changing it.

```bash
kubectl apply -f manifests/02-pod-web.yaml
```

```text
pod/web-pod unchanged
```

The word `unchanged` is the whole point of declarative management. The command states a desired outcome; if the outcome already holds, nothing happens. Running the same command twice is safe.

**Step 7.** Confirm placement and the assigned Pod IP.

```bash
kubectl get pod web-pod -o wide
```

```text
NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
web-pod 1/1 Running 0 31s 10.244.1.4 worker-node-1 <none> <none>
```

The `NODE` column may show either worker. The scheduler chose it; the manifest did not. The `IP` is a cluster-internal Pod address from the range configured in Listing 1, and it is not reachable from the host.

**Step 8.** Confirm the resource declarations took effect.

```bash
kubectl get pod web-pod -o jsonpath='{.spec.containers[0].resources}'
echo
```

```text
{"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
```

These values came from the manifest, not from the LimitRange. Verify by re-reading the quota:

```bash
kubectl describe resourcequota dso202-quota | grep -E 'requests.cpu|requests.memory|pods'
```

```text
pods 1 20
requests.cpu 50m 2
requests.memory 64Mi 2Gi
```

**Step 9.** Describe the Pod and read the `Events:` section at the bottom.

```bash
kubectl describe pod web-pod
```

The events form a timeline of everything the cluster did:

```text
Events:
 Type Reason Age From Message
 ---- ------ ---- ---- -------
 Normal Scheduled 60s default-scheduler Successfully assigned dso202-practical/web-pod to worker-node-1
 Normal Pulling 59s kubelet Pulling image "nginx:1.30-alpine"
 Normal Pulled 54s kubelet Successfully pulled image "nginx:1.30-alpine" in 4.8s
 Normal Created 54s kubelet Created container: web
 Normal Started 53s kubelet Started container web
```

The `From` column identifies which component performed each action, and it maps directly onto the components listed in Unit I 1.1: `default-scheduler` chose the node, and `kubelet` did everything after that. **When a Pod misbehaves, the events section is the first place to look, not the last.**

### 8.4 Labels and selectors

**Step 10.** Display labels alongside the Pod list.

```bash
kubectl get pods --show-labels
```

```text
NAME READY STATUS RESTARTS AGE LABELS
web-pod 1/1 Running 0 3m app=web,managed-by=declarative,tier=frontend
```

**Step 11.** Select Pods by label rather than by name. This is how every controller and every Service finds its Pods.

```bash
kubectl get pods -l app=web
kubectl get pods -l tier=frontend,managed-by=declarative
kubectl get pods -l 'tier in (frontend,backend)'
kubectl get pods -l app!=web
```

The first three commands list `web-pod`. The fourth prints `No resources found in dso202-practical namespace.`

**Step 12.** Add and then remove a label at runtime. The trailing hyphen removes a label.

```bash
kubectl label pod web-pod environment=practical
kubectl get pods -l environment=practical
kubectl label pod web-pod environment-
```

```text
pod/web-pod labeled
NAME READY STATUS RESTARTS AGE
web-pod 1/1 Running 0 4m
pod/web-pod unlabeled
```

**Step 13.** Add an annotation and observe that it cannot be selected on.

```bash
kubectl annotate pod web-pod dso202.rub.edu.bt/practical="01"
kubectl get pod web-pod -o jsonpath='{.metadata.annotations}'
echo
```

Labels are for machines to select on; annotations are for humans and tools to read.

### 8.5 Debugging and troubleshooting commands

These four commands are descriptor section 1.3.3, and they are the most-used commands in the remainder of the module.

**Step 14.** Read the container's log stream. `-f` follows it; `--tail` limits the history.

```bash
kubectl logs web-pod --tail=5
```

```text
/docker-entrypoint.sh: Configuration complete; ready for start up
```

**Step 15.** Open a shell inside the running container. Because `nginx:1.30-alpine` is Alpine-based, the shell is `sh`, not `bash`.

```bash
kubectl exec -it web-pod -- sh
```

Inside the container, run the following, then type `exit`:

```sh
hostname
cat /etc/os-release | head -2
ls /usr/share/nginx/html
wget -qO- http://localhost:80 | head -5
exit
```

The hostname is the Pod name. The server answers on `localhost` from inside its own Pod because the container shares the Pod's network namespace.

**Step 16.** Run a single command inside the container without an interactive session.

```bash
kubectl exec web-pod -- nginx -v
```

```text
nginx version: nginx/1.30.4
```

**Step 17.** Forward a local port to the Pod. This opens a tunnel from the host, through the API server, to the Pod. It is intended for debugging, never for exposing an application.

```bash
kubectl port-forward pod/web-pod 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Leave that command running, and in a **second terminal**:

```bash
curl -s http://localhost:8080 | head -5
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Return to the first terminal and press `Ctrl+C`.

**Step 18.** Use `kubectl explain` whenever a field is unfamiliar. It reads the schema from the live cluster, so its answer is always correct for the running version.

```bash
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.containers.livenessProbe --recursive | head -30
```

**Checkpoint.** `web-pod` is `Running` and `1/1`, its labels are visible with `--show-labels`, and `kubectl logs`, `kubectl exec`, and `kubectl port-forward` have all produced output.

### 8.6 Common pitfalls in this stage

- **Using `bash` in `kubectl exec` against an Alpine image.** Alpine ships `sh`, not `bash`. The error is `exec: "bash": executable file not found in $PATH`.
- **Expecting the Pod IP to work from the host.** Pod IPs live on the cluster network. Use `kubectl port-forward`, or a Service, as in Stage 8.
- **Editing a live Pod's image with `kubectl edit`.** Most Pod fields are immutable after creation. The correct action is to change the manifest and re-apply, which deletes and recreates the Pod.
- **Reading logs of the wrong container.** In a multi-container Pod, `kubectl logs` without `-c` fails and lists the available container names. Stage 6 covers this.
- **Naming a Pod with capital letters or underscores.** Object names must be lowercase alphanumeric characters, `-`, or `.`.

---

## 9. Stage 5 — Deployments

**Why this stage matters.** Descriptor sections 1.2.2 and 1.2.3. Every Pod created so far is unmanaged: delete it, and it is gone. A Deployment adds the two properties that make a workload production-viable — a guaranteed replica count, and a controlled way to change the running version without downtime. Rolling updates and rollbacks are also the behaviour ArgoCD automates in Unit IV.

### 9.1 Definitions

**ReplicaSet.** A controller object holding a Pod template, a replica count, and a label selector. Its control loop counts the Pods matching its selector, creates more if there are too few, and deletes some if there are too many. ReplicaSets are rarely written by hand.

**Deployment.** A controller object that manages ReplicaSets. Changing a Deployment's Pod template causes it to create a **new** ReplicaSet and shift replicas from the old one to the new one gradually. The old ReplicaSet is kept, scaled to zero, so that a rollback is a matter of shifting replicas back.

The chain of ownership is worth memorising:

```text
Deployment ──manages──► ReplicaSet ──manages──► Pods
(strategy, (replica count, (containers,
 revision history) label selector) volumes, probes)
```

**RollingUpdate strategy.** The default. Replicas are replaced incrementally, governed by two fields:

| Field | Meaning |
| --- | --- |
| `maxUnavailable` | How many replicas below the desired count the Deployment may drop during the update. `0` means never serve with fewer replicas than requested. |
| `maxSurge` | How many replicas above the desired count may exist temporarily. `1` means add one new Pod, wait for it to become ready, then remove one old Pod. |

**Recreate strategy.** All old Pods are deleted before any new Pod is created. This causes downtime and is used only when two versions cannot run simultaneously.

### 9.2 Steps

**Step 1.** Generate a Deployment manifest imperatively, purely for comparison. Then read Listing 8 and note everything the generated version omits: labels beyond the bare minimum, resource declarations, probes, a rollout strategy, and comments.

```bash
kubectl create deployment web-deployment \
 --image=nginx:1.30-alpine --replicas=3 --dry-run=client -o yaml | head -25
```

**Step 2.** Copy **Listing 5** into `manifests/03-deployment-web.yaml` and apply it.

```bash
kubectl apply -f manifests/03-deployment-web.yaml
kubectl rollout status deployment/web-deployment
```

```text
deployment.apps/web-deployment created
Waiting for deployment "web-deployment" rollout to finish: 0 of 3 updated replicas are available...
deployment "web-deployment" successfully rolled out
```

**Step 3.** Observe the three-level ownership chain in the cluster.

```bash
kubectl get deployment,replicaset,pod -l app=web
```

```text
NAME READY UP-TO-DATE AVAILABLE AGE
deployment.apps/web-deployment 3/3 3 3 40s

NAME DESIRED CURRENT READY AGE
replicaset.apps/web-deployment-6c9f7d4b58 3 3 3 40s

NAME READY STATUS RESTARTS AGE
pod/web-deployment-6c9f7d4b58-4ktqp 1/1 Running 0 40s
pod/web-deployment-6c9f7d4b58-hb2xn 1/1 Running 0 40s
pod/web-deployment-6c9f7d4b58-tzv9c 1/1 Running 0 40s
```

Two naming rules are visible. The ReplicaSet name is the Deployment name plus a hash of the Pod template. Each Pod name is the ReplicaSet name plus a random suffix. Because the hash is derived from the template, **any change to the template produces a new ReplicaSet name.**

**Step 4.** Confirm the ownership relationship rather than inferring it from names.

```bash
kubectl get replicaset -l app=web -o jsonpath='{.items[0].metadata.ownerReferences[0].kind}/{.items[0].metadata.ownerReferences[0].name}{"\n"}'
```

```text
Deployment/web-deployment
```

**Step 5.** Confirm the scheduler spread the replicas across the worker nodes.

```bash
kubectl get pods -l app=web -o wide --no-headers | awk '{print $1, $7}'
```

```text
web-deployment-6c9f7d4b58-4ktqp worker-node-1
web-deployment-6c9f7d4b58-hb2xn worker-node-2
web-deployment-6c9f7d4b58-tzv9c worker-node-1
```

The exact distribution varies. The point is that the manifest never named a node; the scheduler placed each Pod.

**Step 6.** Demonstrate self-healing. Delete one Pod and immediately list them again.

```bash
victim=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
echo "deleting $victim"
kubectl delete pod "$victim"
kubectl get pods -l app=web
```

```text
deleting web-deployment-6c9f7d4b58-4ktqp
pod "web-deployment-6c9f7d4b58-4ktqp" deleted
NAME READY STATUS RESTARTS AGE
web-deployment-6c9f7d4b58-hb2xn 1/1 Running 0 2m
web-deployment-6c9f7d4b58-tzv9c 1/1 Running 0 2m
web-deployment-6c9f7d4b58-x7prc 1/1 Running 0 6s
```

A replacement Pod appears within seconds, with a new random suffix but the same ReplicaSet prefix. Record the before-and-after listing in the report; this is the single clearest illustration of reconciliation in the practical.

```bash
kubectl get events --field-selector reason=SuccessfulCreate --sort-by=.lastTimestamp | tail -3
```

```text
LAST SEEN TYPE REASON OBJECT MESSAGE
12s Normal SuccessfulCreate replicaset/web-deployment-6c9f7d4b58 Created pod: web-deployment-6c9f7d4b58-x7prc
```

The actor is the ReplicaSet, not the Deployment. The Deployment delegates.

### 9.3 Scaling

**Step 7.** Scale imperatively, then read the result.

```bash
kubectl scale deployment web-deployment --replicas=5
kubectl get deployment web-deployment
```

```text
deployment.apps/web-deployment scaled
NAME READY UP-TO-DATE AVAILABLE AGE
web-deployment 5/5 5 5 3m
```

**Step 8.** Return to three replicas the declarative way, by re-applying the unmodified manifest.

```bash
kubectl apply -f manifests/06-deployment-web.yaml
kubectl get deployment web-deployment
```

```text
deployment.apps/web-deployment configured
NAME READY UP-TO-DATE AVAILABLE AGE
web-deployment 3/3 3 3 4m
```

This is the reason imperative changes are unsuitable for graded work: the next `apply` silently reverses them. Whatever the manifest says wins. In Unit IV, ArgoCD enforces exactly this property continuously.

### 9.4 Rolling update and rollback

**Step 9.** Watch the rollout in a second terminal.

```bash
kubectl get pods -l app=web --watch
```

**Step 10.** In the first terminal, change the image version. `nginx:1.31-alpine` is the current mainline release; `nginx:1.30-alpine` is the stable release used so far.

```bash
kubectl set image deployment/web-deployment web=nginx:1.31-alpine
kubectl annotate deployment web-deployment \
 kubernetes.io/change-cause="upgrade nginx from 1.30-alpine to 1.31-alpine"
kubectl rollout status deployment/web-deployment
```

```text
deployment.apps/web-deployment image updated
deployment.apps/web-deployment annotated
Waiting for deployment "web-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "web-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "web-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "web-deployment" successfully rolled out
```

In the watching terminal, the pattern to observe is: a new Pod is created, it becomes `1/1 Ready`, and only then does an old Pod begin terminating. With `maxUnavailable: 0`, the number of ready Pods never falls below three.

**Step 11.** Confirm two ReplicaSets now exist, one of them scaled to zero.

```bash
kubectl get replicaset -l app=web
```

```text
NAME DESIRED CURRENT READY AGE
web-deployment-6c9f7d4b58 0 0 0 6m
web-deployment-8d5b47c9f2 3 3 3 70s
```

The old ReplicaSet is retained precisely so that a rollback needs no image pull and no new object.

**Step 12.** Read the revision history.

```bash
kubectl rollout history deployment/web-deployment
```

```text
deployment.apps/web-deployment
REVISION CHANGE-CAUSE
1 <none>
2 upgrade nginx from 1.30-alpine to 1.31-alpine
```

The `CHANGE-CAUSE` column is populated from the `kubernetes.io/change-cause` annotation set in Step 10. Revision 1 shows `<none>` because no annotation existed then. Setting this annotation on every change is an examinable good practice.

**Step 13.** Inspect one revision in detail.

```bash
kubectl rollout history deployment/web-deployment --revision=1 | grep -i image
```

```text
 Image: nginx:1.30-alpine
```

**Step 14.** Roll back to the previous revision and confirm.

```bash
kubectl rollout undo deployment/web-deployment
kubectl rollout status deployment/web-deployment
kubectl get deployment web-deployment -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

```text
deployment.apps/web-deployment rolled back
deployment "web-deployment" successfully rolled out
nginx:1.30-alpine
```

**Step 15.** Demonstrate a failed rollout and recover from it. This is deliberate: a rollout that cannot succeed must be recognised quickly.

```bash
kubectl set image deployment/web-deployment web=nginx:9.99-does-not-exist
kubectl rollout status deployment/web-deployment --timeout=60s
```

```text
Waiting for deployment "web-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition
```

```bash
kubectl get pods -l app=web
```

```text
NAME READY STATUS RESTARTS AGE
web-deployment-6c9f7d4b58-4ktqp 1/1 Running 0 9m
web-deployment-6c9f7d4b58-hb2xn 1/1 Running 0 9m
web-deployment-6c9f7d4b58-tzv9c 1/1 Running 0 9m
web-deployment-b47c8f9d55-qm4rp 0/1 ImagePullBackOff 0 62s
```

Note carefully what did **not** happen: the three healthy Pods were never removed. Because `maxUnavailable: 0` and the new Pod never became ready, the rolling update stalled without causing an outage. A correctly configured rollout strategy converts a bad release into a stalled release rather than a failure.

**Step 16.** Diagnose and undo.

```bash
kubectl describe pod -l app=web | grep -A3 'Failed'
kubectl rollout undo deployment/web-deployment
kubectl rollout status deployment/web-deployment
kubectl get pods -l app=web
```

```text
deployment.apps/web-deployment rolled back
deployment "web-deployment" successfully rolled out
NAME READY STATUS RESTARTS AGE
web-deployment-6c9f7d4b58-4ktqp 1/1 Running 0 11m
web-deployment-6c9f7d4b58-hb2xn 1/1 Running 0 11m
web-deployment-6c9f7d4b58-tzv9c 1/1 Running 0 11m
```

**Step 17.** Restore the declared state, so that the repository and the cluster agree again.

```bash
kubectl apply -f manifests/03-deployment-web.yaml
kubectl diff -f manifests/03-deployment-web.yaml && echo "cluster matches manifest"
```

`kubectl diff` compares a manifest against the live object and prints nothing when they agree. It is the correct final check before claiming that a manifest reflects reality.

**Checkpoint.** `web-deployment` reports `3/3`, the image is `nginx:1.30-alpine`, `kubectl rollout history` shows at least three revisions, and `kubectl diff` reports no difference.

### 9.5 Common pitfalls in this stage

- **Editing `spec.selector` on an existing Deployment.** The selector is immutable after creation. The apply fails with a field-is-immutable error, and the only remedy is to delete and recreate the Deployment.
- **A selector that does not match the Pod template's labels.** The Deployment is rejected at creation with `selector does not match template labels`.
- **Deleting Pods to "restart" an application.** The ReplicaSet recreates them immediately, so nothing is achieved. Use `kubectl rollout restart deployment/web-deployment`, which performs a proper rolling restart.
- **Setting `maxUnavailable` greater than 0 for a service that must stay available.** The default is 25%, which permits a temporary capacity reduction.
- **Relying on `kubectl set image` as the record of a change.** It is a debugging command. The manifest is the record, and the next `apply` will overwrite anything the command did.

---

## 10. Stage 6 — Services

**Why this stage matters.** Descriptor section 1.2.4. Every Pod IP address in this practical has been temporary: it changes on every restart, every rollout, and every rescheduling. A Service supplies the one thing clients need and Pods cannot provide — a stable name and address that follows the Pods. Services are also the object that Prometheus discovers targets through in Unit V, and the object Istio's VirtualService routes to in Unit II.

### 10.1 Definitions

**Service.** An object that defines a stable virtual IP address and DNS name, together with a label selector. Traffic sent to the Service is load-balanced across the Pods that both match the selector and are currently ready.

**How it works.** The EndpointSlice controller in `kube-controller-manager` watches Services and Pods. For each Service with a selector, it maintains one or more EndpointSlice objects listing the addresses of the matching **ready** Pods. `kube-proxy` on every node watches those EndpointSlices and programs the node's packet-forwarding rules so that packets addressed to the Service IP are rewritten to one of the listed Pod addresses. CoreDNS also watches Services and answers DNS queries for their names.

**What a Service does not do.** It does not proxy at the application layer, does not terminate TLS, does not route on HTTP paths or hostnames, and does not perform retries. Those functions require an Ingress, covered in Unit II 2.2, or a service mesh, covered in Unit II 2.5.

**Service types.**

| Type | Reachable from | Definition |
| --- | --- | --- |
| `ClusterIP` | Inside the cluster only | The default. Allocates a virtual IP from the Service network. |
| `NodePort` | Outside the cluster, on any node's IP | Everything ClusterIP does, plus it reserves the same port between 30000 and 32767 on **every** node. |
| `LoadBalancer` | Outside the cluster, through a provider | Everything NodePort does, plus it asks the cloud provider for an external load balancer. In kind there is no such provider, so the Service stays `<pending>` forever. |
| Headless (`clusterIP: None`) | Inside the cluster only | Allocates no virtual IP. DNS returns the Pod addresses directly. Used with StatefulSets in Unit II 2.1.3. |

**Port fields.** Three port numbers appear in a Service specification and confusing them is the most common Service error:

| Field | Whose port it is |
| --- | --- |
| `port` | The port on the Service itself, which clients connect to |
| `targetPort` | The port on the Pod that traffic is forwarded to. May be a number or the *name* of a container port. |
| `nodePort` | The port opened on every node, for `NodePort` and `LoadBalancer` Services only |

### 10.2 ClusterIP

**Step 1.** Copy **Listing 6** into `manifests/04-service-clusterip.yaml` and apply it.

```bash
kubectl apply -f manifests/04-service-clusterip.yaml
kubectl get service web-clusterip
```

```text
service/web-clusterip created
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
web-clusterip ClusterIP 10.96.171.42 <none> 80/TCP 5s
```

**Step 2.** Read the EndpointSlice the controller generated. This is the list of addresses the Service will actually send traffic to.

```bash
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
```

```text
NAME ADDRESSTYPE PORTS ENDPOINTS AGE
web-clusterip-x8fq2 IPv4 80 10.244.1.7,10.244.2.5,10.244.1.8 18s
```

Three addresses, matching the three ready Pods of `web-deployment`. The older `kubectl get endpoints` command still works but prints a deprecation warning, because the Endpoints API was deprecated in Kubernetes 1.33 in favour of EndpointSlice.

**Step 3.** Copy **Listing 8** into `manifests/06-pod-client.yaml` and apply it. This Pod exists only to issue requests from inside the cluster.

```bash
kubectl apply -f manifests/09-pod-client.yaml
kubectl wait --for=condition=Ready pod/client-pod --timeout=60s
```

```text
pod/client-pod created
pod/client-pod condition met
```

**Step 4.** Resolve the Service name from inside the cluster.

```bash
kubectl exec client-pod -- nslookup web-clusterip
```

```text
Server: 10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name: web-clusterip
Address 1: 10.96.171.42 web-clusterip.dso202-practical.svc.cluster.local
```

The answer is the Service's ClusterIP, not a Pod IP. The fully qualified name follows a fixed pattern that is worth memorising, because it is used in every later unit:

```text
<service-name>.<namespace>.svc.cluster.local
```

Within the same namespace, the short name `web-clusterip` is sufficient. From another namespace, `web-clusterip.dso202-practical` is required.

**Step 5.** Send requests through the Service.

```bash
kubectl exec client-pod -- wget -qO- http://web-clusterip | head -4
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

**Step 6.** Demonstrate load balancing. Each Pod's hostname is its Pod name, so writing a distinct page into each Pod makes the distribution visible.

```bash
for pod in $(kubectl get pods -l app=web -o name); do
 kubectl exec "${pod#pod/}" -- sh -c 'echo "served by $HOSTNAME" > /usr/share/nginx/html/index.html'
done

for i in $(seq 1 9); do
 kubectl exec client-pod -- wget -qO- http://web-clusterip
done | sort | uniq -c
```

```text
 3 served by web-deployment-6c9f7d4b58-4ktqp
 3 served by web-deployment-6c9f7d4b58-hb2xn
 3 served by web-deployment-6c9f7d4b58-tzv9c
```

The exact split varies; what matters is that all three Pods answered. Note also that this change was made with `kubectl exec` and is therefore invisible to the manifest — deleting a Pod destroys it. That is why `exec` is a debugging tool and not a deployment mechanism.

**Step 7.** Demonstrate that readiness gates traffic. Break the readiness probe on one Pod and watch it leave the EndpointSlice.

The readiness probe in Listing 8 requests `/index.html`, so deleting that file makes the probe fail. Its liveness probe is a TCP check, which keeps passing, so the container is not restarted.

```bash
target=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$target" -- rm -f /usr/share/nginx/html/index.html
sleep 15
kubectl get pod "$target"
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
```

```text
NAME READY STATUS RESTARTS AGE
web-deployment-6c9f7d4b58-4ktqp 0/1 Running 0 15m

NAME ADDRESSTYPE PORTS ENDPOINTS AGE
web-clusterip-x8fq2 IPv4 80 10.244.2.5,10.244.1.8 12m
```

The Pod is still `Running` with `RESTARTS` at `0`, but it has been removed from the EndpointSlice and will receive no further traffic. Repeat the nine-request loop from Step 6 and confirm that only two Pod names now appear. Then restore it:

```bash
kubectl exec "$target" -- sh -c 'echo "served by $HOSTNAME" > /usr/share/nginx/html/index.html'
sleep 10
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
```

The third address returns to the EndpointSlice within one probe period. This automatic removal and restoration is the mechanism that made the zero-downtime rollout in Stage 7 possible.

**Step 8.** Demonstrate the most common Service fault: a selector that matches nothing.

```bash
kubectl create service clusterip broken-service --tcp=80:80
kubectl get endpointslice -l kubernetes.io/service-name=broken-service
```

```text
service/broken-service created
NAME ADDRESSTYPE PORTS ENDPOINTS AGE
broken-service-2c9tj IPv4 <unset> <unset> 6s
```

The imperative command generated the selector `app=broken-service`, which no Pod carries, so the EndpointSlice is empty and every request to the Service will be refused. **An empty EndpointSlice is the diagnostic signature of a selector mismatch.** Remove it:

```bash
kubectl delete service broken-service
```

### 10.3 NodePort

**Step 9.** Copy **Listing 7** into `manifests/05-service-nodeport.yaml` and apply it. It fixes `nodePort: 30080`, which is the port Listing 1 published from the control-plane container to the host.

```bash
kubectl apply -f manifests/05-service-nodeport.yaml
kubectl get service
```

```text
service/web-nodeport created
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
web-clusterip ClusterIP 10.96.171.42 <none> 80/TCP 14m
web-nodeport NodePort 10.96.204.118 <none> 80:30080/TCP 4s
```

The `PORT(S)` notation `80:30080/TCP` reads as "port 80 on the Service, exposed as port 30080 on every node".

**Step 10.** Reach the application from the host machine, outside the cluster, with no port-forward running.

```bash
curl -s http://localhost:30080
```

```text
served by web-deployment-6c9f7d4b58-hb2xn
```

Repeat the command several times and observe different Pod names. The request path is: host port 30080, into the control-plane container's port 30080, to `kube-proxy` on that node, across the Pod network to a ready Pod on a worker node.

**Step 11.** Confirm that the node port is open on every node, not only the one published to the host.

```bash
docker exec dso202-worker curl -s http://localhost:30080
```

```text
served by web-deployment-6c9f7d4b58-4ktqp
```

Only the control-plane container's port was published to the host, because that is what Listing 1 requested. All three nodes are listening.

**Step 12.** Confirm that a `LoadBalancer` Service cannot complete in kind, so that the behaviour is recognised rather than mistaken for a fault.

```bash
kubectl create service loadbalancer lb-demo --tcp=80:80
kubectl get service lb-demo
kubectl delete service lb-demo
```

```text
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
lb-demo LoadBalancer 10.96.88.201 <pending> 80:31640/TCP 5s
```

`EXTERNAL-IP` remains `<pending>` because no cloud provider is present to fulfil the request.

**Checkpoint.** `curl http://localhost:30080` answers from the host, the ClusterIP Service resolves by DNS from `client-pod`, and the EndpointSlice for `web-clusterip` lists three addresses.

### 10.4 Common pitfalls in this stage

- **Confusing `port` with `targetPort`.** If `targetPort` does not match a port the container actually listens on, the Service resolves and connects but every request times out.
- **Choosing a `nodePort` outside 30000–32767.** The API server rejects it. Omitting `nodePort` entirely is usually better, because the cluster then allocates a free one; this practical fixes it only because Listing 1 published a specific host port.
- **Expecting a Service to route on URL paths.** It cannot. That is an Ingress, in Unit II 2.2.
- **Forgetting that unready Pods are excluded.** A Service with a correct selector but no *ready* Pods behaves exactly like one with a wrong selector. Check `kubectl get pods` for the `READY` column before suspecting the selector.
- **Using a Pod IP anywhere in a manifest or configuration file.** Pod IPs are ephemeral. Always address a Service name.

---

## 11. Stage 7 — Cleanup

**Why this stage matters.** A kind cluster holds several gigabytes of disk and continues consuming memory until it is deleted. Reproducibility is also part of the assessment: a cluster that can be destroyed and rebuilt from `cluster/kind-cluster.yaml` and `manifests/` proves that the repository, and not the laptop, holds the work.

**Step 1.** Capture final evidence before deleting anything.

```bash
mkdir -p evidence
kubectl get all -o wide > evidence/final-state-all.txt
kubectl get resourcequota,limitrange,endpointslice -o wide >> evidence/final-state-all.txt
kubectl get nodes -o wide > evidence/final-state-nodes.txt
kubectl get events --sort-by=.lastTimestamp > evidence/final-state-events.txt
```

Note that `kubectl get all` is misleadingly named: it lists common workload and Service objects only, and omits ResourceQuotas, LimitRanges, ConfigMaps, Secrets, and every custom resource. The second command above compensates for part of that.

**Step 2.** Delete the workload objects declaratively, in reverse order of creation. Deleting from the same files that created the objects is the check that no object was created outside version control.

```bash
kubectl delete -f manifests/06-pod-client.yaml
kubectl delete -f manifests/05-service-nodeport.yaml
kubectl delete -f manifests/04-service-clusterip.yaml
kubectl delete -f manifests/03-deployment-web.yaml
kubectl delete -f manifests/02-pod-web.yaml
```

Confirm the namespace is empty apart from the quota and limit range:

```bash
kubectl get all
```

```text
No resources found in dso202-practical namespace.
```

**Step 3.** Rebuild everything from the repository in one command, to prove reproducibility. `kubectl apply -f` accepts a directory and applies the files in lexical order, which is exactly why the filenames are numbered.

```bash
kubectl apply -f manifests/
kubectl get all
```

```text
namespace/dso202-practical unchanged
resourcequota/dso202-quota unchanged
limitrange/dso202-limits unchanged
pod/web-pod created
deployment.apps/web-deployment created
service/web-clusterip created
service/web-nodeport created
pod/client-pod created
```

Record this output in the report. A single command that recreates the entire practical is the strongest possible evidence for the *Configuration Requirements* criterion.

**Step 4.** Reset the kubectl default namespace, so that later work is not silently placed in this namespace.

```bash
kubectl config set-context --current --namespace=default
```

**Step 5.** Delete the cluster.

```bash
kind delete cluster --name dso202
kind get clusters
docker ps
```

```text
Deleting cluster "dso202" ...
Deleted nodes: ["dso202-control-plane" "dso202-worker" "dso202-worker2"]
No kind clusters found.
```

`docker ps` shows no `kindest/node` containers. Deleting the cluster also removes its context from the kubeconfig file.

**Step 6.** Optionally reclaim the node image, which is roughly 1 GB. Keeping it makes the next cluster creation far faster, so remove it only if disk space is short.

```bash
docker image ls | grep kindest
# docker image rm kindest/node:v1.36.1
```

**Checkpoint.** `kind get clusters` reports no clusters, and the repository contains every manifest needed to rebuild the practical from nothing.

---

# 12. Deliverables

Submit one commit to the allocated version-control repository, at or before the deadline announced in class, containing:

1. `cluster/kind-cluster.yaml`
2. All manifests in `manifests/`, each carrying the comments required below
3. `README.md`, containing: the purpose of the repository, the software versions used, the exact sequence of commands needed to rebuild the practical from an empty machine, and the cleanup commands
4. Evidence
5. `report/practical-01-report.md`

### 12.1 Required report structure

The report is assessed out of 5 marks against the descriptor's *Practical Report* criteria. Use these headings.

| Heading | Content | Criterion |
| --- | --- | --- |
| 1. Objective | The purpose of the practical in the student's own words, and the descriptor sections it covers | Documentation |
| 2. Environment | Operating system, Docker version, kind version, kubectl version, cluster Kubernetes version | Documentation |
| 3. Procedure and Observations | One short subsection per stage. For each: what was done, the command output that proves it, and one sentence on what the output shows | Documentation |
| 4. Analysis | Answers to the review questions in section 21 | Documentation, Clarity & Coherence |
| 5. Reflection | What was difficult; which error was met and how it was diagnosed; what would be done differently; one thing that remains unclear | Reflection |
| 6. References | Documentation pages actually consulted, with the date of access | Documentation |

The Reflection section is worth 2 of the 5 report marks — the same as all the documentation combined. A reflection that names a specific error, the command used to diagnose it, and the reasoning that followed will score well. A reflection stating only that the practical was interesting and educational will not.

---

*Source: [DSO202 — Practical 1 - HackMD](https://hackmd.io/@sarojsanyasi/dso202-practical1-guide)*