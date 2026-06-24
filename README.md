# Observability Agent Helm Charts

Agent-side observability stack for multi-cluster setup. Each agent cluster forwards **metrics → vmagent**, **logs → Fluent Bit**, **traces → OTel Collector** to the central observability cluster via internal ALB.

## Repository Structure

```
observability/
├── Non-prod/
│   ├── fluentbit/
│   │   ├── ALCSNG/
│   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   └── charts/fluent-bit-0.48.10.tgz
│   │   ├── HCMNG/
│   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   └── charts/
│   │   └── Hiretech/
│   │       ├── Chart.yaml  values.yaml
│   │       └── charts/
│   ├── otel/
│   │   ├── ALCSNG/
│   │   │   ├── otel-operator/
│   │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
│   │   │   └── otel-collector/
│   │   │       ├── Chart.yaml  Chart.lock  values.yaml
│   │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
│   │   │       └── templates/instrumentation.yaml
│   │   ├── HCMNG/
│   │   │   ├── otel-operator/
│   │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
│   │   │   └── otel-collector/
│   │   │       ├── Chart.yaml  Chart.lock  values.yaml
│   │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
│   │   │       └── templates/instrumentation.yaml
│   │   └── Hiretech/
│   │       ├── otel-operator/
│   │       │   ├── Chart.yaml  values.yaml
│   │       └── otel-collector/
│   │           ├── Chart.yaml  values.yaml
│   │           └── templates/instrumentation.yaml
│   └── vmagent/
│       ├── ALSCNG/
│       │   ├── Chart.yaml  Chart.lock  values.yaml
│       │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
│       ├── HCMNG/
│       │   ├── Chart.yaml  Chart.lock  values.yaml
│       │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
│       └── Hiretech/
│           ├── Chart.yaml  Chart.lock  values.yaml
│           └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
└── Prod/
    ├── fluentbit/
    │   ├── ALCSNG/
    │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   └── charts/fluent-bit-0.48.10.tgz
    │   ├── HCMNG/
    │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   └── charts/
    │   └── Hiretech/
    │       ├── Chart.yaml  values.yaml
    │       └── charts/
    ├── otel/
    │   ├── ALCSNG/
    │   │   ├── otel-operator/
    │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
    │   │   └── otel-collector/
    │   │       ├── Chart.yaml  Chart.lock  values.yaml
    │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
    │   │       └── templates/instrumentation.yaml
    │   ├── HCMNG/
    │   │   ├── otel-operator/
    │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
    │   │   └── otel-collector/
    │   │       ├── Chart.yaml  Chart.lock  values.yaml
    │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
    │   │       └── templates/instrumentation.yaml
    │   └── Hiretech/
    │       ├── otel-operator/
    │       │   ├── Chart.yaml  values.yaml
    │       └── otel-collector/
    │           ├── Chart.yaml  values.yaml
    │           └── templates/instrumentation.yaml
    └── vmagent/
        ├── ALSCNG/
        │   ├── Chart.yaml  Chart.lock  values.yaml
        │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
        ├── HCMNG/
        │   ├── Chart.yaml  Chart.lock  values.yaml
        │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
        └── Hiretech/
            ├── Chart.yaml  values.yaml
            └── charts/
```

## Cluster Reference

| Cluster | Environment | Namespace | vmagent Release | Fluent Bit Release | OTel Operator Release | OTel Collector Release |
|---|---|---|---|---|---|---|
| HCMNG | Non-prod + Prod | `opstree-observability` | `vm` | `fluent-bit` | `otel-operator` | `otel-collector` |
| ALCSNG | Non-prod + Prod | `opstree-observability` | `vm` | `fluent-bit` | `otel-operator` | `otel-collector` |
| Hiretech | Non-prod + Prod | `opstree-observability` | `vm` | `fluent-bit` | `otel-operator` | `otel-collector` |

## Prerequisites

```bash
# Update kubeconfig for target cluster
aws eks update-kubeconfig --name <cluster-name> --region ap-south-1 --profile <profile>

# Create observability namespace
kubectl create namespace opstree-observability

# Verify context
kubectl config current-context
```

## 1. VMAgent

Scrapes metrics from all namespaces and remote-writes to central VictoriaMetrics via internal ALB.

### Step 1 — Update dependencies

```bash
cd <Non-prod|Prod>/vmagent/<CLUSTER_NAME>

helm dependency update .

# Verify charts are downloaded
ls charts/
```

### Step 2 —  Installation

```bash
# First install with vmagent disabled (installs operator only)
helm install vm . \
  -n opstree-observability \
  --set vm.vmagent.enabled=false

# Wait for operator to be ready
kubectl get pods -n opstree-observability -w
# Wait until vm-operator shows 1/1 Running

# Then enable vmagent
helm upgrade vm . \
  -n opstree-observability \
  -f values.yaml
```

### Upgrade

```bash
cd <Non-prod|Prod>/vmagent/<CLUSTER_NAME>

helm upgrade vm . \
  -n opstree-observability \
  -f values.yaml
```

### Verify

```bash
# Check pod is running (should show 2/2)
kubectl get pods -n opstree-observability | grep vmagent

```

## 2. Fluent Bit

Runs as a DaemonSet on every node. Collects container logs from `/var/log/containers/` and forwards to central Loki.

### Step 1 — Update dependencies

```bash
cd <Non-prod|Prod>/fluentbit/<CLUSTER_NAME>

helm dependency update .

# Verify charts are downloaded
ls charts/
```

### Step 2 —  Installation

```bash
helm install fluent-bit . \
  -n opstree-observability \
  -f values.yaml
```

### Upgrade

```bash
cd <Non-prod|Prod>/fluentbit/<CLUSTER_NAME>

helm upgrade fluent-bit . \
  -n opstree-observability \
  -f values.yaml
```

### Verify

```bash
# Check DaemonSet — DESIRED should equal total node count
kubectl get daemonset -n opstree-observability | grep fluent-bit

# Check all pods are Running
kubectl get pods -n opstree-observability -l app.kubernetes.io/name=fluent-bit -o wide

```

## 3. OpenTelemetry Collector

Receives OTLP traces from instrumented app pods, enriches with Kubernetes metadata, and exports to central Tempo.

### Step 1 — Check if otel-operator already installed

```bash
kubectl get pods -A | grep otel-operator
```

> **If otel-operator already exists**, skip Step 2 and go directly to Step 3. Installing a second operator will conflict with existing cluster-scoped CRDs.

### Step 2 — Install otel-operator (skip if already exists)

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-operator

# Update dependencies first
helm dependency update .

# Verify charts downloaded
ls charts/

# Install
helm install otel-operator . \
  -n opstree-observability \
  -f values.yaml
```

#### Upgrade otel-operator

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-operator

helm upgrade otel-operator . \
  -n opstree-observability \
  -f values.yaml
```

### Step 3 — Apply Instrumentation CR

```bash
kubectl apply -f <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-collector/templates/instrumentation.yaml

# Verify
kubectl get instrumentation -n opstree-observability
```

### Step 4 — Deploy otel-collector

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-collector

# Update dependencies first
helm dependency update .

# Verify charts downloaded
ls charts/

# Install
helm install otel-collector . \
  -n opstree-observability \
  -f values.yaml
```

#### Upgrade otel-collector

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-collector

helm upgrade otel-collector . \
  -n opstree-observability \
  -f values.yaml
```

### Step 5 — Add annotation to app deployments

| Language | Annotation |
|---|---|
| Node.js / NestJS | `instrumentation.opentelemetry.io/inject-nodejs: "opstree-observability/otel-instrumentation"` |
| Java | `instrumentation.opentelemetry.io/inject-java: "opstree-observability/otel-instrumentation"` |
| Python | `instrumentation.opentelemetry.io/inject-python: "opstree-observability/otel-instrumentation"` |
| React (frontend) | No annotation — runs in browser |

```bash
# Restart deployment to pick up annotation
kubectl rollout restart deployment/<app-name> -n <app-namespace>
```


### Verify

```bash
# Check collector pod is running
kubectl get pods -n opstree-observability | grep otel-collector

# Check health
kubectl port-forward -n opstree-observability \
  svc/otel-collector-collector 13133:13133
curl http://localhost:13133/

# Check collector logs
kubectl logs -n opstree-observability \
  $(kubectl get pod -n opstree-observability -l app.kubernetes.io/name=otel-collector \
  -o jsonpath='{.items[0].metadata.name}') \
  --tail=20 | grep -iE "error|failed"
```

## Central ALB Endpoint Paths

| Signal | Path | Agent |
|---|---|---|
| Metrics | `/insert/0/prometheus/api/v1/write` | vmagent remoteWrite |
| Logs | `/loki/api/v1/push` | Fluent Bit loki output |
| Traces | `/v1/traces` | OTel Collector otlphttp exporter |

## Helm Release Reference

| Component | Release Name | Namespace | Chart Location |
|---|---|---|---|
| VMAgent | `vm` | `opstree-observability` | `<env>/vmagent/<CLUSTER>` |
| Fluent Bit | `fluent-bit` | `opstree-observability` | `<env>/fluentbit/<CLUSTER>` |
| OTel Operator | `otel-operator` | `opstree-observability` | `<env>/otel/<CLUSTER>/otel-operator` |
| OTel Collector | `otel-collector` | `opstree-observability` | `<env>/otel/<CLUSTER>/otel-collector` |


```
