# Local OpenTelemetry Demo Setup with kind & Podman Desktop (macOS)

This setup enables you to run the OpenTelemetry Demo locally using a Kubernetes cluster provisioned with `kind` and containerized using Podman Desktop on macOS. It is tailored for reproducible scientific experiments in observability and cloud-native monitoring.

## 🧰 Requirements

- macOS (tested on macOS Sequoia Version 15.5)
- [Podman Desktop](https://podman.io/)
- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/)
- [k9s (optional)](https://k9scli.io/)

## 🚀 Initial Setup and Cluster Provisioning

### 1. Create Podman Machine (VM)

```bash
# Apple MacBook Pro M1 16 GB 2021:
podman machine init podman-machine-otel-demo --cpus 8 --memory 10248 --disk-size 150

# Apple MacBook Air M1 2020:
podman machine init podman-machine-otel-demo --cpus 8 --memory 8192 --disk-size 150
```

### 2. Start Podman Machine
Start Podman VM via the Podman Desktop App or by the following command.

```bash
podman machine start podman-machine-otel-demo
```

### 3. Create Kind Cluster
> ⚠️ **Important Notice:**  
> The following environment variable tells kind to use Podman instead of Docker as the container runtime.
> It may be needed on macOS if both Podman and Docker Desktop are installed and you want to force kind to use Podman.

```bash
export KIND_EXPERIMENTAL_PROVIDER=podman
```

```bash
# Apple MacBook Pro M1 16 GB 2021:
kind create cluster --name observability-platform --config ./local-dev/kind-config.yaml

# Apple MacBook Air M1 2020:
kind create cluster --name observability-platform
```

### 4. Create Kubernetes namespace
```bash
kubectl create ns observability
```

### 5. Add the OpenTelemetry Helm Chart Repository
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

### 6. Update Your Local Helm Repository Cache
```bash
helm repo update
```

### 7. Install the OpenTelemetry Demo Using Helm
> ⚠️ **Important Notice**
> Before starting the OpenTelemetry Demo Kubernetes cluster with kind, **please close other applications** such as browsers, IDEs, or heavy background processes. This ensures your system has **enough CPU and memory resources** available to run the cluster smoothly and avoid issues like pod crashes or slow startup.
💡 **Tip for laptop users**
> Running Kubernetes workloads locally is CPU- and memory-intensive. If you're using a MacBook or similar laptop, it's strongly recommended to **plug it into power** while running the cluster. In battery mode, macOS may throttle performance, which can lead to slower startup times or unstable workloads.
```bash
helm install observability-platform-demo open-telemetry/opentelemetry-demo --version 0.37.1 --namespace observability
```
> 🎯 **Why a fixed version?**
We use a specific version (0.37.1) of the OpenTelemetry Demo to ensure reproducibility and consistency for scientific evaluation. Helm chart versions may change over time and introduce breaking changes or new behaviors.
If you want to use the latest available version, simply omit the --version flag:
```bash
helm install observability-platform-demo open-telemetry/opentelemetry-demo --namespace observability
```
> 📌 To view all available versions:
```bash
helm search repo open-telemetry/opentelemetry-demo --versions
```

### 8. Observe the Cluster Startup with k9s
In a new terminal tab, you can monitor the startup of the Kubernetes cluster using ```k9s```. This helps you verify that all pods are initializing correctly after executing the Helm install in step 7.

```bash
k9s --context kind-observability-platform
```
This will open a terminal-based UI where you can see the pods and their status live. Wait until all pods are in a Running or Completed state before proceeding.

![k9s running screenshot](./readme-img/k9s-opentelemetry-demo-screenshot-2025-07-10.png)

### 9. Port forwarding
To access the individual components of the OpenTelemetry Demo via your web browser the frontend-proxy service can be used. Run the following command to make the services available:

```bash
kubectl port-forward service/frontend-proxy 8080:8080 --namespace observability
```

The following web interfaces are accessible locally through port forwarding:

| Service             | URL                                  | Description                      |
|---------------------|---------------------------------------|----------------------------------|
| Web Store           | [http://localhost:8080](http://localhost:8080)                 | Demo frontend of the web shop   |
| Grafana             | [http://localhost:8080/grafana](http://localhost:8080/grafana) | Metrics & dashboards            |
| Feature Flags UI    | [http://localhost:8080/feature](http://localhost:8080/feature) | Feature flag management         |
| Load Generator UI   | [http://localhost:8080/loadgen/](http://localhost:8080/loadgen/) | Simulates user traffic          |
| Jaeger UI           | [http://localhost:8080/jaeger/ui](http://localhost:8080/jaeger/ui) | Distributed tracing             |


#### Prometheus
To access the Prometheus web UI, forward its service in a separate terminal tab using:

```bash
kubectl port-forward svc/prometheus 9090:9090 --namespace observability
```
| Service             | URL                                  | Description                      |
|---------------------|---------------------------------------|----------------------------------|
| Prometheus UI           | [http://localhost:9090](http://localhost:9090)                 | Monitoring and alerting system for metrics collection and querying |

#### Alternative: Forward each service in a separate terminal tab
If the default port-forward through frontend-proxy does not work or you want to access individual services directly, you can forward each service separately in its own terminal tab. This approach can help isolate issues and gives direct access to service UIs.

```bash
# Frontend (Onlineshop)
kubectl port-forward svc/frontend-proxy 8080:8080 -n observability
# Access at http://localhost:8080/

# Prometheus
kubectl port-forward svc/prometheus 9090:9090 -n observability
# Access at http://localhost:9090/

# Grafana
kubectl port-forward svc/grafana 3000:80 -n observability
# Access at http://localhost:3000/

# Jaeger
kubectl port-forward svc/jaeger-query 16686:16686 -n observability
# Access at http://localhost:16686/

# For Locust (Load Generator) use:
http://localhost:8080/loadgen/ (via frontend-proxy)
```

## 🛑 Shut Down the Cluster and Demo
To shut down the OpenTelemetry demo and the local Kubernetes cluster cleanly:

### 1. Close all terminal windows with active kubectl port-forward
If you started port-forwardings in separate tabs, simply close those terminal tabs or press ```Ctrl+C``` in each.

To forcefully stop all port-forwardings:
```bash
pkill -f "kubectl port-forward"
```

### 2. Uninstall the OpenTelemetry demo
This removes all deployed demo components but keeps the namespace and cluster:
```bash
helm uninstall observability-platform-demo -n observability
```

### 3. Delete the namespace
Optional: If you want a fully clean state:

```bash
kubectl delete ns observability
```

### 4. Stop the kind cluster (indirectly by stopping Podman)
```bash
podman machine stop podman-machine-otel-demo
```
- This also stops the kind cluster, which is running inside Podman.
- No need to explicitly stop kind — it's tied to Podman's lifecycle.


## ✅ Re-initialize and restart the Cluster
If you want to completely reset the cluster, run:

```bash
kind delete cluster --name observability-platform
```

Then repeat steps 2, 3, 4, and 7 from [Initial Setup and Cluster Provisioning](#1-initial-setup-and-cluster-provisioning).

- Step 2: Start Podman Machine
- Step 3: Create the kind cluster again
- Step 4: Create the observability namespace
- Step 7: Install the OpenTelemetry demo via Helm


## Make changes to the OpenTelemetry Demo (Log)

### cd to your workspace
```bash
cd /Users/markusschmid/Workspaces/OpenTelemetryDemoApp/opentelemetry-demo
```

### Make a change to a service's file (e.g. cart service)
```bash
vim /Users/markusschmid/Workspaces/OpenTelemetryDemoApp/opentelemetry-demo/src/cart/src/cartstore/ValkeyCartStore.cs
```

### Build service according to its own README.md file (e.g. cart with Microsoft .NET tools)
```bash
cd src/cart
dotnet restore
dotnet build
```

### Build container image for cart
```bash
cd ../...
docker-compose build cart
```

### (Optional) Build container images for entire project
```bash
docker-compose build # Works better
podman-compose build # Works
```

### Check podman container images
```bash
podman images
```

### Tagging
```bash
podman tag ghcr.io/open-telemetry/demo:latest-cart localhost/open-telemetry/demo:kusilog
```

### Create tar file
```bash
podman save localhost/open-telemetry/demo:kusilog -o otel-cart.tar
```

### Save image to kind
```bash
kind load image-archive otel-cart.tar --name observability-platform
```

### Create or adapt custom-values.yaml file to define which container image to user
```bash
vim /Users/markusschmid/Workspaces/OpenTelemetryDemoApp/opentelemetry-demo/local-dev/custom-values.yaml
```

### Deploy to kind using local image according to custom-values.yaml (cart from local repository)
This command runs the OpenTelemetry demo mainly with the online repository. Services mentioned in the
custom-values.yaml are started from the local built and deployed source code though.
```bash
helm install observability-platform-demo open-telemetry/opentelemetry-demo \
  --namespace observability \
  --version 0.37.1 \
  -f local-dev/custom-values.yaml
```

### Redeploy a changed container image to a running Kubernetes cluster
```bash
helm upgrade observability-platform-demo open-telemetry/opentelemetry-demo \
  --namespace observability \
  --version 0.37.1 \
  -f local-dev/custom-values.yaml
```
## Make changes to the OpenTelemetry Demo (Metric)

### cd to your workspace
```bash
cd /Users/markusschmid/Workspaces/OpenTelemetryDemoApp/opentelemetry-demo
```

### Make a change to a service's file (e.g. checkout service)
```bash
vim /Users/markusschmid/Workspaces/OpenTelemetryDemoApp/opentelemetry-demo/src/checkout/main.go
```

### Build service according to its own README.md file (e.g. checkout with Go tools)
```sh
cd src/checkout
go build -o bin/checkout/
```

### Build container image for cart
```sh
docker-compose build checkout
```

### Check podman container images
```bash
podman images
```

### Tagging
```bash
podman tag ghcr.io/open-telemetry/demo:latest-checkout localhost/open-telemetry/demo:kusimetric
```

### Create tar file
```bash
podman save localhost/open-telemetry/demo:kusimetric -o otel-checkout.tar
```

### Save image to kind
```bash
kind load image-archive otel-checkout.tar --name observability-platform
```

### Create or adapt custom-values.yaml file to define which container image to user
```bash
vim /Users/markusschmid/Workspaces/OpenTelemetryDemoApp/opentelemetry-demo/local-dev/custom-values.yaml
```

### Deploy to kind using local image according to custom-values.yaml (cart from local repository)
This command runs the OpenTelemetry demo mainly with the online repository. Services mentioned in the
custom-values.yaml are started from the local built and deployed source code though.
```bash
helm install observability-platform-demo open-telemetry/opentelemetry-demo \
  --namespace observability \
  --version 0.37.1 \
  -f local-dev/custom-values.yaml
```

### Redeploy a changed container image to a running Kubernetes cluster
```bash
helm upgrade observability-platform-demo open-telemetry/opentelemetry-demo \
  --namespace observability \
  --version 0.37.1 \
  -f local-dev/custom-values.yaml

## 📄 License
This setup is based on open-source tools and intended for academic and research use. See individual tool licenses for details.