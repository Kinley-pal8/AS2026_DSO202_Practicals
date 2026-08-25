# DSO202 Practical 1: Local Kubernetes Cluster with kind

## Purpose

This repository sets up a three-node Kubernetes cluster locally using kind
(Kubernetes IN Docker), then deploys and inspects a static nginx web server
through a Namespace, ResourceQuota, LimitRange, Pod, Deployment, and two
Services (ClusterIP and NodePort). It fulfils DSO202 Practical 1.

## Software versions used

| Software | Version |
| --- | --- |
| Docker | 29.7.2 (Kali GNU/Linux Rolling) |
| kind | v0.32.0 |
| kubectl | v1.36.3 |
| Kubernetes (cluster) | v1.36.1 |

## Repository structure

```text
dso202-practical-01/
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

## Rebuilding from an empty machine

```bash
# 1. Create the three-node cluster
kind create cluster --config cluster/kind-cluster.yaml

# 2. Apply every manifest (lexical order: namespace, quota/limits, pod,
#    deployment, services, client pod)
kubectl apply -f manifests/

# 3. Set the namespace as default for the current context
kubectl config set-context --current --namespace=dso202-practical-01

# 4. Verify
kubectl get all
kubectl rollout status deployment/web-deployment
curl -s http://localhost:30080
```

## Cleanup

```bash
kubectl delete -f manifests/
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202
```
