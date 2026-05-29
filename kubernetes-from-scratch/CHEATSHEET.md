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

## Controllers: ReplicationController → ReplicaSet → Deployment

These are the three workload controllers that keep Pods alive. They build on each other — know all three for the exam, even though you'll only really use Deployments day-to-day.

### ReplicationController (RC) — the original

- The **older** way of running N copies of a Pod.
- Ensures the specified number of replicas is always running. If a Pod dies, RC spins up a new one.
- Supports only **equality-based label selectors** (`key=value`).
- `selector` field is **optional** — if you omit it, RC inherits and monitor already existing / created pods from the Pod template labels.
- API: `apiVersion: v1`, `kind: ReplicationController`.

### ReplicaSet (RS) — the newer replacement

- Same job as RC: keep N Pods running.
- Adds **set-based selectors** (`key in (a, b, c)`, `key notin (...)`, `key`, `!key`) — much more flexible.
- `selector` field is **required**, and it must match the Pod template labels (the API server will reject the manifest otherwise).
- API: `apiVersion: apps/v1`, `kind: ReplicaSet`.

> **Rule of thumb:** never reach for RC in new manifests. RS is a strict superset.

### Deployment — the one you actually use

- A **higher-level controller** that manages ReplicaSets for you.
- Adds the things you actually need in production:
  - **Rolling updates** — replace Pods gradually instead of all at once.
  - **Rollbacks** — `kubectl rollout undo` reverts to the previous ReplicaSet.
  - **Revision history** — every change creates a new RS; old ones are kept (default: 10) so you can roll back.
  - **Pause/resume** rollouts mid-flight.
- API: `apiVersion: apps/v1`, `kind: Deployment`.

**How Deployment relates to RS:** a Deployment _owns_ a ReplicaSet, which _owns_ the Pods. When you change the Pod template in a Deployment, it creates a **new** RS, scales it up while scaling the old RS down, and keeps the old one around (at 0 replicas) for rollback.

### Why would you ever use a ReplicaSet on its own?

Honestly, almost never in production. But the exam likes to test the distinction, and there are a few cases:

- **You don't need rolling updates or rollbacks** — e.g., a stateless workload you'll never change after deploy.
- **You're writing a custom controller** that manages Pod lifecycle itself and just wants the "keep N alive" guarantee from RS without the Deployment's rollout machinery on top.
- **CKA exam tasks** — they sometimes ask you to write/fix an RS manifest directly to test that you understand the selector/template relationship without the Deployment doing it for you.

> **Rule of thumb:** for any real app, use a Deployment. Use a bare ReplicaSet only when you _actively don't want_ the Deployment's update behavior.

---

## kubectl commands cheat sheet

### Creating & updating

| Command                                | What it does                                                                 |
| -------------------------------------- | ---------------------------------------------------------------------------- |
| `kubectl create -f manifest.yaml`      | Create the object. **Fails** if it already exists.                           |
| `kubectl apply -f manifest.yaml`       | Create or update (declarative). Use this most of the time.                   |
| `kubectl replace -f manifest.yaml`     | Replace an existing object with the manifest. **Fails** if it doesn't exist. |
| `kubectl replace --force -f file.yaml` | Delete and recreate. Useful when a field is immutable (e.g. RS `selector`).  |
| `kubectl delete -f manifest.yaml`      | Delete whatever the manifest describes.                                      |
| `kubectl edit <kind> <name>`           | Open the live object in `$EDITOR`. Saving applies the change.                |

> `create` vs `apply`: `create` is imperative (one-shot). `apply` is declarative — it tracks the last-applied config and merges changes. Pick one style per workflow and stick with it.

### Reading

| Command                                 | What it shows                                              |
| --------------------------------------- | ---------------------------------------------------------- |
| `kubectl get pods`                      | List Pods in the current namespace.                        |
| `kubectl get pods -o wide`              | Adds node, Pod IP, etc.                                    |
| `kubectl get pods -A`                   | All namespaces.                                            |
| `kubectl get rc`                        | ReplicationControllers.                                    |
| `kubectl get rs` (or `replicaset`)      | ReplicaSets.                                               |
| `kubectl get deployments` (or `deploy`) | Deployments.                                               |
| `kubectl get all`                       | Pods, Services, Deployments, RS, etc. in one shot.         |
| `kubectl describe pod <name>`           | Detailed status + recent events. First stop for debugging. |
| `kubectl describe deploy <name>`        | Same, for a Deployment.                                    |
| `kubectl logs <pod>`                    | Stdout/stderr from the Pod's container.                    |
| `kubectl logs <pod> -c <container>`     | Specific container in a multi-container Pod.               |
| `kubectl logs -f <pod>`                 | Follow (tail).                                             |

### Scaling

| Command                                        | What it does                      |
| ---------------------------------------------- | --------------------------------- |
| `kubectl scale --replicas=5 rs/<name>`         | Scale an RS imperatively.         |
| `kubectl scale --replicas=5 deployment/<name>` | Scale a Deployment imperatively.  |
| `kubectl scale --replicas=5 -f manifest.yaml`  | Scale via the manifest reference. |

> Imperative `scale` is fine in the moment, but it won't update your YAML — next `apply` will revert it. For permanent changes, edit the manifest and `apply`.

### Rollouts (Deployments only)

| Command                                              | What it does                        |
| ---------------------------------------------------- | ----------------------------------- |
| `kubectl rollout status deploy/<name>`               | Watch a rollout finish.             |
| `kubectl rollout history deploy/<name>`              | List previous revisions.            |
| `kubectl rollout undo deploy/<name>`                 | Roll back to the previous revision. |
| `kubectl rollout undo deploy/<name> --to-revision=N` | Roll back to a specific revision.   |
| `kubectl rollout pause deploy/<name>`                | Pause an in-flight rollout.         |
| `kubectl rollout resume deploy/<name>`               | Resume it.                          |

### Quick imperative shortcuts (handy on the exam)

```bash
# Generate a Pod manifest without creating it
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Generate a Deployment manifest
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Explain a field — invaluable when you forget the YAML schema
kubectl explain pod.spec.containers
kubectl explain replicaset.spec --recursive
```

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

## Production-shape ingress flow (local Vagrant cluster)

The traffic path I built for the hard-way cluster (2 control planes, 2 workers, 1 LB node):

```
browser
   │  http://<app>.192.168.56.30.nip.io
   ▼
HAProxy on LB node (:80)            ← one stable entrypoint
   │  round-robin, health-checked
   ▼
worker:<ingress NodePort>           ← e.g. 31665 (HTTP), 30439 (HTTPS)
   │
   ▼
ingress-nginx controller pod        ← matches Host header against Ingress rules
   │
   ▼
ClusterIP Service                   ← stable in-cluster name
   │
   ▼
app pod(s)
```

**Why this shape:**

- HAProxy config stays **static** — just `worker1:NodePort` and `worker2:NodePort`. Adding a new app never touches it.
- Routing decisions live in **Ingress resources**, which is real Kubernetes YAML.
- App Services are **ClusterIP only** — no per-app NodePort, no extra surface area.
- Two layers of load balancing: HAProxy (across workers) → kube-proxy (across pods).

### Components — install vs. write

| Layer                          | Install once                                                     | Write per app                        |
| ------------------------------ | ---------------------------------------------------------------- | ------------------------------------ |
| HAProxy on LB node             | `/etc/haproxy/haproxy.cfg` (one frontend + one backend on `:80`) | —                                    |
| Ingress **controller**         | `kubectl apply -f .../ingress-nginx/.../baremetal/deploy.yaml`   | —                                    |
| Ingress **resource**           | —                                                                | One per app (`host:` → service name) |
| Deployment + ClusterIP Service | —                                                                | Per app                              |

> **Controller vs. resource:** Controller = the running program (like `apt install nginx`). Resource = one routing rule (like a `server { }` block). Same `kubectl apply`, completely different roles.

### HAProxy: add alongside the existing apiserver frontend

The LB already balances `kubectl` traffic on 6443 across control planes. **Add** the ingress frontend; don't replace.

```
frontend kubernetes-ingress-http
    bind 192.168.56.30:80
    mode tcp
    default_backend kubernetes-ingress-backend

backend kubernetes-ingress-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server worker01 192.168.56.21:<ingress-NodePort> check fall 3 rise 2
    server worker02 192.168.56.22:<ingress-NodePort> check fall 3 rise 2
```

Then: `sudo haproxy -c -f /etc/haproxy/haproxy.cfg` → `sudo systemctl reload haproxy`.

### DNS with zero hosts-file editing — `nip.io`

`<anything>.<IP>.nip.io` resolves to that IP. Use the LB IP in the middle, vary the prefix per app:

```
pokemon.192.168.56.30.nip.io   →  192.168.56.30
api.192.168.56.30.nip.io       →  192.168.56.30
```

The `host:` in the Ingress matches on the prefix. No DNS server, no hosts file, no notepad-as-admin.

### Gotchas that bit me

- **Wrong manifest URL for ingress-nginx** — make sure the version tag in the URL exists (`controller-v1.15.1` worked; older guesses 404'd). Pull the current one from `https://kubernetes.github.io/ingress-nginx/deploy/`.
- **Admission webhook timeout** (`context deadline exceeded`) on bare-metal — apiserver can't reach the webhook pod IP because CNI routing isn't symmetric. Lab workaround: `kubectl delete validatingwebhookconfiguration ingress-nginx-admission`. Controller still works fine; you just lose YAML pre-validation.
- **`ingressClassName` must match** the installed controller's IngressClass (`kubectl get ingressclass` → usually `nginx`, **not** `nginx-example`).
- **YAML list mistake** — `rules:` must start with `- host:` (it's a list, not a map). If you forget the dash you get `cannot unmarshal object into Go struct field IngressSpec.spec.rules of type []v1.IngressRule`.
- **Service selector is an AND** — pods must have _all_ labels in the selector. Selector `app: pokedex, type: frontend` but pod only labeled `app: pokedex` → zero endpoints → ingress returns **503 Service Temporarily Unavailable**.
- **Random NodePorts drift on reinstall** — pin the ingress controller's NodePort (edit the `ingress-nginx-controller` Service's `nodePort:` to something fixed) so HAProxy doesn't silently break.
- **PowerShell `curl` is `Invoke-WebRequest`** — use `curl.exe -H "Host: x"` for the Unix-style syntax.
- **HAProxy `bind 192.168.56.30:80` doesn't listen on localhost** — only on that specific interface. Test against the actual bound IP from the LB node.
- **`:latest` doesn't auto-update on apply** — `kubectl rollout restart deployment/<name>` forces fresh pull (because `imagePullPolicy` defaults to `Always` for `:latest`).

### Debugging "Ingress isn't working"

Walk the chain from the bottom up. `kubectl get endpoints <service>` is the single best command:

| Symptom                                  | Most likely cause                                             | Check                                                              |
| ---------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------ |
| 503 Service Temporarily Unavailable      | Service has no endpoints (selector mismatch / pods not Ready) | `kubectl get endpoints <svc>` — should list pod IPs                |
| 404 Not Found                            | Ingress rule didn't match the request                         | `kubectl describe ingress` — verify `host:` and `ingressClassName` |
| `connection refused`                     | HAProxy not listening, or wrong port                          | `sudo ss -tlnp \| grep haproxy` on LB node                         |
| `connection refused` only on `localhost` | HAProxy bound to specific IP, not `*`                         | Expected; hit the bound IP instead                                 |
| 400 Bad Request on the HTTPS NodePort    | Sent plain HTTP to a TLS listener                             | Use the HTTP NodePort (port `80:` mapping)                         |

### Adding a new app (5-minute checklist)

1. Write `deployment.yaml` (label your pods uniquely, e.g. `app: <name>`).
2. Write `service.yaml` — `type: ClusterIP`, selector matches **exactly** the pod labels.
3. Write `ingress.yaml` with `host: <name>.192.168.56.30.nip.io` and `ingressClassName: nginx`.
4. `kubectl apply -f <folder>/`.
5. `kubectl get endpoints <svc>` — confirm pods are wired up. Browser → `http://<name>.192.168.56.30.nip.io`.

No HAProxy change. No hosts file change. That's the architectural win.

---

## Things I want to remember

- Pods are **ephemeral**. Don't rely on Pod IPs — go through a Service.
- Manifests describe **desired state**. Controllers continuously reconcile to reach it.
- Every interaction with the cluster goes through the **kube-apiserver** — including the kubelet's check-ins.
- The scheduler picks the node; the kubelet places the Pod. Two different responsibilities, easy to conflate.
- etcd v2 vs v3 commands are _different_ — check the version before debugging a "command not found".
