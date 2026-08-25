# DSO202 — Practical 1 — Extra: Vocabulary, Imperative vs Declarative, Troubleshooting, Command Reference

## 1. Vocabulary

Every term below is used later in this guide. Reading this section first will make the remaining pages considerably faster to follow.

### 1.1 Container and image terminology

| Term | Definition |
| --- | --- |
| Container image | A read-only package containing an application together with its libraries, dependencies, and default start-up command. An image is a build-time artefact and does not run on its own. |
| Container | A running instance of a container image, isolated from other processes on the same machine by kernel features such as namespaces and cgroups. |
| Container runtime | The software on a machine that pulls images and starts, stops, and supervises containers. The runtime used inside kind nodes is containerd. |
| Registry | A server that stores and distributes container images. `nginx:1.30-alpine` is pulled from the default public registry, Docker Hub. |
| Image tag | The text after the colon in an image reference, identifying a specific version, for example `1.30-alpine`. Pinning a tag makes a deployment reproducible; the floating tag `latest` does not. |

### 1.2 Cluster terminology

| Term | Definition |
| --- | --- |
| Kubernetes | An open-source system that runs containers across a group of machines, decides where each container should run, restarts containers that fail, and exposes them on the network. |
| Cluster | The complete set of machines managed together by Kubernetes, plus the Kubernetes software running on them. |
| Node | A single machine, physical or virtual, that belongs to a cluster and can run containers. In kind each node is itself a Docker container. |
| Control plane | The set of components that make cluster-wide decisions and store cluster state. A node that runs those components is called a control-plane node. |
| Worker node | A node whose purpose is to run application workloads. |
| kube-apiserver | The control-plane component that exposes the Kubernetes HTTP API. Every read and every write to the cluster passes through it. It is the only component that talks to etcd. |
| etcd | The distributed key-value store that holds the entire desired and observed state of the cluster. Losing etcd means losing the cluster's memory. |
| kube-scheduler | The control-plane component that watches for Pods with no assigned node and selects a suitable node for each one. |
| kube-controller-manager | The control-plane component that runs the control loops which drive actual cluster state towards the declared desired state, for example by creating replacement Pods. |
| kubelet | The agent running on every node. It receives the list of Pods assigned to its node and instructs the container runtime to start and stop the corresponding containers. It also runs health probes. |
| kube-proxy | The component on every node that programs the node's packet-forwarding rules so that traffic sent to a Service address reaches one of the Pods behind that Service. |
| CNI plugin | The networking component that gives every Pod an IP address and makes Pod-to-Pod traffic routable across nodes. kind installs its own plugin, kindnet, automatically. |
| Add-on | A cluster component that is not part of the control plane proper but is required for a usable cluster, such as the in-cluster DNS server CoreDNS. |
| kind | A tool that creates a Kubernetes cluster by running each node as a Docker container on one host machine. It is intended for development, teaching, and automated testing, not production. |

### 1.3 API and object terminology

| Term | Definition |
| --- | --- |
| Object | A persistent record in the cluster describing something the cluster should have or does have, for example a Pod or a Service. Objects are stored in etcd and read or written through the API server. |
| Resource | The API endpoint through which a category of object is managed, for example `pods` or `services`. The two words are often used interchangeably in practice. |
| Manifest | A YAML or JSON file describing one or more objects. Manifests are the files committed to version control. |
| Desired state | What the manifest says should exist. |
| Observed state | What the cluster reports actually exists, held in each object's `status` section. |
| Reconciliation | The continuous process by which controllers compare observed state against desired state and act to close the gap. |
| `apiVersion` | The API group and version that defines the object's schema, for example `v1` or `apps/v1`. |
| `kind` | The type of object being described, for example `Pod` or `Deployment`. This field is unrelated to the tool named kind. |
| `metadata` | Identifying information for an object: its `name`, its `namespace`, its `labels`, and its `annotations`. |
| `spec` | The desired state section, written by the author of the manifest. |
| `status` | The observed state section, written by the cluster and never by the author. |
| Namespace | A named partition of the cluster into which most objects are placed. Object names must be unique within a namespace, not across the whole cluster. |
| Label | A key-value pair attached to an object's metadata, used to group objects, for example `app: web`. |
| Selector | A query over labels, used by one object to identify the set of objects it applies to. A Service uses a selector to find its Pods. |
| Annotation | A key-value pair attached to an object's metadata to carry descriptive information for humans or tools. Annotations are never used for selection. |
| `kubectl` | The official command-line client for the Kubernetes API. |
| kubeconfig | The file, by default `~/.kube/config`, that stores cluster addresses, credentials, and contexts for kubectl. |
| Context | A named combination of cluster, user, and default namespace inside a kubeconfig file. Switching context switches which cluster kubectl talks to. |

### 1.4 Workload terminology

| Term | Definition |
| --- | --- |
| Pod | The smallest object that Kubernetes schedules and runs. A Pod holds one or more containers that share a network address, storage volumes, and a lifecycle. Containers in the same Pod always run on the same node. |
| Volume | A directory made available to the containers in a Pod. An `emptyDir` volume is created empty when the Pod starts and is deleted when the Pod is removed. |
| Probe | A periodic diagnostic performed by the kubelet against a container to determine its health. |
| Startup probe | A probe that runs first and determines when a slow-starting container has finished starting. While it is running, the liveness and readiness probes are suspended. |
| Readiness probe | A probe that determines whether a container is ready to receive network traffic. Failure removes the Pod from Service traffic but does not restart it. |
| Liveness probe | A probe that determines whether a container is still functioning. Failure causes the kubelet to restart the container. |
| ReplicaSet | A controller object that keeps a stated number of identical Pods running. It is normally created and managed by a Deployment rather than written by hand. |
| Deployment | A controller object that manages ReplicaSets in order to provide declarative updates, rolling releases, and rollbacks for a stateless application. |
| Service | An object that gives a stable name and IP address to a changing set of Pods, and load-balances traffic across them. |
| ClusterIP | The default Service type. It is reachable only from inside the cluster. |
| NodePort | A Service type that additionally opens the same high-numbered port on every node, making the Service reachable from outside the cluster. |
| EndpointSlice | An object, generated automatically, listing the network addresses currently behind a Service. It is the modern replacement for the deprecated Endpoints object. |
| ResourceQuota | A namespace-scoped object that caps the total compute resources and object counts that the namespace may consume. |
| LimitRange | A namespace-scoped object that supplies default and boundary values for container resource requests and limits within that namespace. |
| Request | The amount of CPU or memory a container is guaranteed. The scheduler uses requests to decide whether a node has room for a Pod. |
| Limit | The maximum amount of CPU or memory a container may use. Exceeding a memory limit terminates the container; exceeding a CPU limit throttles it. |

---

## 2. Two Ways to Talk to the Cluster

Kubernetes can be driven in two styles. Both are examined in DSO202, and this practical uses both deliberately.

| | Imperative | Declarative |
| --- | --- | --- |
| Form of the instruction | A command stating an action: "create this Pod" | A file stating an outcome: "a Pod with these properties should exist" |
| Example | `kubectl run web --image=nginx:1.30-alpine` | `kubectl apply -f manifests/02-pod-web.yaml` |
| Where the truth lives | In the terminal history of one person | In a file, in version control |
| Repeatability | Poor; the exact command must be remembered and retyped | Exact; the same file produces the same result |
| Reviewable before execution | No | Yes, through normal code review |
| Suitable for | Quick experiments, inspection, one-off debugging | Anything that must be reproduced, graded, or deployed again |

The rule applied throughout DSO202: **imperative commands are for exploring, declarative manifests are for delivering.** Marks for configuration implementation are awarded on the manifests, not on terminal history.

A useful bridge between the two styles is `--dry-run=client -o yaml`, which asks kubectl to build the object it *would* have sent and print it instead of sending it. This converts an imperative command into a starting-point manifest, and is used in Stage 4.

---

## 3. Troubleshooting

The first three commands to run for almost any problem, in this order:

```bash
kubectl get pods -o wide # what state is it actually in
kubectl describe pod <name> # read the Events section at the bottom
kubectl logs <name> [--previous] [-c <name>] # what did the application itself say
```

### 3.1 Cluster creation

| Symptom | Cause | Resolution |
| --- | --- | --- |
| `ERROR: failed to create cluster: node(s) already exist for a cluster with the name "dso202"` | A previous cluster, possibly a failed one, still exists | `kind delete cluster --name dso202`, then create again |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Docker is not running | Start Docker Desktop, or `sudo systemctl start docker`. On Linux, add the account to the `docker` group and open a new shell |
| `ERROR: failed to create cluster: command "docker run ..." failed` with a port-binding error | Host port 30080 is already in use | Stop whatever holds the port, or change both `hostPort` in Listing 1 and `nodePort` in Listing 10 to a free value |
| Creation stops at `Starting control-plane` and times out | Insufficient memory or CPU allocated to Docker | Raise Docker's memory allocation to at least 4 GB and CPUs to at least 2, then recreate |
| `WARNING: cgroups v1 detected` | The host kernel is using the older cgroup interface | The cluster will still start. Kubernetes has deprecated cgroup v1; upgrading the host distribution is the long-term fix |
| `invalid configuration: ... unknown field` in `kubeadmConfigPatches` | The patch does not match the kubeadm API version in use | kind uses kubeadm `v1beta4` for Kubernetes 1.36 and later. Listing 1 deliberately omits `apiVersion` from its patches so that kind converts them automatically. Do not add an `apiVersion` line |
| Creation fails while pulling `kindest/node:v1.36.1@sha256:...`, or the cluster starts but behaves oddly, and `kind version` reports a release newer than v0.32.0 | Node images are not guaranteed to be compatible across kind releases, and Listing 1 pins one built for v0.32.0 | Delete the three `image:` lines from Listing 1 so that kind selects its own default image, then recreate. Record the substituted Kubernetes version in the report's Environment section |
| `kubectl` reports `connection refused` after a host reboot | The node containers are stopped, not deleted | `docker start dso202-control-plane dso202-worker dso202-worker2`, wait for `kubectl get nodes` to report `Ready` |

### 3.2 Node and naming

| Symptom | Cause | Resolution |
| --- | --- | --- |
| `kubectl get nodes` shows `dso202-control-plane` instead of `control-plane` | The `kubeadmConfigPatches` blocks in Listing 1 were omitted or mis-indented | Compare against Listing 1, delete the cluster, and recreate. The patch must be indented under the node entry it belongs to |
| Cluster creation fails only when the name patches are present | The kind release in use handles the node-name patch differently | Use **Listing 1B**, the fallback configuration in the companion file, and record the substitution in the report's Reflection section. The remaining stages are unaffected; only the node names differ |
| One worker reports `NotReady` | The kubelet in that container has not finished starting, or the host is short of memory | Wait 60 seconds; then `docker logs dso202-worker`, and `kubectl describe node worker-node-1` to read the node conditions |

### 3.3 Pods

| Symptom | Cause | Resolution |
| --- | --- | --- |
| `Pending` with event `0/3 nodes are available: insufficient memory` | The requests cannot be satisfied by any node | Lower `requests` in the manifest, or reduce the replica count |
| `Pending` with event `... node(s) didn't match Pod's node affinity/selector` | A `nodeSelector` names a label no node carries | Correct the selector, or label a node |
| `Pending` with no events at all | The API server accepted the object but the scheduler has not acted | Check that `kube-scheduler-control-plane` is `Running` in `kube-system` |
| `ImagePullBackOff` | The image name or tag is wrong, or there is no network access to the registry | `kubectl describe pod <name>` names the exact image string attempted. Verify the tag exists, and verify host internet access |
| `CrashLoopBackOff` | The container's main process exits shortly after starting | `kubectl logs <name> --previous`. A container whose command completes is treated as a crash; long-running containers must not exit |
| `CreateContainerConfigError` | A referenced ConfigMap, Secret, or key does not exist | The describe output names the missing object |
| `Error from server (Forbidden): ... exceeded quota` | A ResourceQuota cap has been reached | `kubectl describe resourcequota dso202-quota` and compare `Used` against `Hard` |
| `must specify limits.memory` on apply | The quota caps `limits.memory`, so every container must declare it | Add the declaration to the manifest |
| `Running` but `0/1` | The readiness probe is failing | `kubectl describe pod <name>`; the `Unhealthy` warning gives the probe type, mechanism, and status code |
| `RESTARTS` climbing steadily | The liveness probe is failing, or the process is exiting | `kubectl logs <name> --previous`, and check whether the probe timing is too aggressive |
| `Terminating` for more than a minute | A container is ignoring the termination signal | `kubectl delete pod <name> --force --grace-period=0` as a last resort only, and record why it was needed |

### 3.4 kubectl and connectivity

| Symptom | Cause | Resolution |
| --- | --- | --- |
| `No resources found in default namespace.` | The default namespace of the current context is not `dso202-practical` | `kubectl config set-context --current --namespace=dso202-practical` |
| `The connection to the server ... was refused` | The cluster is deleted or stopped, or the wrong context is selected | `kind get clusters`, then `kubectl config get-contexts` and `kubectl config use-context kind-dso202` |
| `error: a container name must be specified for pod <name>` | The Pod has more than one container | Add `-c <container-name>`; the error message lists the valid names |
| `exec: "bash": executable file not found in $PATH` | The image is Alpine-based | Use `-- sh` instead of `-- bash` |
| `curl http://localhost:30080` refuses the connection | The port was not published, or no Pod is ready behind the Service | Confirm `hostPort: 30080` in Listing 1, confirm `nodePort: 30080` in Listing 10, and confirm the EndpointSlice is not empty |
| DNS name does not resolve from a Pod | The name, the namespace, or CoreDNS itself | Check spelling against `kubectl get service`, try the fully qualified name, and confirm the two CoreDNS Pods are `Running` in `kube-system` |
| `Warning: v1 Endpoints is deprecated in v1.33+` | `kubectl get endpoints` was used | Use `kubectl get endpointslice -l kubernetes.io/service-name=<service>`. The warning is informational, not an error |
| `error: error validating data: ... unknown field` on apply | A field name is misspelled, or is at the wrong indentation level | `kubectl explain <kind>.<path>` gives the authoritative field list for the running cluster version |
| `error converting YAML to JSON: yaml: line N: did not find expected key` | Indentation is inconsistent, or a tab character is present | Open line N. Configure the editor to show whitespace and to insert two spaces for the Tab key |

### 3.5 When nothing else helps

```bash
kubectl get events --sort-by=.lastTimestamp | tail -30 # cluster-wide recent history
kubectl get pods -A -o wide # every namespace, with node placement
kubectl describe node worker-node-1 | sed -n '/Conditions/,/Addresses/p'
docker logs dso202-worker --tail 50 # the kubelet's own output
kind export logs ./evidence/kind-logs # a complete diagnostic bundle
```

`kind export logs` writes the logs of every node, every control-plane component, and the container runtime into a directory. Attach it to any request for help from the module tutor.

---

## 4. Command Reference

### 4.1 Cluster lifecycle

| Command | Purpose |
| --- | --- |
| `kind create cluster --config cluster/kind-cluster.yaml` | Create the cluster described by the file |
| `kind get clusters` | List kind clusters on this host |
| `kind get nodes --name dso202` | List the Docker container names of a cluster's nodes |
| `kind delete cluster --name dso202` | Delete the cluster and its kubeconfig context |
| `kind export logs ./evidence/kind-logs` | Write a full diagnostic bundle |
| `kubectl cluster-info` | Print the API server and CoreDNS addresses |
| `kubectl config get-contexts` | List available contexts |
| `kubectl config use-context kind-dso202` | Switch clusters |
| `kubectl config set-context --current --namespace=<ns>` | Set the default namespace |
| `kubectl api-resources` | List every object kind the cluster supports |
| `kubectl explain <kind>.<field>` | Read the schema from the live cluster |

### 4.2 Reading objects

| Command | Purpose |
| --- | --- |
| `kubectl get <kind>` | List objects of one kind |
| `kubectl get <kind> -o wide` | Add node, IP, and image columns |
| `kubectl get <kind> <name> -o yaml` | Print the full stored object |
| `kubectl get <kind> -o jsonpath='{...}'` | Extract one field |
| `kubectl get <kind> --show-labels` | Add a labels column |
| `kubectl get <kind> -l key=value` | Filter by label |
| `kubectl get <kind> --watch` | Stream changes until interrupted |
| `kubectl get all` | List common workload and Service objects only |
| `kubectl describe <kind> <name>` | Human-readable detail, including events |
| `kubectl get events --sort-by=.lastTimestamp` | Cluster history in order |
| `kubectl top pod` | Live CPU and memory use. Requires metrics-server, which kind does not install; introduced in Unit V |

### 4.3 Creating and changing objects

| Task | Imperative | Declarative |
| --- | --- | --- |
| Namespace | `kubectl create namespace <ns>` | `kubectl apply -f 00-namespace.yaml` |
| Pod | `kubectl run <name> --image=<img> --restart=Never` | `kubectl apply -f 02-pod-web.yaml` |
| Deployment | `kubectl create deployment <name> --image=<img> --replicas=3` | `kubectl apply -f 06-deployment-web.yaml` |
| Service from a Deployment | `kubectl expose deployment <name> --port=80 --target-port=80` | `kubectl apply -f 07-service-clusterip.yaml` |
| Service standalone | `kubectl create service nodeport <name> --tcp=80:80` | `kubectl apply -f 08-service-nodeport.yaml` |
| Scale | `kubectl scale deployment <name> --replicas=5` | Edit `replicas` and re-apply |
| Change image | `kubectl set image deployment/<name> <container>=<img>` | Edit `image` and re-apply |
| Add a label | `kubectl label pod <name> key=value` | Edit `metadata.labels` and re-apply |
| Delete | `kubectl delete <kind> <name>` | `kubectl delete -f <file>` |
| Preview a manifest | `kubectl <command> --dry-run=client -o yaml` | `kubectl diff -f <file>` |
| Restart a Deployment | `kubectl rollout restart deployment/<name>` | Not applicable |

### 4.4 Rollouts

| Command | Purpose |
| --- | --- |
| `kubectl rollout status deployment/<name>` | Block until the rollout finishes or fails |
| `kubectl rollout history deployment/<name>` | List revisions and change causes |
| `kubectl rollout history deployment/<name> --revision=N` | Show one revision's template |
| `kubectl rollout undo deployment/<name>` | Return to the previous revision |
| `kubectl rollout undo deployment/<name> --to-revision=N` | Return to a named revision |
| `kubectl rollout pause deployment/<name>` | Suspend an in-progress rollout |
| `kubectl rollout resume deployment/<name>` | Continue a paused rollout |
| `kubectl annotate deployment <name> kubernetes.io/change-cause="..."` | Populate the `CHANGE-CAUSE` column |

### 4.5 Debugging

| Command | Purpose |
| --- | --- |
| `kubectl logs <pod>` | Print a container's log |
| `kubectl logs <pod> -c <container>` | Choose a container in a multi-container Pod |
| `kubectl logs <pod> --previous` | Read the log of the crashed previous instance |
| `kubectl logs <pod> -f --tail=20` | Follow the last 20 lines |
| `kubectl logs -l app=web --prefix` | Aggregate logs from every Pod matching a label |
| `kubectl exec -it <pod> -- sh` | Open a shell in a container |
| `kubectl exec <pod> -- <command>` | Run one command in a container |
| `kubectl port-forward pod/<pod> 8080:80` | Tunnel a host port to a Pod |
| `kubectl port-forward service/<svc> 8080:80` | Tunnel a host port to a Service |
| `kubectl wait --for=condition=Ready pod/<pod> --timeout=60s` | Block until a Pod is ready |

### 4.6 YAML field quick reference

| Field | Applies to | Purpose |
| --- | --- | --- |
| `metadata.labels` | All objects | Key-value pairs that selectors match on |
| `metadata.annotations` | All objects | Descriptive data, never selected on |
| `spec.containers[].resources.requests` | Pod | Guaranteed CPU and memory; used for scheduling |
| `spec.containers[].resources.limits` | Pod | Maximum CPU and memory |
| `spec.containers[].ports[].name` | Pod | A name a Service's `targetPort` can refer to |
| `spec.containers[].startupProbe` | Pod | Determines when a slow start has completed |
| `spec.containers[].readinessProbe` | Pod | Determines whether traffic may be sent |
| `spec.containers[].livenessProbe` | Pod | Determines whether to restart the container |
| `spec.volumes[]` | Pod | Volumes available to the Pod's containers |
| `spec.containers[].volumeMounts[]` | Pod | Where a volume appears inside one container |
| `spec.nodeSelector` | Pod | Restricts placement to nodes carrying given labels |
| `spec.restartPolicy` | Pod | `Always`, `OnFailure`, or `Never` |
| `spec.replicas` | Deployment | Desired number of Pods |
| `spec.selector.matchLabels` | Deployment, Service | Which Pods the object applies to. Immutable on a Deployment |
| `spec.template` | Deployment | The Pod specification to create replicas from |
| `spec.strategy.type` | Deployment | `RollingUpdate` or `Recreate` |
| `spec.strategy.rollingUpdate.maxSurge` | Deployment | Extra Pods permitted during an update |
| `spec.strategy.rollingUpdate.maxUnavailable` | Deployment | Missing Pods permitted during an update |
| `spec.minReadySeconds` | Deployment | How long a new Pod must stay ready before it counts as available |
| `spec.revisionHistoryLimit` | Deployment | How many old ReplicaSets to retain for rollback |
| `spec.type` | Service | `ClusterIP`, `NodePort`, or `LoadBalancer` |
| `spec.ports[].port` | Service | The port clients connect to on the Service |
| `spec.ports[].targetPort` | Service | The Pod port traffic is forwarded to |
| `spec.ports[].nodePort` | Service | The port opened on every node |
| `spec.hard` | ResourceQuota | The caps for the namespace |
| `spec.limits[].default` | LimitRange | Default container limits |
| `spec.limits[].defaultRequest` | LimitRange | Default container requests |

---

*Source: [Extra - HackMD](https://hackmd.io/@sarojsanyasi/HJtoSFwUGl)*