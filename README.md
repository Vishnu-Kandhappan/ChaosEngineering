# Chaos Engineering Playground

This repository provides a hands-on guide for running chaos experiments against the [GoogleCloudPlatform microservices demo](https://github.com/GoogleCloudPlatform/microservices-demo) using [Chaos Mesh](https://chaos-mesh.org/). Use it to practice introducing controlled faults, observe system behavior, and harden services before issues reach production.

## Prerequisites
- Kubernetes cluster (Kind, Minikube, GKE, etc.) with `kubectl` configured
- [Helm](https://helm.sh/docs/intro/install/)
- [kubectl-kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) (bundled with modern `kubectl`)
- Optional: [Prometheus/Grafana stack](https://github.com/prometheus-operator/kube-prometheus) for observability

## 1. Create a local cluster with Kind (example)
```bash
kind create cluster --name chaos-playground --image kindest/node:v1.27.3
kubectl cluster-info
```

## 2. Deploy the microservices demo
The demo is namespaced for isolation and uses the upstream Kubernetes manifests.
```bash
kubectl create namespace hipster-shop
kubectl apply -n hipster-shop -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
kubectl wait -n hipster-shop --for=condition=Ready pods --all --timeout=300s
```
To access the frontend locally:
```bash
kubectl port-forward -n hipster-shop deployment/frontend 8080:8080
```

## 3. Install Chaos Mesh via Helm
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
After installation, expose the dashboard locally:
```bash
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```
Log in to the dashboard, grant the `cluster-scoped` permission when prompted, and select the `hipster-shop` namespace to target workloads.

## 4. Core Chaos Experiments and Why They Matter
Create experiments through the dashboard or by applying YAML definitions with `kubectl apply -f <file>`. Each experiment should have clear blast-radius limits (labels/namespace selectors), start and end times, and alerting hooks.

### PodKill
- **What it does:** Randomly deletes pods in a target deployment or statefulset.
- **Why:** Validates autoscaling, readiness/liveness probes, and service mesh retries so single-instance crashes do not cascade into outages.

### PodFailure
- **What it does:** Simulates container creation failures (e.g., image pull errors) without deleting existing pods.
- **Why:** Ensures rollouts and CI/CD pipelines handle bad releases gracefully and can automatically roll back.

### NetworkChaos (Delay/Loss/Duplication/Partition)
- **What it does:** Injects latency, packet loss, or partitions between services.
- **Why:** Reveals tail-latency sensitivity, timeout tuning, and retry storm risks that can overload downstream dependencies.

### StressChaos (CPU/Memory)
- **What it does:** Adds CPU or memory pressure to target pods or nodes.
- **Why:** Tests auto-scaling thresholds and verifies graceful degradation instead of hard crashes under load spikes.

### IOChaos
- **What it does:** Adds latency or faults to filesystem operations for a container.
- **Why:** Surfaces assumptions about fast disk access, ensuring data-intensive services keep working when I/O slows or fails intermittently.

### DNSChaos
- **What it does:** Returns error codes or wrong IPs for DNS queries.
- **Why:** Validates service discovery fallbacks and caches; prevents cascading failures when DNS infrastructure is degraded.

### TimeChaos
- **What it does:** Skews clocks within selected pods.
- **Why:** Exposes bugs in token expiration, certificate validation, and scheduled jobs that depend on synchronized time.

### KernelChaos
- **What it does:** Uses eBPF to inject low-level faults such as packet corruption or syscall failures.
- **Why:** Hardens components that rely on kernel behavior, catching rare edge cases that can lead to production outages.

## 5. Running a Sample Experiment (Network Delay)
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
Monitor the frontend for increased latency, verify alerts fire, and confirm the experiment auto-reverts after 5 minutes.

## 6. Cleanup
```bash
kubectl delete ns hipster-shop chaos-mesh
kind delete cluster --name chaos-playground
```

## Best Practices to Avoid Production Outages
- **Start small:** Limit experiments to non-critical paths, short durations, and minimal blast radius before expanding scope.
- **Automate observability:** Pair every experiment with metrics, logs, and tracing to measure impact and prove recovery.
- **Game days:** Run scheduled chaos scenarios with runbooks so teams practice detection, mitigation, and communication.
- **Guardrails:** Require approvals, maintenance windows, and automatic experiment pauses when SLOs are breached.
- **Learn and iterate:** Document findings, tune timeouts/retries, and improve deployment pipelines based on experiment outcomes.

This playbook turns the microservices demo into a safe sandbox for building resilient services and preventing real production outages.
