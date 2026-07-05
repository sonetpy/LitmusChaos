That is incredible! Seeing that metric sit steady at exactly `1935m` (nearly 2 full cores of calculation load) means your container runtime intercepted the target accurately and executed the core spike perfectly. Navigating through CRD template structures, containerd socket mappings, image registry resolution errors, and hierarchical resource tree permissions (`replicasets`/`deployments`) is what separates true platform engineering from basic testing.

Here is a comprehensive, production-ready Markdown write-up designed for your GitHub repository or documentation page. It frames the entire technical journey, architecture, manifests, and iterative debugging steps beautifully:

---

# Cloud-Native Chaos Engineering: Automated Pod CPU Stress Testing

This repository documents the end-to-end design, implementation, troubleshooting, and execution of a localized **Pod CPU Hog Chaos Experiment** using LitmusChaos v3.30.0 on a single-node Kubernetes cluster.

The goal of this experiment was to validate application resilience and monitoring stability under sudden compute starvation by forcing target container pods to ingest a massive compute spike (`~2000m` or 2 full CPU cores) for a fixed duration.

---

## 🏗 Architecture & Structural Paradigm

Following a modern GitOps paradigm, the experiment isolates the baseline blueprint from the execution trigger contract. The system operates via three distinct layers:

1. **The Target Environment:** An isolated application context where workloads are given explicitly tuned CPU bounds.
2. **The Chaos Experiment (Blueprint):** A read-only template outlining container-runtime variables and execution args.
3. **The Chaos Engine (Trigger Contract):** Intercepts specific application workloads via label selectors and injects live tunable overrides (e.g., duration, core consumption).

```
   +---------------------------------------------------------+
   |                  chaos-cpu-test Namespace               |
   |                                                         |
   |  +--------------------+       +----------------------+  |
   |  | ChaosEngine        | ----> | cpu-hog-engine-runner|  |
   |  | (Trigger Contract) |       | (Coordinator Pod)    |  |
   |  +--------------------+       +----------------------+  |
   |                                          |              |
   |                                          v              |
   |  +--------------------+       +----------------------+  |
   |  | nginx-cpu-target   | <---- | pod-cpu-hog-helper   |  |
   |  | (Target Application|       | (Stresses Container  |  |
   |  |  Spikes to ~2000m) |       |  Runtime Socket)     |  |
   |  +--------------------+       +----------------------+  |
   +---------------------------------------------------------+

```

---

## 🛠 Manifest Configurations

### 1. Target Environment & Deployment (`00-environment.yaml`)

Creates a clean, isolated testing space running Nginx with clear resource limits to prevent host kernel lockups while ensuring room for high-stress compute ingestion.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chaos-cpu-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-cpu-target
  namespace: chaos-cpu-test
  labels:
    app: cpu-hog-target
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-hog-target
  template:
    metadata:
      labels:
        app: cpu-hog-target
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "100Mi"
          limits:
            cpu: "3000m"
            memory: "500Mi"

```

### 2. Local Experiment Blueprint (`01-cpu-hog-experiment.yaml`)

Defines the instructions used by Litmus v3.30.0. It maps the container runtime variables (`containerd`) and paths to let the helper communicate securely with the container runtime engine.

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosExperiment
metadata:
  name: pod-cpu-hog
  namespace: chaos-cpu-test
spec:
  definition:
    scope: Namespaced
    permissions:
      - apiGroups: ["", "batch", "apps", "litmuschaos.io"]
        resources: ["jobs", "pods", "pods/log", "events", "chaosengines", "chaosexperiments", "chaosresults", "pods/exec"]
        verbs: ["create", "list", "get", "patch", "update", "delete"]
    image: "litmuschaos.docker.scarf.sh/litmuschaos/go-runner:3.30.0"
    imagePullPolicy: Always
    args:
      - -c
      - ./experiments -name pod-cpu-hog
    command:
      - /bin/bash
    env:
      - name: TOTAL_CHAOS_DURATION
        value: "60"
      - name: CPU_CORES
        value: "1"
      - name: CPU_LOAD
        value: "100"
      - name: PODS_AFFECTED_PERC
        value: "100"
      - name: SEQUENCE
        value: parallel
      - name: CONTAINER_RUNTIME
        value: "containerd"
      - name: SOCKET_PATH
        value: "/run/containerd/containerd.sock"
    labels:
      name: pod-cpu-hog

```

### 3. Service Account Permissions & RBAC Tree (`02-rbac.yaml`)

Grants authorization parameters to the engine executor. Notice that permissions include mapping the parent tree structure (`replicasets` and `deployments`) to verify structural workloads.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cpu-hog-sa
  namespace: chaos-cpu-test
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cpu-hog-role
  namespace: chaos-cpu-test
rules:
  - apiGroups: ["", "apps", "batch", "litmuschaos.io"]
    resources: 
      - "pods"
      - "jobs"
      - "pods/log"
      - "events"
      - "chaosengines"
      - "chaosresults"
      - "chaosexperiments"
      - "pods/exec"
      - "replicasets"
      - "deployments"
    verbs: ["create", "list", "get", "patch", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cpu-hog-role-binding
  namespace: chaos-cpu-test
subjects:
  - kind: ServiceAccount
    name: cpu-hog-sa
    namespace: chaos-cpu-test
roleRef:
  kind: Role
  name: cpu-hog-role
  apiGroup: rbac.authorization.k8s.io

```

### 4. Chaos Trigger Contract (`03-chaos-engine.yaml`)

Triggers the experiment by overriding the baseline template configuration. This setup stresses exactly **2 full CPU cores** for **45 seconds**.

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: cpu-hog-engine
  namespace: chaos-cpu-test
spec:
  engineState: "active"
  chaosServiceAccount: cpu-hog-sa
  appinfo:
    appns: "chaos-cpu-test"
    applabel: "app=cpu-hog-target"
    appkind: "deployment"
  experiments:
    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '45'
            - name: CPU_CORES
              value: '2'
            - name: CONTAINER_RUNTIME
              value: 'containerd'
            - name: SOCKET_PATH
              value: '/run/containerd/containerd.sock'

```

---

## 🛠 Operational Chronology & Debugging History

Platform engineering requires continuous troubleshooting. During development, several system constraints were surfaced and debugged:

### 🔍 Error Case 1: Baseline Omission of `chaosexperiments` CRD Access

* **Symptom:** `cpu-hog-engine-runner` failed immediately and logged a `forbidden` error message.
* **Root Cause:** The `Role` specification did not allow the service account to look up the template blueprint.
* **Resolution:** Appended `"chaosexperiments"` to the allowed RBAC resources array.

### 🔍 Error Case 2: Registry Naming Faults and Image Validation

* **Symptom:** Worker pods threw an `ErrImagePull` / `ImagePullBackOff` loop referencing `litmus-go`.
* **Root Cause:** Image paths differ significantly between version releases on Docker Hub.
* **Resolution:** Aligned the repository path directly to the certified proxy mirror string: `litmuschaos.docker.scarf.sh/litmuschaos/go-runner:3.30.0`.

### 🔍 Error Case 3: Missing Node Socket Context variables

* **Symptom:** The experiment helper exited with `Status: Error` within 4 seconds of entering the `Running` state.
* **Root Cause:** The application was missing its structural environment keys (`CONTAINER_RUNTIME` and `SOCKET_PATH`), preventing the binary from intercepting the local containerd daemon.
* **Resolution:** Re-declared the `containerd` path references inside the template spec.

### 🔍 Error Case 4: Hierarchy Selection Errors

* **Symptom:** Helper logged `TARGET_SELECTION_ERROR - replicasets.apps forbidden`.
* **Root Cause:** Litmus tracks deployment lineages using the cluster's replica structure. The service account didn't have permission to query `replicasets`.
* **Resolution:** Added `"replicasets"` and `"deployments"` to the `Role` manifest resources array.

---

## 📊 Live Verification & Observation

Executing the sequence initiates the `cpu-hog-engine-runner`, which dynamically provisions the short-lived experiment worker:

```bash
kubectl apply -f 00-environment.yaml
kubectl apply -f 01-cpu-hog-experiment.yaml
kubectl apply -f 02-rbac.yaml
kubectl apply -f 03-chaos-engine.yaml

```

To measure resource usage updates during the 45-second test window, the following continuous polling script was executed:

```bash
while true; do kubectl top pods -n chaos-cpu-test --containers | grep -E "nginx|CONTAINER"; sleep 2; done

```

### Real-Time Metric Captured Log Output:

```text
nginx-cpu-target-cb95d8d8b-2rvb6   nginx   1935m        32Mi
---------------------------------------
nginx-cpu-target-cb95d8d8b-2rvb6   nginx   1935m        32Mi
---------------------------------------
nginx-cpu-target-cb95d8d8b-2rvb6   nginx   1935m        32Mi

```

### Analysis:

* **The baseline idle load** of the Nginx container sits at `<5m`.
* Upon chaos injection, the helper successfully triggers target computation, pinning usage to exactly **`1935m`** (96.7% efficiency of the requested 2 full CPU cores).
* After 45 seconds, the system seamlessly heals, automatically clearing out execution jobs and gracefully returning the app namespace to its healthy idle state.

---

## 🏅 Conclusion

This experiment successfully proves the stability of the cluster's underlying container runtime interception capabilities. By relying entirely on code-driven declarative manifests rather than graphical UI managers, this configuration provides a repeatable benchmark pattern for automated continuous resilience verification pipelines.
