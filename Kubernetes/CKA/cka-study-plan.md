# CKA Prep Plan — Zero to Certified

**Timeline:** ~14-16 weeks, ~8-10 hrs/week (adjust to your pace)
**Exam:** 2 hours, 15-20 hands-on tasks, live terminal, no partial credit
**Budget:** $445 exam (includes 1 free retake) + killer.sh access (comes free with voucher)

---

## Phase 1: Foundations (Weeks 1-3)
Goal: understand what K8s *is* before touching `kubectl` seriously.

- [ ] K8s architecture: control plane vs worker nodes, API server, etcd, scheduler, controller manager, kubelet, kube-proxy
- [ ] Core objects: Pods, ReplicaSets, Deployments, Services, Namespaces
- [ ] Set up local lab: `kind` or `minikube` (kind is closer to real multi-node behavior, recommended)
- [ ] Course: KodeKloud "Kubernetes for Beginners" or the free CNCF "Kubernetes and Cloud Native Associate" material
- [ ] Practice: create/delete pods, deployments, services purely via `kubectl` — no YAML copy-paste, type it yourself

**Checkpoint:** You can explain what happens end-to-end when you run `kubectl apply -f deployment.yaml` without looking anything up.

---

## Phase 2: Exam Domains, Deep Dive (Weeks 4-10)
CKA exam weight by domain — study time should roughly match:

| Domain | Exam Weight | Weeks |
|---|---|---|
| Troubleshooting | 30% | 2 |
| Cluster Architecture, Installation & Config | 25% | 2 |
| Services & Networking | 20% | 1.5 |
| Workloads & Scheduling | 15% | 1 |
| Storage | 10% | 0.5 |

### Troubleshooting (heaviest weight — don't skimp)
- Debug broken pods (crashloop, image pull errors, resource limits)
- Node not ready — check kubelet logs, `journalctl`
- Network policy misconfigurations blocking traffic
- Cluster component failures — control plane pod logs in `kube-system`
- Application logs, `kubectl describe`, `kubectl logs -p` fluency

### Cluster Architecture & Installation
- `kubeadm` cluster setup **from scratch** (bootstrap a cluster manually, don't rely on managed K8s for this)
- Static pods, node maintenance (`cordon`, `drain`, `uncordon`)
- RBAC — Roles, RoleBindings, ClusterRoles, ServiceAccounts (**this maps directly to your supply chain security background** — least-privilege access design is the same muscle)
- etcd backup/restore (this trips people up under time pressure — drill it repeatedly)

### Services & Networking
- ClusterIP, NodePort, LoadBalancer, Ingress
- CNI basics (don't need to master a specific plugin, just understand the model)
- NetworkPolicies — again, overlaps with your background (segmentation/least-privilege at the network layer)
- DNS in-cluster (CoreDNS troubleshooting)

### Workloads & Scheduling
- Deployments, DaemonSets, Jobs/CronJobs
- Resource requests/limits, taints/tolerations, node affinity
- ConfigMaps and Secrets (pay attention to how Secrets are actually stored — relevant to your interest in supply chain/secrets exposure)

### Storage
- PersistentVolumes, PersistentVolumeClaims, StorageClasses
- Volume mount types — lighter weight, don't overinvest here

---

## Phase 3: Speed & Exam Simulation (Weeks 11-14)
This is where most failures happen — not lack of knowledge, but running out of time.

- [ ] Master `kubectl` imperative commands (`--dry-run=client -o yaml` to generate YAML fast — you will NOT have time to write manifests from memory)
- [ ] Set up `kubectl` aliases and autocomplete exactly as you'll have them on exam day (`k` alias, `kubectl completion bash`)
- [ ] Bookmark exam-allowed docs (kubernetes.io) — practice *navigating* them fast, since you're allowed to reference them live
- [ ] Do **killer.sh** simulator (free with your exam voucher, 2 attempts) — it's harder than the real exam on purpose. Take it once early (diagnostic), once ~1 week before exam (readiness check)
- [ ] Timed practice: solve 15-20 random scenario tasks in under 2 hours, repeatedly, until it's not sweaty

**Checkpoint:** You can generate a Deployment, expose it as a Service, and troubleshoot a broken pod inside 15 minutes total, without hesitating on syntax.

---

## Phase 4: Exam Week (Weeks 15-16)
- [ ] Final killer.sh run
- [ ] Review your personal "gotcha list" — mistakes you kept repeating in practice
- [ ] Re-drill etcd backup/restore and RBAC one more time (both are common last-minute forgets)
- [ ] Rest the day before — this exam punishes fatigue-driven typos more than knowledge gaps

---

## After CKA: Natural Next Step
Once certified, **CKS (Kubernetes Security Specialist)** requires an active CKA and builds directly on this — image scanning, admission controllers, supply chain attestation, runtime security. Given your background, that's likely the higher-leverage cert to chase next rather than a general K8s deep dive.

## Resources
- KodeKloud CKA course (labs-heavy, closest to exam format)
- Kubernetes the Hard Way (optional, deepens architecture understanding but not required)
- kubernetes.io official docs (practice navigating — you'll use these live in the exam)
- killer.sh (comes free with exam registration, use both attempts strategically)
