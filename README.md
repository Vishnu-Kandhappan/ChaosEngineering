# Chaos Engineering Playground

This repository provides a hands-on guide for running chaos experiments against the GoogleCloudPlatform microservices demo (Hipster Shop) using Chaos Mesh and LitmusChaos. The intent is to demonstrate common chaos experiments (pod kills, network faults, resource stress, time skews, kernel faults, etc.) and to show how these experiments help validate resilience patterns (retries, timeouts, autoscaling, graceful degradation).

IMPORTANT: Chaos experiments can cause service disruptions. Always run in an isolated namespace or cluster and limit blast radius using selectors and schedules.

## Prerequisites
- Kubernetes cluster (Kind, Minikube, GKE, EKS, AKS, or any conformant cluster) with `kubectl` configured.
  - For Kind, see kind installation: https://kind.sigs.k8s.io/
- Helm (https://helm.sh/docs/intro/install/)
- `kubectl` with `kustomize` support (modern kubectl bundles kustomize)
- Optional: Prometheus/Grafana/Kube-Prometheus for observability (recommended)
- Optional: `jq`, `yq` for inspecting/manipulating YAML in the shell

Notes:
- If you're using Kind (or a cluster using containerd), you may need to set Chaos Mesh daemon runtime and socketPath accordingly (examples below).
- KernelChaos and eBPF-based experiments require node/kernel support and CAP_SYS_ADMIN/privileged access — these often won't work on managed or constrained environments (e.g., default Kind without elevated privileges).

---

## 1. Create a local cluster with Kind (example)
Example (Kind with Kubernetes v1.27.x):
```bash
kind create cluster --name chaos-playground --image kindest/node:v1.27.3
kubectl cluster-info
```

If you use Docker as the node runtime (older setups), Chaos Mesh daemon runtime settings differ — see the Chaos Mesh docs for compatible settings.

---

## 2. Deploy the Hipster Shop microservices demo
The demo is namespaced for isolation and uses the upstream Kubernetes manifests.

```bash
kubectl create namespace hipster-shop
kubectl apply -n hipster-shop -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
# Wait for pods to be ready (adjust timeout as needed)
kubectl wait -n hipster-shop --for=condition=Ready pods --all --timeout=300s
```

Validate deployments and pods:
```bash
kubectl get deployments -n hipster-shop
kubectl get pods -n hipster-shop -o wide
kubectl get svc -n hipster-shop
```

Common deployments/services present in the demo (verify these after deploy):
- frontend
- productcatalogservice
- currencyservice
- cartservice
- recommendationservice
- adservice
- checkoutservice
- paymentservice
- shippingservice
- emailservice
- loadgenerator

To access the frontend locally (port-forward):
```bash
kubectl -n hipster-shop port-forward deployment/frontend 8080:8080
# then open http://localhost:8080
```
Alternatively, expose as NodePort or configure an Ingress if you want cluster-wide access.

---

## 3. Install Chaos Mesh via Helm
Add the Chaos Mesh Helm repo and install into the `chaos-mesh` namespace. Adjust `chaosDaemon.runtime` and `chaosDaemon.socketPath` for your environment (containerd vs docker). For Kind (containerd) the following settings are common; for Docker runtime you would set `chaosDaemon.runtime=docker` and use the appropriate socket path.

```bash
kubectl create namespace chaos-mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update

helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace=chaos-mesh \
  --version 2.6.3 \
  --set dashboard.create=true \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

Expose the Chaos Mesh dashboard locally:
```bash
kubectl -n chaos-mesh port-forward svc/chaos-dashboard 2333:2333
# Dashboard available at http://localhost:2333
```

When you first access the dashboard it will prompt for permission; if you want cluster-wide experiments, grant the cluster-scoped permission when prompted or configure RBAC per the docs.

Enable namespace injection / scheduler and the metrics sidecars if needed:
```bash
kubectl annotate ns hipster-shop admission-webhook.pingcap.com/apply="true" --overwrite
kubectl -n chaos-mesh get pods
```

Notes:
- If your cluster uses Docker runtime or a different socket path, set `--set chaosDaemon.runtime=docker` and `--set chaosDaemon.socketPath=/var/run/dockershim.sock` (or other appropriate socket). Check your node runtime before installing.
- Chaos Mesh requires webhook admission; ensure your cluster allows this.

---

## 4. Core Chaos Experiments and Why They Matter
You can create experiments through the Chaos Mesh dashboard or by applying YAML definitions with `kubectl apply -f <file>`. Each experiment should have clear blast-radius limits (namespace and label selectors), a fixed duration, and a scheduler where appropriate.

Best practices for experiments:
- Start with short, low-impact experiments (one pod at a time, low percent).
- Scope experiments by namespace and `labelSelectors`.
- Ensure monitoring/alerting is enabled to capture downstream effects.
- Document expected behavior and rollback criteria.

### PodKill
- What it does: randomly deletes pods in a target deployment or statefulset.
- Why: validates autoscaling, readiness/liveness probes, and retry behavior so single-instance crashes do not cascade into outages.

Example (kill one frontend pod every 10 minutes for 2 minutes):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: hipster-frontend-podkill
  namespace: hipster-shop
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: frontend
  duration: '2m'
  scheduler:
    cron: '@every 10m'
```

Apply with:
```bash
kubectl apply -f hipster-frontend-podkill.yaml
```

### PodFailure
- What it does: simulates container creation failures (e.g., image pull errors) without deleting existing pods.
- Why: ensures rollouts and CI/CD pipelines handle bad releases gracefully and can automatically roll back.

Example (block new cartservice pods for 5 minutes during rollouts):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: hipster-cart-podfailure
  namespace: hipster-shop
spec:
  action: pod-failure
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: cartservice
  duration: '5m'
  scheduler:
    cron: '@every 20m'
```

### NetworkChaos (Delay / Loss / Duplication / Partition)
- What it does: injects latency, packet loss, or partitions between services.
- Why: reveals tail-latency sensitivity, timeout tuning, and retry storm risks that can overload downstream dependencies.

Example (introduce latency to frontend):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: hipster-frontend-delay
  namespace: hipster-shop
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: frontend
  delay:
    latency: '500ms'
    jitter: '50ms'
  duration: '5m'
  scheduler:
    cron: '@every 15m'
```

### StressChaos (CPU / Memory)
- What it does: adds CPU or memory pressure to target pods.
- Why: tests auto-scaling thresholds and verifies graceful degradation under load spikes.

Example (CPU pressure on adservice for 3 minutes):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: hipster-adservice-cpu
  namespace: hipster-shop
spec:
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: adservice
  mode: one
  stressors:
    cpu:
      workers: 2
      load: 80
  duration: '3m'
  scheduler:
    cron: '@every 30m'
```

### IOChaos
- What it does: adds latency or faults to filesystem operations for a container.
- Why: surfaces assumptions about fast disk access, ensuring data-intensive services degrade gracefully.

Example (inject disk latency into cartservice for 5 minutes):
- WARNING: confirm the container name (containerNames) and paths used by the target pod. In the Hipster Shop containers, the server container may be named `server`, but verify with `kubectl -n hipster-shop get pod <pod> -o yaml`.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: hipster-cart-disk-latency
  namespace: hipster-shop
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: cartservice
  delay: '200ms'
  percent: 50
  duration: '5m'
  scheduler:
    cron: '@every 20m'
  volumePath: /var/lib   # path inside pod's filesystem or volume path; adjust as required
  path: /                # file system path to affect
  containerNames:
    - server
```

### DNSChaos
- What it does: returns error codes or wrong IPs for DNS queries.
- Why: validates service discovery fallbacks and caches; prevents cascading failures when DNS infrastructure is degraded.

Example (randomize DNS answers for currencyservice for 3 minutes):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: DNSChaos
metadata:
  name: hipster-currency-dns-random
  namespace: hipster-shop
spec:
  action: random
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: currencyservice
  patterns:
    - "*.google.com"
    - "*.hipster.shop"
  duration: '3m'
  scheduler:
    cron: '@every 30m'
```
Note: DNSChaos schema and fields can change between versions — consult Chaos Mesh docs for the exact schema for your version.

### TimeChaos
- What it does: skews clocks within selected pods.
- Why: exposes bugs in token expiration, certificate validation, and scheduled jobs that depend on synchronized time.

Example (shift checkoutservice clock back 10 minutes during rollout tests):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: TimeChaos
metadata:
  name: hipster-checkout-timeskew
  namespace: hipster-shop
spec:
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: checkoutservice
  timeOffset: '-10m'
  clockIds:
    - CLOCK_REALTIME
  duration: '10m'
  scheduler:
    cron: '@every 1h'
```

### KernelChaos
- What it does: uses eBPF or kernel-level hooks to inject faults such as syscall failures or packet corruption.
- Why: hardens components that rely on kernel behavior, catching rare edge cases that can lead to production outages.

WARNING: KernelChaos requires kernel/eBPF support and privileged host access. It may not run on managed clusters or unprivileged local clusters (e.g., default Kind without extra privileges).

Example (fail 30% of `openat` syscalls inside paymentservice for 2 minutes):
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: KernelChaos
metadata:
  name: hipster-payment-openat-fail
  namespace: hipster-shop
spec:
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: paymentservice
  failKernRequest:
    failtype: 0
    probability: 30
    callchain:
      - funcname: "__x64_sys_openat"
  duration: '2m'
  scheduler:
    cron: '@every 45m'
```

---

## 5. Running a Sample Experiment (Network Delay)
Apply this network delay and monitor frontend latency and alerts. The experiment will automatically revert after the configured duration.

```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: hipster-frontend-delay
  namespace: hipster-shop
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - hipster-shop
    labelSelectors:
      app: frontend
  delay:
    latency: '500ms'
    jitter: '50ms'
  duration: '5m'
  scheduler:
    cron: '@every 15m'
CHAOS_YAML
```

Monitor:
```bash
# Pods and events
kubectl get pods -n hipster-shop
kubectl describe pod <frontend-pod> -n hipster-shop

# Observe metrics in Prometheus/Grafana if installed
```

---

## 6. More scenarios with LitmusChaos
LitmusChaos provides CRDs and workflows similar to Chaos Mesh, with chaoscenter UI as an option. Below are example steps to install a lightweight operator and run experiments namespace-scoped.

Install the Litmus operator:
```bash
kubectl create ns litmus
kubectl apply -n litmus -f https://litmuschaos.github.io/litmus/2.14.0/litmus-operator-lite.yaml
# Create a service account for chaos experiments
kubectl create serviceaccount -n hipster-shop litmus-sa
kubectl create clusterrolebinding litmus-sa --clusterrole=cluster-admin --serviceaccount=hipster-shop:litmus-sa
```

Example ChaosEngine to delete pods (terminates one pod every 10 minutes):
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: frontend-pod-delete
  namespace: hipster-shop
spec:
  appinfo:
    appns: hipster-shop
    applabel: "app=frontend"
    appkind: deployment
  chaosServiceAccount: litmus-sa
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "120"   # seconds
            - name: CHAOS_INTERVAL
              value: "600"   # seconds
            - name: PODS_AFFECTED_PERC
              value: "50"
```

CPU hog on `adservice` using Litmus (consume 2 cores for 3 minutes):
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: adservice-cpu-hog
  namespace: hipster-shop
spec:
  appinfo:
    appns: hipster-shop
    applabel: "app=adservice"
    appkind: deployment
  chaosServiceAccount: litmus-sa
  experiments:
    - name: node-cpu-hog
      spec:
        components:
          env:
            - name: CPU_CORES
              value: "2"
            - name: TOTAL_CHAOS_DURATION
              value: "180"
            - name: NODE_CPU_CORE
              value: "2"
```

Network loss with Litmus (drop 30% packets for checkoutservice downstream calls):
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: checkout-network-loss
  namespace: hipster-shop
spec:
  appinfo:
    appns: hipster-shop
    applabel: "app=checkoutservice"
    appkind: deployment
  chaosServiceAccount: litmus-sa
  experiments:
    - name: pod-network-loss
      spec:
        components:
          env:
            - name: NETWORK_PACKET_LOSS_PERCENTAGE
              value: "30"
            - name: TOTAL_CHAOS_DURATION
              value: "240"
            - name: PODS_AFFECTED_PERC
              value: "50"
```

---

## 7. stress-ng one-liners for fast validation
Use `stress-ng` inside a debug pod to validate resource limits quickly (no CRDs required). Install `stress-ng` inside a debug pod such as `busybox` or `ubuntu` and run:

- CPU: `stress-ng --cpu 4 --cpu-load 80 --timeout 180s`
- IO: `stress-ng --hdd 2 --hdd-bytes 1G --timeout 180s`
- Memory: `stress-ng --vm 2 --vm-bytes 1G --oomable --timeout 180s`
- Network sockets: `stress-ng --sock 4 --timeout 120s`
- Mixed: `stress-ng --matrix 2 --stream 2 --timeout 180s`

Example: create a debug pod and run stress:
```bash
kubectl -n hipster-shop run -it --rm stress-debug --image=alpine -- sh -c "apk add --no-cache stress-ng ; stress-ng --cpu 2 --timeout 120s"
```

---

## Tips and troubleshooting
- If a Chaos Mesh CRD application fails, inspect the logs for the `chaos-controller-manager` and `chaos-daemon` in `chaos-mesh` namespace:
  ```bash
  kubectl -n chaos-mesh logs deploy/chaos-controller-manager
  kubectl -n chaos-mesh logs ds/chaos-daemon
  ```
- Validate label selectors: ensure the labels you use in `labelSelectors` match the pods you intended. Example:
  ```bash
  kubectl get pods -n hipster-shop --show-labels
  ```
- For IOChaos and container-level experiments, verify the target `containerNames` from the pod spec:
  ```bash
  kubectl -n hipster-shop get pod <pod-name> -o yaml
  ```
- KernelChaos and eBPF-based experiments often require `privileged` pods or DaemonSet with host kernel access — do not run these on shared environments without approval.
- Always have monitoring and alerts enabled, and ensure you have rollback/runbook steps documented for any experiment causing unexpected impacts.
