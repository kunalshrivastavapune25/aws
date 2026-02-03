Here are structured, professional sample answers for your Kubernetes (K8s) interview questions. These are based on real-world best practices and common expectations in 2025–2026 interviews for platform/DevOps/SRE roles. Tailor them with your personal experiences where possible.

### 1. Disaster Management / Recovery in Kubernetes (Common question: "How do you handle disaster recovery in Kubernetes?")

**Sample Answer:**

In Kubernetes, disaster recovery (DR) focuses on minimizing **RPO** (Recovery Point Objective – data loss) and **RTO** (Recovery Time Objective – downtime).

Key components I prioritize:

- **Control plane (etcd) backups** — Regular automated snapshots of etcd (the cluster's single source of truth) using kubeadm-builtin tools, Velero, or operators like etcd-backup-restore. Store them off-cluster (e.g., S3, GCS) with encryption and versioning. Restore involves recreating the cluster and restoring etcd.

- **Application-level backups** — Use tools like **Velero** (formerly Heptio Ark) or Trilio for application-consistent backups of PVs, PVCs, deployments, configmaps, secrets, etc. It supports hooks for pre/post backup actions (e.g., quiesce DB).

- **High Availability setup** — Run etcd in HA mode (3+ nodes), multiple control plane nodes across AZs. Use multi-region or multi-cluster DR (active-passive or active-active with global load balancing).

- **Storage & Data** — For stateful apps, use CSI drivers with snapshots (e.g., Portworx, Longhorn, or cloud-native like EBS volumes). Aim for near-zero RPO with synchronous replication for critical apps (metro region) or asynchronous for others.

- **Best practices** — 
  - Automate everything (GitOps for manifests).
  - Regularly test restores in a staging/drill environment (simulate failures).
  - Immutable infrastructure + multi-region deployments.
  - Secure backups (immutable storage to protect against ransomware).

In one project, we used Velero + Restic for PV backups and achieved <15 min RTO for most services after quarterly DR drills.

### 2. Daily Challenges You Face in Kubernetes

**Sample Answer:**

Kubernetes is powerful but operationally intensive. Some common daily/ongoing challenges I've faced (and how I handle them):

- **Resource mismanagement / over-provisioning** — Many workloads request far more CPU/memory than needed (often 65%+ underutilized), leading to waste and scheduling issues. I use VPA (Vertical Pod Autoscaler) in recommendation mode, Prometheus + Grafana dashboards, and tools like Goldilocks or Kubecost to right-size requests/limits.

- **Troubleshooting pod/startup issues** — CrashLoopBackOff, ImagePullBackOff, OOMKilled, or pending pods due to taints/node pressure. Daily use `kubectl describe`, logs, events, and tools like Komodor or Lens for faster root cause (79% of outages from recent changes).

- **Configuration drift & change management** — Drift from GitOps desired state. Mitigate with ArgoCD/Flux + policy enforcement (Kyverno/OPA Gatekeeper).

- **Observability gaps** — Hard to correlate app + infra metrics. Use Prometheus, Grafana, Loki for logs, and eBPF tools (Pixie) for deep insights without agents.

- **Security & compliance firefighting** — Misconfigurations (root containers, exposed ports). Enforce Pod Security Standards, RBAC least privilege, network policies, and image scanning (Trivy).

- **Scaling & cost control** — Unexpected spikes or idle resources. Cluster autoscaler + HPA + Karpenter for dynamic scaling, and spot/preemptible nodes for cost savings.

Most days involve 60% troubleshooting/reactive work and 40% proactive optimization. The key is shifting left with good CI/CD, GitOps, and observability.

### 3. Performance Improvements in Kubernetes

**Sample Answer:**

Performance tuning in K8s spans cluster, workload, and app levels. My approach:

- **Resource optimization** — Always set realistic requests/limits (avoid OOM/evictions). Use HPA for horizontal scaling and VPA for vertical. Right-size nodes (avoid tiny nodes causing scheduling overhead).

- **Scheduler tuning** — For large clusters (>1000 nodes), adjust `percentageOfNodesToScore` to balance speed vs accuracy.

- **Networking** — Choose efficient CNI (Calico, Cilium with eBPF for better perf). Use service mesh (Istio/Linkerd) only when needed (it adds latency). Optimize DNS (CoreDNS tuning).

- **Storage** — Use local SSDs or high-perf CSI for latency-sensitive workloads. Avoid NFS for high IOPS.

- **Workload best practices** — Lightweight images (Alpine/distroless), multi-stage builds. Use readiness/liveness probes correctly. Avoid anti-affinity unless necessary (it fragments pods).

- **Monitoring-driven** — Prometheus alerts on high CPU steal, network latency, pod restarts. Use cluster-profiling tools.

In one case, we reduced latency by 40% by switching to Cilium + tuning HPA + right-sizing (from over-provisioned to 80% utilization).

### 4. Planning to Migrate 20 Legacy Applications to Kubernetes

**Sample Answer:**

Migrating 20 legacy (often monolithic/stateful) apps requires a phased, low-risk approach — not big bang.

**High-level strategy** (Strangler Fig pattern + iterative):

1. **Assessment phase (2–4 weeks)**  
   - Inventory: Catalog all 20 apps (tech stack, dependencies, stateful/stateless, traffic, SLAs).  
   - Prioritize: Start with stateless, low-complexity, non-critical (quick wins). Group by similarity (e.g., Java Spring apps).  
   - Identify pain points: DB connections, file storage, sessions, configs.

2. **Preparation & Platform setup**  
   - Build golden Kubernetes platform (managed EKS/AKS/GKE or self-managed).  
   - Set up GitOps (ArgoCD/Flux), observability (Prometheus/Grafana), security (RBAC, policies), CI/CD.  
   - Provide developer templates (Helm charts/operators).

3. **Migration patterns (per app, iterative 1–3 months each)**  
   - **Lift & Shift** — Containerize as-is (Dockerfile), run as Deployment/StatefulSet. Use ConfigMaps/Secrets.  
   - **Refactor lightly** — Externalize configs, use service discovery.  
   - **Strangle** — For monoliths, extract microservices gradually (sidecar proxies if needed).  
   - Handle state: PVCs + CSI snapshots, or migrate DB to cloud-managed (RDS → Aurora).

4. **Execution & Testing**  
   - Parallel run: Blue-green or canary via Istio/ingress.  
   - Testing: Functional, load, chaos (Litmus/Chaos Mesh).  
   - Cutover: Gradual traffic shift, rollback plan.

5. **Post-migration**  
   - Decommission old infra.  
   - Optimize: Autoscaling, cost monitoring.  
   - Knowledge transfer & self-service.

For 20 apps, aim for 4–6 waves (group similar ones). Risks mitigated via pilots, rollback, and DR testing.

Here are concise, interview-ready sample answers for your new Kubernetes questions. These reflect real-world practices (as of 2026), focusing on clarity, tools, and reasoning that interviewers value. Tailor with your experiences.

### 1. How do you find unused and not necessarily pods and remove them?  
(Interpreting as: How to find and clean up non-running / terminated / evicted / failed / unused pods?)

**Sample Answer:**

Kubernetes doesn't immediately delete all terminated pods — it keeps Failed, Succeeded, Evicted, or Completed ones for debugging (logs/events). The **terminated-pod-gc-threshold** (default 12,500) triggers the garbage collector when exceeded, but we shouldn't wait for that in production.

**Steps to identify and clean:**

- List non-running pods:  
  `kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Pending`

  Or specifically:  
  - Evicted/Failed: `kubectl get pods --all-namespaces | grep Evicted` or `--field-selector=status.phase=Failed`  
  - Completed/Succeeded (e.g., Jobs): `kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded`

- Bulk delete (safest one-liner for evicted):  
  `kubectl delete pods --all-namespaces --field-selector=status.phase=Evicted`  
  Or script for all terminated:  
  ```bash
  kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Pending -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}' | xargs -n2 kubectl delete pod -n
  ```

- For "unused" pods (rare, as standalone pods without controllers are unusual): If pods lack ownerReferences (not managed by Deployment/ReplicaSet/Job), use tools like **Kor** (`kor all --namespace all`) or **k8s-cleaner** controller to detect orphaned/unused pods.

**Best practices / prevention:**
- Use **TTLAfterFinished** on Jobs (`ttlSecondsAfterFinished: 300`) for auto-cleanup.
- Monitor with Prometheus alerts on `kube_pod_status_phase{phase=~"Failed|Evicted|Succeeded"} > 0`.
- In production, cronjob/script weekly cleanup + force delete stuck Terminating pods (`--grace-period=0 --force` — but cautiously).

This keeps etcd clean, reduces API server load, and avoids quota issues.

### 2. How do you approach a new solution?  
(Assuming: How do you approach designing/implementing a new solution/architecture in Kubernetes?)

**Sample Answer:**

My structured approach for a new Kubernetes-based solution (platform feature, workload, or migration):

1. **Understand requirements deeply** (1–2 days)  
   - Functional + non-functional (scale, HA, latency, cost, security, observability).  
   - Workload type: stateless vs stateful, batch vs long-running, traffic pattern.  
   - Constraints: team skills, existing stack, compliance.

2. **Choose primitives wisely** (not everything needs CRDs/operators)  
   - Stateless → Deployment + HPA + Service.  
   - Stateful → StatefulSet + PVC + headless Service (or operator if complex like DB).  
   - Batch → Job/CronJob with TTL.  
   - Advanced → Custom resources only if needed (e.g., via Operator SDK).

3. **Design for Day 2 operations first**  
   - GitOps from day 1 (ArgoCD/Flux).  
   - Observability: Prometheus rules, Grafana dashboards, Loki/Pixie.  
   - Security: PodSecurityStandards, NetworkPolicies, RBAC least-privilege.  
   - Cost: Requests/limits tuned, Karpenter/Cluster Autoscaler.

4. **Prototype & validate** (quick PoC in dev namespace)  
   - Helm chart or Kustomize for templating.  
   - Chaos testing early (Litmus/Chaos Mesh).  
   - Load test with Locust/k6.

5. **Iterate & productionize**  
   - Canary/blue-green rollout (Argo Rollouts or Flagger).  
   - Policy enforcement (Kyverno/OPA).  
   - Documentation + self-service templates for devs.

This "shift-left" approach avoids rework — I've used it to launch 10+ services with zero major incidents in prod.

### 3. When should we think of migrating EKS to ECS (which is fully managed)?

**Sample Answer:**

EKS (Kubernetes) and ECS (AWS-proprietary) serve similar goals, but ECS is simpler/fully managed with less control. Migration from EKS → ECS is a "simplification" move — consider it when Kubernetes overhead outweighs benefits.

**Key scenarios to think about migrating EKS to ECS:**

- **Small/medium team size** (<10–15 DevOps/platform engineers) struggling with Day 2 ops: control-plane upgrades, etcd management, CNI tuning, kubelet issues, frequent CVEs in core components.
- **Operational burden too high**: Spending >30–40% time on Kubernetes itself (node patching, version upgrades, debugging scheduler/evictions) instead of app delivery.
- **Workloads are AWS-native & steady-state**: Simple containerized services, no heavy use of Helm charts, operators, CRDs, or multi-cloud portability needs. ECS integrates tightly with ALB, CloudWatch, IAM roles for tasks, Fargate.
- **Cost + simplicity priority over features**: ECS on Fargate is often cheaper at low–medium scale (no node management), easier monitoring, faster onboarding. EKS has ~$72/month cluster fee + node costs.
- **No Kubernetes ecosystem lock-in**: Not relying on Istio/Linkerd, ArgoCD, Prometheus operators, or community tools that don't exist in ECS.

**When NOT to migrate (stick with EKS):**
- Need multi-cloud/portability.
- Heavy microservices with service mesh, advanced autoscaling, custom controllers.
- Already invested in K8s skills/GitOps — migration cost (rewrite manifests → task defs, lose Helm/operators) often 4–12 weeks.
- High-scale/complex workloads where EKS (with Karpenter, Cilium, Auto Mode) outperforms.

In 2025–2026, many teams migrate EKS → ECS for startups/mid-size after "Kubernetes fatigue," but it's rare the other way unless standardizing on open-source K8s.

Practice these with confidence — add metrics or examples from your work. If you share more context (e.g., team size or workload type), I can refine further. All the best! 🚀 