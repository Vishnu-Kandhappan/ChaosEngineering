# Chaos Engineering Playground

This repository provides a hands-on guide for running chaos experiments against the [GoogleCloudPlatform microservices demo](https://github.com/GoogleCloudPlatform/microservices-demo) using [Chaos Mesh](https://chaos-mesh.org/). It also highlights alternative platforms (LitmusChaos, stress-ng) so you can choose the right tool for your stack. Use it to practice introducing controlled faults, observe system behavior, and harden services before issues reach production.

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

### What you just deployed
The Hipster Shop demo consists of polyglot services communicating over gRPC/HTTP:
- **frontend (Go):** Serves the web UI.
- **productcatalogservice (Go):** Provides product data.
- **currencyservice (Node.js):** Handles currency conversion.
- **cartservice (C# .NET):** Manages user carts.
- **recommendationservice (Python):** Suggests related products.
- **adservice (Java):** Returns ads.
- **checkoutservice (Go):** Coordinates checkout flow.
- **paymentservice (Node.js):** Simulates payments.
- **shippingservice (Go):** Calculates shipping quotes.
- **emailservice (Python):** Sends order confirmations.
- **loadgenerator (Python):** Provides background traffic.

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

To enable the Chaos Mesh scheduler and metrics helpers:
```bash
kubectl annotate ns hipster-shop admission-webhook.pingcap.com/apply="true" --overwrite
kubectl -n chaos-mesh get pods
```

## 4. Core Chaos Experiments and Why They Matter
Create experiments through the dashboard or by applying YAML definitions with `kubectl apply -f <file>`. Each experiment should have clear blast-radius limits (labels/namespace selectors), start and end times, and alerting hooks. The snippets below can be pasted directly after updating labels/namespaces as needed.

### PodKill
- **What it does:** Randomly deletes pods in a target deployment or statefulset.
- **Why:** Validates autoscaling, readiness/liveness probes, and service mesh retries so single-instance crashes do not cascade into outages.

Example (kill one frontend pod every 10 minutes for 2 minutes):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
CHAOS_YAML
```

### PodFailure
- **What it does:** Simulates container creation failures (e.g., image pull errors) without deleting existing pods.
- **Why:** Ensures rollouts and CI/CD pipelines handle bad releases gracefully and can automatically roll back.

Example (block new cartservice pods for 5 minutes during a rollout):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
CHAOS_YAML
```

### NetworkChaos (Delay/Loss/Duplication/Partition)
- **What it does:** Injects latency, packet loss, or partitions between services.
- **Why:** Reveals tail-latency sensitivity, timeout tuning, and retry storm risks that can overload downstream dependencies.

### StressChaos (CPU/Memory)
- **What it does:** Adds CPU or memory pressure to target pods or nodes.
- **Why:** Tests auto-scaling thresholds and verifies graceful degradation instead of hard crashes under load spikes.

Example (CPU pressure on adservice for 3 minutes):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
CHAOS_YAML
```

### IOChaos
- **What it does:** Adds latency or faults to filesystem operations for a container.
- **Why:** Surfaces assumptions about fast disk access, ensuring data-intensive services keep working when I/O slows or fails intermittently.

Example (inject disk latency into cartservice for 5 minutes):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
  volumePath: /var/lib
  path: /
  containerNames:
    - server
CHAOS_YAML
```

### DNSChaos
- **What it does:** Returns error codes or wrong IPs for DNS queries.
- **Why:** Validates service discovery fallbacks and caches; prevents cascading failures when DNS infrastructure is degraded.

Example (randomize DNS answers for currencyservice for 3 minutes):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
CHAOS_YAML
```

### TimeChaos
- **What it does:** Skews clocks within selected pods.
- **Why:** Exposes bugs in token expiration, certificate validation, and scheduled jobs that depend on synchronized time.

Example (shift checkoutservice clock back 10 minutes during rollout tests):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
CHAOS_YAML
```

### KernelChaos
- **What it does:** Uses eBPF to inject low-level faults such as packet corruption or syscall failures.
- **Why:** Hardens components that rely on kernel behavior, catching rare edge cases that can lead to production outages.

Example (fail 30% of `openat` syscalls inside paymentservice for 2 minutes):
```bash
cat <<'CHAOS_YAML' | kubectl apply -f -
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
CHAOS_YAML
```

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

## 6. More scenarios with LitmusChaos
LitmusChaos provides CRDs and workflows similar to Chaos Mesh, with chaoscenter UI as an option.

Install the Litmus operator (namespace-scoped example):
```bash
kubectl create ns litmus
kubectl apply -n litmus -f https://litmuschaos.github.io/litmus/2.14.0/litmus-operator-lite.yaml
kubectl create serviceaccount -n hipster-shop litmus-sa
kubectl create clusterrolebinding litmus-sa --clusterrole=cluster-admin --serviceaccount=hipster-shop:litmus-sa
```

Pod delete experiment against `frontend` (terminates one pod every 10 minutes):
```bash
cat <<'LITMUS_YAML' | kubectl apply -f -
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
              value: "120"
            - name: CHAOS_INTERVAL
              value: "600"
            - name: PODS_AFFECTED_PERC
              value: "50"
LITMUS_YAML
```

CPU hog on `adservice` using Litmus (consume 2 cores for 3 minutes):
```bash
cat <<'LITMUS_YAML' | kubectl apply -f -
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
LITMUS_YAML
```

Network loss with Litmus (drop 30% packets between checkoutservice and downstream calls):
```bash
cat <<'LITMUS_YAML' | kubectl apply -f -
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
LITMUS_YAML
```

## 7. stress-ng one-liners for fast validation
Use [stress-ng](https://wiki.ubuntu.com/Kernel/Reference/stress-ng) inside the cluster (install in a debug pod) to validate resource limits without installing CRDs:

- CPU: `stress-ng --cpu 4 --cpu-load 80 --timeout 180s`
- IO: `stress-ng --hdd 2 --hdd-bytes 1G --timeout 180s`
- Memory: `stress-ng --vm 2 --vm-bytes 1G --oomable --timeout 180s`
- Network sockets: `stress-ng --sock 4 --timeout 120s`
- Mixed: `stress-ng --matrix 2 --stream 2 --timeout 180s`
