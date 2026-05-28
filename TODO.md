# Kubernetes Learning Roadmap

Personal learning plan, sequenced from immediate practical wins to long-term DevOps/Cloud portfolio pieces. The goal is **declarative-first** Kubernetes fluency, layered with Ansible + Terraform skills as a DevOps/Cloud engineer.

---

## Phase 1 — Bridge local repo to the cluster (today, ~15 min)

- [ ] Install `kubectl` on Windows
- [ ] Copy admin kubeconfig from `controlplane01` to `C:\Users\derab\.kube\config`
- [ ] Edit the `server:` line to point to the loadbalancer (`https://192.168.56.30:6443`)
- [ ] Verify: `kubectl get nodes` works from PowerShell
- [ ] Test applying a manifest from the local repo: `kubectl apply -f .\k8\pods\pod-definition.yml`

**Why:** This is how real workplaces deploy — devs apply YAML from their laptop, not by SSHing into masters.

---

## Phase 2 — Fearless VM destruction (~2 min)

- [ ] Take a Vagrant snapshot of the current working cluster:
  ```bash
  vagrant snapshot save baseline
  ```
- [ ] Document the restore command: `vagrant snapshot restore baseline`
- [ ] Take a fresh snapshot after each major milestone

**Why:** Gets ~80% of the "I can break things safely" benefit with ~2% of the effort of Ansible automation.

---

## Phase 3 — Declarative YAML cookbook (1–2 weeks, ongoing)

Build a `manifests/` directory in this repo. One folder per object kind, each containing:

- a **minimal** working manifest
- a **maxed-out, annotated** manifest with comments on every field
- short notes on what's mandatory / optional / defaulted

Topics in suggested order:

- [ ] `01-pod/` — anatomy: apiVersion, kind, metadata, spec.containers
- [ ] `02-deployment/` — replicas, selector, template, strategy, rollout
- [ ] `03-service/` — ClusterIP, NodePort, LoadBalancer, selectors, ports
- [ ] `04-configmap-secret/` — env vars, volume mounts, base64 secrets
- [ ] `05-ingress/` — rules, paths, TLS, ingress class
- [ ] `06-pvc-pv/` — storage classes, access modes, reclaim policy
- [ ] `07-job-cronjob/` — completions, parallelism, backoffLimit, schedule
- [ ] `08-namespace-resourcequota/` — multi-tenancy basics
- [ ] `09-rbac/` — Role, ClusterRole, RoleBinding, ServiceAccount

**Why:** This is the K8s equivalent of how Pipelines became muscle memory — by collecting and annotating real templates until the shape is intuitive.

---

## Phase 4 — Learn Ansible properly (before touching kubespray)

Don't jump into kubespray yet. Build the alphabet first.

- [ ] Write a basic playbook against the Vagrant VMs: install `nginx` on `node01`, prove inventory + ssh + privilege escalation works
- [ ] Add a role: extract the nginx install into `roles/nginx/`
- [ ] Add a handler: restart nginx on config change
- [ ] Add a template: jinja2-templated nginx.conf with vars
- [ ] Add group_vars and host_vars to differentiate behavior per node
- [ ] Document the playbook in this repo (`ansible/`)

**Why:** Reading kubespray's codebase is unreadable without knowing what inventories, roles, handlers, and templates are. Build the alphabet before reading the novel.

---

## Phase 5 — Write a minimal K8s Ansible playbook yourself

Build a *toy kubespray* — your own ~200-line Ansible role to install K8s.

- [ ] Playbook 1: install containerd on all nodes
- [ ] Playbook 2: install kubeadm/kubelet/kubectl packages
- [ ] Playbook 3: `kubeadm init` on controlplane, `kubeadm join` on workers
- [ ] Playbook 4: install a CNI (Calico or Flannel) as a final task
- [ ] Test: `vagrant destroy && vagrant up`, then run the playbook — get a working cluster from scratch

**Why:** Now you have a baby kubespray. You understand the shape. When you read the real one, it will be *the same shape, with production polish around it* — not magic.

---

## Phase 6 — Read kubespray (not run it)

- [ ] Clone `kubernetes-sigs/kubespray`
- [ ] Read `cluster.yml` top-to-bottom — see how it sequences phases
- [ ] Pick 3 roles and read them deeply: `roles/etcd`, `roles/kubernetes/master`, `roles/network_plugin/calico`
- [ ] For each, diff mentally against:
  - what you did manually in *Kubernetes the Hard Way*
  - what you wrote yourself in Phase 5
- [ ] Note 5 things kubespray handles that your minimal version doesn't (idempotency, OS variance, error recovery, cert rotation, etc.)

**Why:** Ansible is *designed* to be readable — declarative YAML, not opaque binaries. Treat kubespray as a textbook on production K8s install, not a black box.

---

## Phase 7 — Run kubespray on your Vagrant VMs

Now you can press the magic button without it being magic.

- [ ] `vagrant snapshot save pre-kubespray` (safety net)
- [ ] `vagrant destroy && vagrant up` for a clean slate (or use fresh VMs)
- [ ] Configure `inventory/mycluster/hosts.yaml` with your 4 VMs
- [ ] Configure `group_vars/` — pick K8s version, CNI, container runtime
- [ ] Run `ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -vvv` (verbose so you see every task)
- [ ] Compare the resulting cluster against your hard-way cluster — what's different? Why?

**Why:** By this point you can predict what each task is doing before kubespray prints it. The verbose flag turns it into a live lecture.

---

## Phase 8 — Layer Terraform underneath (portfolio piece)

The full DevOps pipeline: infra → cluster → apps.

- [ ] Terraform module that provisions 4 VMs (libvirt locally, or AWS free tier)
- [ ] Output: an Ansible inventory file consumable by kubespray
- [ ] Kubespray installs K8s on those Terraform-created VMs
- [ ] ArgoCD or Flux pulls the `manifests/` cookbook from Phase 3 and deploys it

**Why:** This is a strong DevOps/Cloud Engineer portfolio piece. CV line: *"Built an automated bare-VM → production K8s pipeline using Terraform + kubespray + ArgoCD."*

---

## Notes

- Phases 1–3 can run in parallel with daily work — they're cheap and immediately useful.
- Phases 4–7 are a focused sequence: don't skip ahead, the order matters for understanding.
- Phase 8 is the long-term project, do it once everything before is solid.
- Snapshot after every major milestone. `vagrant destroy` is cheap when Phase 5+ exists.
