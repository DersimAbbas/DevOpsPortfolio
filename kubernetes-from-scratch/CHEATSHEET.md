# Kubernetes Cheatsheet

Personal notes from the _Course Certified Kubernetes Administrator_ Udemy course, kept alongside the YAML manifests I built during the labs in [`k8/`](./k8).

> **Rule of thumb:** 1 container per Pod — **unless** you need a helper (sidecar) container that must live with the application. Then multiple containers in one Pod is fine.

---

## Quick reference

| Component                   | Lives on | Job                                                                                     |
| --------------------------- | -------- | --------------------------------------------------------------------------------------- |
| **etcd**                    | Master   | Distributed key/value store. Source of truth for cluster state.                         |
| **kube-apiserver**          | Master   | Front door to the cluster. Every component talks through it.                            |
| **kube-scheduler**          | Master   | Decides _which_ node a Pod runs on.                                                     |
| **kube-controller-manager** | Master   | Runs the controllers that reconcile desired vs. actual state.                           |
| **kubelet**                 | Worker   | Node agent. Talks to the API server, manages Pods/containers via the container runtime. |
| **kube-proxy**              | Worker   | Maintains network rules (iptables) so Services can reach the right Pods.                |
| **Container runtime**       | Worker   | Actually runs the containers (containerd, CRI-O, Docker historically).                  |

---

## Cluster architecture

**Master node (control plane):**

- etcd cluster
- kube-apiserver
- kube-controller-manager
- kube-scheduler

**Worker nodes:**

- kubelet
- kube-proxy
- Container runtime engine (containerd, CRI-O, Docker)

**Big picture:**

- _Worker nodes_ host applications as containers (inside Pods).
- _Master node_ plans, schedules, and monitors the workers.
- Kubelet listens for instructions from the kube-apiserver.
- kube-proxy enables communication between Services across worker nodes.

---

## Control plane components

### etcd

- Distributed, reliable **key/value store**.
- Stores **all cluster state**: config, deployments, monitoring, health, recent changes.
- Runs as an isolated server on port **2379** (should be its own Pod).

**etcd v2 vs v3 commands** (they changed between versions — easy to mix up):

| v2                       | v3                        |
| ------------------------ | ------------------------- |
| `etcdctl backup`         | `etcdctl snapshot save`   |
| `etcdctl cluster-health` | `etcdctl endpoint health` |
| `etcdctl set`            | `etcdctl put`             |
| `etcdctl get`            | `etcdctl get`             |
| `etcdctl mk` / `mkdir`   | _(removed)_               |

Basic usage:

```bash
etcdctl put key1 value1
etcdctl get key1
```

### kube-apiserver

- The only component everything else talks to.
- Makes it possible for nodes/Pods to communicate with the rest of the cluster.

### kube-scheduler

- Decides **which Pod goes on which node**.
- Process: **filter** nodes that have the right resources → **rank** them 0–10 → pick the node with the most leftover resources after the Pod is placed.
- Note: the scheduler _decides_, but it's the **kubelet** that actually places the Pod on the node.

### kube-controller-manager

- Bundles many controllers into a single process.
- Each controller continuously **watches** a component's status and **acts** to match the desired state in the manifest.
- Example: a ReplicaSet says "3 replicas" — the controller keeps actual count == 3.
- (ArgoCD is a deployment controller in the same spirit, but external.)

---

## Worker components

### kubelet

- Agent that runs on **every node**.
- Talks to the kube-apiserver — reports status, receives Pod specs.
- Uses the container runtime to actually spin up/tear down containers.

### kube-proxy

- Watches for new Services and updates **iptables rules** on the node so traffic gets forwarded to the right backend Pods.
- Without kube-proxy, Services wouldn't be able to pinpoint Pods across nodes.

---

## Core objects

### Pod

- **Smallest object in Kubernetes.**
- A Pod = one instance of an application.
- Runs containers (usually one, sometimes one + helper sidecars).

### Service

- A **stable endpoint** for talking to Pods.
- Pods are **stateless** — they die, get recreated, get new IPs. Services give you a constant address.
- Acts as the "source of truth" for "where is my app right now?"

### Pod network

- The **virtual internal network** all nodes share so Pods can talk to each other across nodes.

---

## Container runtimes & CRI

- **CRI** = Container Runtime Interface. The contract a runtime must implement to plug into Kubernetes.
- Any vendor can be a Kubernetes container runtime as long as they follow the **OCI** (Open Container Initiative):
  - **imagespec** — how images should be built.
  - **runtimespec** — how a runtime should run them.

### CLIs to know

- `docker` — original Docker CLI (Docker as a runtime is deprecated in K8s, but the CLI is still everywhere).
- `ctr` — low-level containerd CLI. Awkward.
- `nerdctl` — Docker-CLI-compatible front-end for containerd. **More user-friendly, more production-stable** than `ctr`.

---

## Things I want to remember

- Pods are **ephemeral**. Don't rely on Pod IPs — go through a Service.
- Manifests describe **desired state**. Controllers continuously reconcile to reach it.
- Every interaction with the cluster goes through the **kube-apiserver** — including the kubelet's check-ins.
- The scheduler picks the node; the kubelet places the Pod. Two different responsibilities, easy to conflate.
- etcd v2 vs v3 commands are _different_ — check the version before debugging a "command not found".
