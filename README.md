Here is a comprehensive technical writeup summarizing your entire engineering milestone journey across cluster configuration, security auditing, and cloud-native resilience engineering, followed by the mandatory updates needed for your repository's `README.md` file.

---

### Part 1: Architecture & Engineering Journey Writeup

#### 1. Cluster Foundation (Single-Node Dev-Prod Simulation)

The infrastructure foundation consists of bootstrapping a highly optimized single-node Kubernetes master and worker co-located cluster running **Kubernetes v1.36** on an **Ubuntu 24.04 Server** host equipped with 16GB RAM and a 4-core CPU.

* **Core Runtime:** Switched entirely away from Docker to the industry-standard native **containerd** runtime interface.
* **Control Plane Un-Tainting:** To achieve full single-node convergence, the control-plane node's native `NoSchedule` taint was cleanly scrubbed, permitting active developer workloads to schedule adjacent to the cluster's core management controllers.
* **Networking plugin:** Configured **Calico CNI** to handle standard intra-cluster pod networking, establishing the baseline framework for standard container routing.

#### 2. Advanced Security Hardening: API Auditing & SIEM Readiness

To achieve strict security compliance and centralized SIEM visibility, an advanced **Kubernetes API Server Audit Policy** was built from scratch.

* **Granular Policy Design:** The policy systematically isolates security events across three pillars:
1. *Administrative Actions:* Capturing high-risk runtime operations (`kubectl exec`, `kubectl attach`, `kubectl port-forward`, `kubectl cp`).
2. *Sensitive Object Access:* Capturing read/write/patch states on `Secrets` and `ConfigMaps`.
3. *RBAC Control Plain Changes:* Continuous monitoring of mutations to `Roles`, `ClusterRoles`, `RoleBindings`, `ClusterRoleBindings`, and `ServiceAccounts`.


* **Kubelet Control Integration:** The policy file was mounted into the static `kube-apiserver` manifest under `/etc/kubernetes/audit-policy.yaml`, modifying the API stream arguments (`--audit-policy-file`, `--audit-log-path`, `--audit-log-maxsize`) and verifying runtime socket updates dynamically via container runtime flags using `crictl`.

#### 3. Pure GitOps Chaos Engineering Pipeline

Rather than using a visual dashboard ("the manager's way"), you constructed a **Declarative SRE / GitOps Chaos Pipeline** powered by **LitmusChaos**. You successfully targeted application namespaces and safely iterated through two major automated chaos types to validate architecture self-healing:

* **Experiment 1: Network Loss/Latency Injection (`pod-network-loss`)**
* *Mechanics:* A dedicated Chaos Runner orchestrated a highly privileged helper pod. This worker directly intercepted the containerd execution namespace of the target application container using advanced container security flags.
* *Execution:* Using Linux traffic control (`tc qdisc`) and `iptables` rules, it forced a complete network drop-block inside the container's network namespace for exactly 15 seconds before cleanly auto-healing and tearing down the interception rules.


* **Experiment 2: Memory Resources Stress Test (`node-memory-hog`)**
* *Mechanics:* Designed, troubleshot, and executed an automated memory pod-hogging experiment.
* *Validation:* Navigated OOM (Out-of-Memory) hazards, safely tuned memory limits, and systematically iterated configurations until achieving a flawless End-Of-Test (EOT) metric evaluation.
* *Final Result Scorecard:* Achieved an official **Verdict: Pass** with a **Probe Success Percentage of 100%**.



---

### Part 2: Mandatory Changes Required for `README.md`

To accurately represent this advanced infrastructure and engineering state, your project repository's `README.md` needs to serve as explicit "Operations Runbook" rather than a basic greeting file. Ensure the following mandatory sections are added or revised:

#### 1. Mandatory Prerequisites & Node Setup

Document the hard environmental settings required to prevent the cluster configuration from collapsing upon restart.

```markdown
## 📋 Cluster Prerequisites & Bootstrap

Before initializing the cluster, the host must be prepared exactly as follows:
- **OS:** Ubuntu 24.04 Server
- **Runtime:** Containerd (`/var/run/containerd/containerd.sock`)
- **Swap Space:** Permanently disabled (`swapoff -a` and commented out in `/etc/fstab`).
- **Control Plane Taint Removal:** ```bash
  kubectl taint nodes --all node-role.kubernetes.io/control-plane-

```

```

#### 2. Security Configuration & API Audit Policy Specifications
Explicitly document the exact locations and parameters used for the API Server audit logging so any contributor or automated runner knows how the security baseline is configured.
```markdown
## 🔒 Security Compliance: API Server Auditing

Centralized SIEM auditing is enabled via an explicit policy file.

### Required File Layout
- Audit Policy Path: `/etc/kubernetes/audit-policy.yaml`
- Active Audit Log Stream: `/var/log/kubernetes/audit.log`

### Mandatory KubeServer Arguments (`/etc/kubernetes/manifests/kube-apiserver.yaml`)
Your static manifest **must** define these flags and corresponding `hostPath` volume mounts:
- `--audit-policy-file=/etc/kubernetes/audit-policy.yaml`
- `--audit-log-path=/var/log/kubernetes/audit.log`
- `--audit-log-maxsize=100`

### Validation Verification Checklist
If the audit log stream goes missing, check process binding logs via:
```bash
sudo crictl ps | grep kube-apiserver
sudo crictl logs <container_id>

```

```

#### 3. Strict RBAC Rules for Chaos Automation
Document the granular RBAC configuration so that standard pods are not left blind or dangerously over-privileged.
```markdown
## 💥 Chaos Engineering Operations (GitOps Flow)

This repository executes Automated Chaos Engineering via **LitmusChaos CRDs** using pure GitOps declarations.

### Critical RBAC Constraints
- **Same-Namespace Injection:** Standard pods have zero ambient API privileges. The Chaos Runner requires a dedicated `ServiceAccount` explicitly bound to a `Role` with permissions to watch, get, and patch `pods`, `chaosengines`, and `chaosresults`.
- **Cross-Namespace Injection:** Cross-namespace testing requires a wide-scoped `ClusterRoleBinding` or dedicated target namespace `RoleBindings` to allow the helper pod to break boundaries and manipulate target containerd runtimes.

```

#### 4. Chaos Validation Matrix (The Scorecard)

Include a formal execution ledger showcasing the verified testing criteria and the successful benchmarks you've achieved.

```markdown
### 📊 Verified Resiliency Scorecard

| Experiment Type | Scope | Target Action | Target Duration | Expected Result | Official Verdict |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `pod-network-loss` | App Container | 100% Packet Drop via `tc` / `iptables` | 15 Seconds | Auto-recovers traffic | **Pass (100%)** |
| `node-memory-hog` | Cluster Memory | Resource Consumption Stress Test | 60 Seconds | Endures without crash loops | **Pass (100%)** |

To inspect the real-time SRE compliance metric block, always verify the status via:
```bash
kubectl get chaosresult <experiment-name> -o yaml
# Look specifically under status.experimentStatus.verdict and probeSuccessPercentage

```

```

```
