<!-- GitAds-Verify: REPLACE_WITH_GITADS_CODE -->

# What They Don't Tell You About Kubernetes

[![Sponsored by GitAds](https://gitads.dev/v1/ad-serve?source=atryx/what-they-dont-tell-you-about-kubernetes@github)](https://gitads.dev/v1/ad-track?source=atryx/what-they-dont-tell-you-about-kubernetes@github)

> Brutally honest Kubernetes gotchas, hidden costs, and foot-guns that official docs skip. Lessons learned from production clusters.

**If you've run K8s in production, you've felt the pain. This repo documents it.**

---

## Table of Contents

- [Networking Nightmares](#networking-nightmares)
- [Resource Limits Are Lies](#resource-limits-are-lies)
- [DNS Will Break You](#dns-will-break-you)
- [Storage Is Harder Than You Think](#storage-is-harder-than-you-think)
- [Security Defaults Are Insecure](#security-defaults-are-insecure)
- [Upgrades Are Terrifying](#upgrades-are-terrifying)
- [Costs Nobody Warned You About](#costs-nobody-warned-you-about)
- [Observability Is Not Optional](#observability-is-not-optional)
- [YAML Hell Is Real](#yaml-hell-is-real)
- [The Human Cost](#the-human-cost)
- [Multi-Tenancy Is a Fantasy](#multi-tenancy-is-a-fantasy)
- [Managed K8s Still Hurts](#managed-k8s-still-hurts)
- [Related Repos](#related-repos)
- [Contributing](#contributing)

---

## Networking Nightmares

### 1. Pod-to-pod networking works... until it doesn't

**What happens:** Everything works in dev. In production with 200+ nodes, you get random packet drops, connection resets, and timeouts that are impossible to reproduce.

**Why:** The CNI plugin (Calico, Cilium, Flannel) adds an overlay network. Each has different failure modes. Calico's BGP peering breaks with certain switch configs. Flannel's VXLAN has MTU issues. Cilium's eBPF programs can conflict with kernel versions.

**How to fix:**
- Always test with realistic node counts (not 3-node clusters)
- Set MTU explicitly — don't rely on auto-detection
- Monitor `conntrack` table fullness (`nf_conntrack_max`)
- Use `tcpdump` on the node, not in the pod, for real packet captures

**Real-world impact:** A major SaaS company lost 6 hours of production debugging "random 502s" that turned out to be conntrack table exhaustion on 3 nodes.

### 2. Service mesh adds latency you can't see in benchmarks

**What happens:** You add Istio/Linkerd for mTLS and observability. Benchmarks show "only 2ms overhead." In production, P99 latency increases 15-40ms because every hop goes through a sidecar proxy.

**Why:** Benchmarks test single-hop latency. Real requests traverse 5-10 services. Each hop adds sidecar encode → proxy → decode overhead. Under load, sidecar proxy CPU contention makes it worse.

**How to fix:**
- Measure end-to-end P99, not per-hop averages
- Use ambient mesh (Istio ambient mode) to avoid sidecars where possible
- Set proper CPU requests on sidecar containers (default is way too low)
- Consider if you actually need a service mesh — many teams don't

### 3. NetworkPolicies are not enforced by default

**What happens:** You create `NetworkPolicy` resources. They exist. They do nothing. All traffic still flows freely.

**Why:** NetworkPolicies require a CNI that supports them. The default `kubenet` on many managed K8s setups doesn't. You have to explicitly install a policy-aware CNI (Calico, Cilium).

**How to fix:**
- Verify your CNI supports NetworkPolicies before relying on them
- Test with a deny-all default policy and verify traffic is actually blocked
- Use `kubectl exec` to test connectivity between pods after applying policies

---

## Resource Limits Are Lies

### 4. CPU limits cause throttling even when the node has spare capacity

**What happens:** Your pod has `cpu: 500m` limit. The node is 30% idle. Your pod still gets throttled and responds slowly.

**Why:** CPU limits use CFS (Completely Fair Scheduler) bandwidth control. When your pod uses its 500m quota within a 100ms period, it gets throttled for the remainder — regardless of node-level CPU availability. This is by design.

**How to fix:**
- **Remove CPU limits** (yes, really) — many companies (Google, Datadog) run without CPU limits
- Keep CPU **requests** (for scheduling) but drop **limits** (for throttling)
- If you must use limits, set them 2-3x the request value
- Monitor `container_cpu_cfs_throttled_periods_total` in Prometheus

**Real-world impact:** Buffer found that removing CPU limits reduced P99 latency by 50% across their fleet.

### 5. Memory limits trigger OOMKill with misleading errors

**What happens:** Your Java/Node.js app gets OOMKilled. You set `memory: 512Mi`. The app reports using 400Mi. But it still dies.

**Why:** The memory limit counts everything in the cgroup: app heap + native memory + page cache + kernel buffers + tmpfs mounts (including secret volumes). `kubectl top` only shows RSS, not the full picture.

**How to fix:**
- Set memory limit 25-50% above what `kubectl top` shows
- For JVM apps: set `-XX:MaxRAMPercentage=75` (not 100%)
- Monitor `container_memory_working_set_bytes` (what K8s actually uses for OOM decisions)
- Check `/sys/fs/cgroup/memory/memory.stat` inside the container for the real breakdown

### 6. Requests vs limits: the scheduling trap

**What happens:** You set `requests: 100m / 128Mi` and `limits: 1000m / 1Gi`. K8s schedules your pod on a node with only 200m spare. Your pod tries to use 800m and gets throttled or killed.

**Why:** K8s scheduler only looks at **requests** for placement decisions. Limits are enforced at runtime. If every pod on a node bursts to its limit simultaneously, the node is overcommitted.

**How to fix:**
- Keep requests and limits close together for predictable behavior
- Use `LimitRange` to enforce sane ratios cluster-wide
- Monitor actual usage vs requests with Prometheus — right-size over time
- Use VPA (Vertical Pod Autoscaler) in recommendation mode to find correct values

---

## DNS Will Break You

### 7. CoreDNS is a single point of failure nobody monitors

**What happens:** Intermittent `NXDOMAIN` responses. Services can't find each other. Everything looks fine, then randomly breaks for 30 seconds.

**Why:** CoreDNS runs as a Deployment (usually 2 replicas). Under load, it can't keep up. DNS queries get dropped. The default `ndots: 5` setting means every DNS lookup generates 5 queries before trying the actual hostname.

**How to fix:**
- Scale CoreDNS with cluster-proportional-autoscaler (not just 2 replicas)
- Add `dnsConfig.ndots: 2` to your pod specs (reduces lookup amplification by 60%)
- Use `dnsConfig.searches` to minimize search domain attempts
- Enable CoreDNS cache and increase its size
- Monitor CoreDNS latency and error rates — treat it as critical infrastructure

```yaml
# Add to your pod spec to reduce DNS amplification
dnsConfig:
  ndots: "2"
  options:
    - name: single-request-reopen
```

### 8. DNS resolution for external services is surprisingly fragile

**What happens:** Your pods can reach internal services fine but randomly fail to resolve external domains (e.g., `api.stripe.com`).

**Why:** CoreDNS forwards external queries to upstream resolvers (usually the node's `/etc/resolv.conf`). If those resolvers are slow, overloaded, or rate-limited (AWS VPC DNS has a per-ENI limit), queries time out silently.

**How to fix:**
- Set explicit upstream forwarders in CoreDNS config (don't rely on node defaults)
- Use node-local DNS cache (NodeLocal DNSCache DaemonSet)
- Monitor upstream DNS latency separately from CoreDNS internal latency
- On AWS: be aware of the 1024 packets/second/ENI DNS limit

---

## Storage Is Harder Than You Think

### 9. PersistentVolumes are not portable across AZs

**What happens:** Your StatefulSet pod gets rescheduled to a different availability zone. It's stuck in `Pending` forever because the PV is zone-locked.

**Why:** EBS volumes (AWS), Persistent Disks (GCP), and Azure Disks are all AZ-specific. The PV is physically in `us-east-1a`, but the new node is in `us-east-1b`. K8s can't move the volume.

**How to fix:**
- Use `topology.kubernetes.io/zone` constraints on StatefulSets
- Consider EFS/Filestore/Azure Files for cross-AZ access (with performance tradeoffs)
- Design your app for data replication (don't rely on single PVs for HA)
- Use `volumeBindingMode: WaitForFirstConsumer` in your StorageClass

### 10. Storage class "Delete" reclaim policy will destroy your data

**What happens:** You delete a PVC (maybe accidentally, maybe via Helm uninstall). The underlying volume is immediately deleted. Your data is gone.

**Why:** Most default StorageClasses use `reclaimPolicy: Delete`. When the PVC is removed, the PV and its backing storage are destroyed. There's no recycle bin.

**How to fix:**
- Change default StorageClass to `reclaimPolicy: Retain`
- Use Velero or similar for PV backups
- Add `"helm.sh/resource-policy": keep` annotations to PVCs in Helm charts
- Implement VolumeSnapshot schedules as backup insurance

---

## Security Defaults Are Insecure

### 11. Pods run as root by default

**What happens:** Every container in your cluster runs as UID 0 (root) unless you explicitly prevent it. A container escape gives the attacker root on the node.

**Why:** Most Docker images use root. K8s doesn't override this. There's no default `SecurityContext` enforced cluster-wide. Pod Security Standards (PSS) exist but aren't enforced by default.

**How to fix:**
- Enable Pod Security Standards at namespace level (at minimum `baseline`)
- Set `runAsNonRoot: true` and `readOnlyRootFilesystem: true` in pod specs
- Use `seccompProfile: RuntimeDefault` on all workloads
- Drop all capabilities: `drop: ["ALL"]`, then add back only what's needed

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65534
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

### 12. RBAC is too permissive out of the box

**What happens:** Your developers have `cluster-admin` because it was "easier during setup." Service accounts have access to secrets across namespaces.

**Why:** Many tutorials and Helm charts create ClusterRoleBindings with excessive permissions. The default service account in every namespace can access the K8s API.

**How to fix:**
- Audit RBAC with `kubectl auth can-i --list --as=system:serviceaccount:default:default`
- Set `automountServiceAccountToken: false` on pods that don't need API access
- Create namespace-scoped Roles (not ClusterRoles) wherever possible
- Use tools like `rbac-tool` or `krane` to visualize and audit permissions

### 13. Secrets are just base64-encoded, not encrypted

**What happens:** You store database passwords in K8s Secrets thinking they're secure. Anyone with `get secrets` RBAC permission can read them in plain text.

**Why:** K8s Secrets are base64-encoded (not encrypted) at rest by default. `etcd` stores them in plain text. Anyone with API access can decode them trivially.

**How to fix:**
- Enable etcd encryption at rest (EncryptionConfiguration)
- Use external secret managers (Vault, AWS Secrets Manager, Azure Key Vault) via External Secrets Operator
- Restrict RBAC `get/list/watch` on secrets to absolute minimum
- Never log secret values — `kubectl get secret -o yaml` should be audited

---

## Upgrades Are Terrifying

### 14. Minor version upgrades break things constantly

**What happens:** You upgrade from 1.29 to 1.30. Suddenly your Ingress stops working, PodDisruptionBudgets behave differently, or deprecated APIs your Helm charts use are removed.

**Why:** K8s removes deprecated APIs on schedule (they warn, but nobody reads CHANGELOG). Beta features get promoted or removed. Each minor version changes behavior in subtle ways.

**How to fix:**
- Run `pluto` or `kubent` before every upgrade to detect deprecated APIs
- Read the [changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/) — yes, the whole thing
- Upgrade in staging first and run your full test suite
- Never skip more than one minor version
- Use `kubectl convert` to migrate manifests to newer API versions

### 15. Node upgrades can silently break your apps

**What happens:** You roll new node images. Pods get evicted and rescheduled. But some pods take 10+ minutes to become ready, causing cascading failures.

**Why:** PodDisruptionBudgets might not be set correctly. Preemption priorities aren't configured. Large images take forever to pull on fresh nodes. Init containers with external dependencies timeout.

**How to fix:**
- Set PDBs on every production workload
- Pre-pull critical images on new nodes (use DaemonSets)
- Configure pod topology spread constraints for HA during rolling updates
- Set `terminationGracePeriodSeconds` appropriately (default 30s may be too short)
- Use `maxUnavailable` and `maxSurge` in your Deployment strategy

---

## Costs Nobody Warned You About

### 16. The control plane is free — everything else is not

**What happens:** AWS EKS: control plane is $73/month. Your actual bill: $15,000/month. Where did that come from?

**The hidden costs:**

| Cost Source | Why It Hurts |
|-------------|-------------|
| NAT Gateway | $0.045/GB — all pod-to-internet traffic flows through it |
| Load Balancers | Each `Service type: LoadBalancer` creates an NLB/ALB ($16+/month each) |
| Cross-AZ traffic | $0.01/GB between AZs — adds up fast with microservices |
| EBS volumes | PVs are billed even when pods aren't running |
| CloudWatch / logging | Container logs at scale = hundreds of $/month |
| Node overhead | ~30% of each node's resources go to K8s system components |

**How to fix:**
- Use internal load balancers + one Ingress controller (not one LB per service)
- Monitor cross-AZ traffic — colocate chatty services in same AZ
- Use spot/preemptible instances for non-critical workloads (save 60-80%)
- Set log retention policies aggressively
- Right-size nodes — fewer large nodes > many small nodes (less overhead)

### 17. You need 3 engineers minimum to run K8s properly

**What happens:** Your team of 2 sets up K8s. It works. Then someone goes on vacation. An upgrade is needed. A node fails at 2 AM. Nobody knows how to debug it.

**Why:** K8s has an enormous operational surface area: networking, storage, security, upgrades, monitoring, RBAC, cert rotation, etcd maintenance. No single person can master all of it.

**Realistic team requirements:**

| Cluster Scale | Minimum Team | Responsibilities |
|--------------|-------------|-----------------|
| < 10 services | 1-2 engineers | Basic ops, but use managed K8s |
| 10-50 services | 2-3 engineers | + monitoring, security, upgrade planning |
| 50+ services | 3-5 engineers | Full platform team |
| 200+ services | Dedicated platform org | Internal developer platform |

---

## Observability Is Not Optional

### 18. `kubectl top` is almost useless

**What happens:** You run `kubectl top pods` to debug a performance issue. It shows CPU and memory numbers that don't match your monitoring system and lag by 30-60 seconds.

**Why:** Metrics Server (which `kubectl top` uses) is a point-in-time snapshot with 30s+ delay. It doesn't show historical data, trends, throttling, or disk/network metrics.

**What you actually need:**
- **Prometheus + Grafana** for metrics (or Datadog/New Relic if budget allows)
- **Loki or OpenSearch** for logs (not just `kubectl logs`)
- **Jaeger or Tempo** for distributed tracing
- **AlertManager** with PagerDuty/OpsGenie integration
- Custom dashboards per team, not one generic cluster dashboard

### 19. Logs disappear when pods restart

**What happens:** Your pod crashes. You run `kubectl logs <pod>`. It shows logs from the new container instance, not the one that crashed.

**Why:** Container logs are stored on the node's filesystem. When a container restarts, the previous logs are only available with `--previous` flag. When the pod is rescheduled to a different node, they're gone forever.

**How to fix:**
- Ship logs to a central system (Loki, ELK, CloudWatch) before they're lost
- Use `kubectl logs --previous` to see the last container's logs
- Set `terminationMessagePolicy: FallbackToLogsOnError` to capture crash context
- Implement structured logging (JSON) for searchability

---

## YAML Hell Is Real

### 20. Helm charts are black boxes of complexity

**What happens:** You `helm install` a popular chart. It creates 47 resources you didn't know about — ServiceAccounts, ClusterRoles, ConfigMaps, PDBs, NetworkPolicies. Something breaks and you have no idea what half of these resources do.

**Why:** Helm charts are designed to be "complete." They include every possible feature, most enabled by default. The `values.yaml` file is 500+ lines with nested conditionals.

**How to fix:**
- Always run `helm template` before `helm install` to see what you're deploying
- Read the chart source code, not just the README
- Override `values.yaml` explicitly — don't rely on defaults
- Consider Kustomize for simpler overlay-based customization
- Pin chart versions — `helm upgrade` with unpinned versions is a gamble

### 21. The manifest sprawl nobody talks about

**What happens:** You start with 5 YAML files. Six months later you have 200+. Nobody knows which ones are active. Some reference resources that no longer exist.

**Why:** K8s is declarative, but there's no built-in way to track what manifests belong together, which are outdated, or which have drifted from source control.

**How to fix:**
- Use GitOps (ArgoCD/Flux) — the repo is the single source of truth
- Label everything consistently (team, app, environment, version)
- Run periodic drift detection between cluster state and Git
- Use `kubectl diff` before applying changes
- Implement resource quotas to prevent namespace sprawl

---

## The Human Cost

### 22. On-call for K8s clusters is 10x harder than for VMs

**What happens:** 3 AM alert: "Pod CrashLoopBackOff." Could be: app bug, OOM, missing secret, node failure, storage full, DNS issue, network policy blocking traffic, image pull error, or cert expiration. Good luck figuring out which one in your pajamas.

**Why:** K8s adds layers of abstraction. Each layer can fail independently. The error messages are often generic and don't point to root cause. You need to understand networking, storage, scheduling, and the application to debug effectively.

**Survival guide:**
- Build runbooks for the 10 most common alerts (CrashLoopBackOff, OOMKilled, Pending, ImagePullBackOff, etc.)
- Create Grafana dashboards that answer "is it the cluster or the app?"
- Implement structured error handling in your apps (crash with meaningful exit codes)
- Use `kubectl describe` → `kubectl logs` → `kubectl events` as your debug flow

### 23. The learning curve never ends

**What happens:** You spend 6 months learning K8s basics. Then you discover you also need to learn: Helm, ArgoCD, Prometheus, Istio, Cert-Manager, External-DNS, KEDA, Karpenter, OPA/Gatekeeper, Velero, Crossplane...

**Why:** K8s is a platform for building platforms. The core is intentionally minimal. Every real-world need requires an addon, operator, or custom controller. The CNCF landscape has 1000+ projects.

**The minimum viable addon stack:**

| Category | You'll Need | Alternative |
|----------|------------|-------------|
| Ingress | Nginx Ingress / Traefik | Cloud ALB controller |
| Certs | cert-manager | Manual or cloud-managed |
| Monitoring | Prometheus + Grafana | Datadog / New Relic |
| Logging | Loki / OpenSearch | CloudWatch / Stackdriver |
| GitOps | ArgoCD / Flux | Helm + CI/CD pipeline |
| Autoscaling | KEDA / Karpenter | HPA + Cluster Autoscaler |
| Secrets | External Secrets Operator | Sealed Secrets |
| Policy | OPA Gatekeeper / Kyverno | Pod Security Standards |

---

## Multi-Tenancy Is a Fantasy

### 24. Namespace isolation is not real isolation

**What happens:** Team A's runaway pod starves Team B's pods of CPU on the same node. A noisy neighbor in one namespace affects all namespaces on the node.

**Why:** Namespaces are logical boundaries, not security boundaries. Pods from different namespaces share the same nodes, network, and kernel. ResourceQuotas limit total consumption but don't prevent burst contention.

**How to fix:**
- Use node affinity / node pools to physically separate tenants
- Enforce ResourceQuotas AND LimitRanges per namespace
- Implement NetworkPolicies to isolate namespace traffic
- For hard multi-tenancy, use virtual clusters (vCluster) or separate clusters

### 25. RBAC multi-tenancy requires more effort than anyone admits

**What happens:** You set up namespaces per team. Then you realize teams need cross-namespace access for shared services, monitoring, and debugging. RBAC rules multiply exponentially.

**Why:** K8s RBAC was designed for operator/developer separation, not complex multi-team isolation. Every cross-cutting concern (shared databases, monitoring, logging) needs explicit RBAC rules.

**The reality:**

| What You Want | What You Get |
|--------------|-------------|
| Isolated namespaces | 50+ RBAC rules per namespace |
| Shared services access | ClusterRoles that partially break isolation |
| Self-service namespace creation | Custom controllers or Capsule/HNC |
| Per-team cost visibility | Additional tools (kubecost, OpenCost) |

---

## Managed K8s Still Hurts

### 26. "Managed" doesn't mean "managed"

**What happens:** You choose EKS/GKE/AKS thinking the cloud provider handles everything. You still manage: node groups, networking, Ingress, monitoring, RBAC, upgrades, storage classes, DNS, certs, and every addon.

**What each provider actually manages:**

| Responsibility | EKS | GKE | AKS |
|---------------|-----|-----|-----|
| Control plane | ✅ | ✅ | ✅ |
| Control plane upgrades | Manual trigger | Auto (standard) | Manual trigger |
| Node upgrades | Manual / managed node groups | Auto (standard) | Manual / auto |
| Networking CNI | VPC CNI (default) | GKE networking | Azure CNI |
| Ingress | You manage | GKE Ingress / Gateway | You manage |
| Monitoring | You manage | Cloud Monitoring (basic) | You manage |
| RBAC | You manage | You manage | You manage |
| Storage classes | You manage | Preconfigured | Preconfigured |

### 27. Vendor lock-in is deeper than you think

**What happens:** You build on GKE with Config Connector, GKE Ingress, Workload Identity, and Cloud SQL Auth Proxy. "It's still K8s, we can migrate anytime." No, you can't.

**Lock-in points:**
- **IAM integration** — AWS IRSA, GKE Workload Identity, AKS Pod Identity are all different
- **Load balancers** — Each cloud's annotations and controller are unique
- **Storage** — CSI drivers and StorageClasses are provider-specific
- **Networking** — VPC CNI (AWS), GKE Dataplane V2, Azure CNI Overlay are incompatible
- **Addons** — Managed Prometheus (GKE), Container Insights (EKS), are proprietary

**How to minimize:**
- Use cloud-agnostic tools where possible (Terraform, ArgoCD, Prometheus)
- Abstract cloud-specific configs into Helm values files
- Maintain a "portability layer" document listing all cloud-specific dependencies

---

## The Bottom Line

### Should you still use Kubernetes?

**Yes, if:**
- You have 10+ microservices that need independent scaling
- You have a team of 3+ engineers who can dedicate time to platform work
- You need multi-cloud or hybrid deployment capabilities
- Your scale justifies the operational complexity

**No, if:**
- You have < 5 services (use ECS, Cloud Run, or Azure Container Apps)
- Your team is < 3 engineers (use a PaaS)
- You're doing it because "everyone else does" (they're also struggling)
- You don't have monitoring, logging, and alerting figured out first

> **The best K8s cluster is the one you don't need to run.**

---

## Related Repos

| Repo | Description |
|------|-------------|
| [docker-vs-podman](https://github.com/atryx/docker-vs-podman) | Container runtime comparison |
| [docker-cheatsheet](https://github.com/atryx/docker-cheatsheet) | Docker commands quick reference |
| [devops-learning-path](https://github.com/atryx/devops-learning-path) | Full DevOps roadmap including K8s |
| [production-ready-snippets](https://github.com/atryx/production-ready-snippets) | K8s manifests, Docker Compose, Nginx configs |
| [awesome-dev-errors](https://github.com/atryx/awesome-dev-errors) | Common K8s and Docker error fixes |
| [terraform-gotchas](https://github.com/atryx/terraform-gotchas) | Terraform pitfalls for infrastructure |
| [aws-hidden-costs](https://github.com/atryx/aws-hidden-costs) | AWS billing surprises |

---

## Contributing

Found a gotcha we missed? Have a war story? [See CONTRIBUTING.md](CONTRIBUTING.md) to add your own.

---

## License

MIT — Share freely, attribute kindly.

⭐ **If this saved you from a production incident, star this repo.**
