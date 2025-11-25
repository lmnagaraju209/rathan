# Docker Images from ACR - Complete Inventory

This document lists all Docker images used from Azure Container Registry (ACR) and other container registries in the Digital SOH Istio project.

## 1. Azure Container Registry (acr14031e1dv01.azurecr.io)

### Istio Images
These images are pushed to ACR from Docker Hub via the `install-prereqs.ps1` script.

#### Supported Istio Versions:
- 1.13.1
- 1.17.2
- 1.17.5
- 1.17.6
- 1.19.0
- 1.20.5
- 1.21.2
- 1.22.5

#### Images for Each Version:

**Regular Base Images:**
1. **acr14031e1dv01.azurecr.io/istio/operator:$VERSION**
   - Purpose: Istio Operator
   - Versions: All supported versions listed above

2. **acr14031e1dv01.azurecr.io/istio/pilot:$VERSION**
   - Purpose: Istio Control Plane (Pilot/Istiod)
   - Versions: All supported versions listed above

3. **acr14031e1dv01.azurecr.io/istio/proxyv2:$VERSION**
   - Purpose: Istio Sidecar Proxy
   - Versions: All supported versions listed above

**Distroless Base Images:**
4. **acr14031e1dv01.azurecr.io/istio/operator:$VERSION-distroless**
   - Purpose: Istio Operator (Distroless variant)
   - Versions: All supported versions listed above

5. **acr14031e1dv01.azurecr.io/istio/pilot:$VERSION-distroless**
   - Purpose: Istio Control Plane (Distroless variant)
   - Versions: All supported versions listed above

6. **acr14031e1dv01.azurecr.io/istio/proxyv2:$VERSION-distroless**
   - Purpose: Istio Sidecar Proxy (Distroless variant)
   - Versions: All supported versions listed above

---

## 2. Monitoring & Observability Components (ACR with Variable Substitution)

These images use `#{ACR_NAME}#` variable substitution in deployment manifests.

### Prometheus Stack

1. **#{ACR_NAME}#.azurecr.io/prom/prometheus:v2.41.0**
   - Component: Prometheus Server
   - Files: 
     - `addons/monitoring/prometheus/prometheus-v2.41.0.yml`
     - `addons/monitoring/prometheus/prometheus-v15.9.0.yml`

2. **#{ACR_NAME}#.azurecr.io/jimmidyson/configmap-reload:v0.8.0**
   - Component: ConfigMap Reload Sidecar
   - Files: 
     - `addons/monitoring/prometheus/prometheus-v2.41.0.yml`
     - `addons/monitoring/prometheus/prometheus-v15.9.0.yml`

### Kiali (Service Mesh Observability)

3. **#{ACR_NAME}#.azurecr.io/istio/kiali:v1.67**
   - Component: Kiali Dashboard
   - Files:
     - `addons/monitoring/kiali/kiali-v1.83.0.yml`
     - `addons/monitoring/kiali/kiali-v1.63.1.yml`

### Jaeger (Distributed Tracing)

4. **#{ACR_NAME}#.azurecr.io/jaegertracing/all-in-one:1.48**
   - Component: Jaeger All-in-One
   - File: `addons/monitoring/jaeger/jaeger-v1.48.yml`

5. **#{ACR_NAME}#.azurecr.io/jaegertracing/all-in-one:1.35**
   - Component: Jaeger All-in-One
   - File: `addons/monitoring/jaeger/jaeger-v1.35.yml`

### Grafana (Dashboards & Visualization)

6. **#{ACR_NAME}#.azurecr.io/istio/grafana:10.4.0**
   - Component: Grafana Dashboard
   - File: `addons/monitoring/grafana/grafana-v10.4.0.yml`

7. **#{ACR_NAME}#.azurecr.io/istio/grafana:9.0.1**
   - Component: Grafana Dashboard
   - File: `addons/monitoring/grafana/grafana-v9.0.1.yml`

### Kubernetes Dashboard

8. **#{ACR_NAME}#.azurecr.io/11614-et-oss/dashboard:v2.7.0**
   - Component: Kubernetes Dashboard UI
   - File: `addons/monitoring/dashboard/kubernetes-dashboard-v2.7.0.yaml`

9. **#{ACR_NAME}#.azurecr.io/11614-et-oss/metrics-scraper:v1.0.9**
   - Component: Dashboard Metrics Scraper
   - File: `addons/monitoring/dashboard/kubernetes-dashboard-v2.7.0.yaml`

### Kube State Metrics

10. **#{ACR_NAME}#.azurecr.io/kravikumar/kube-state-metrics:v2.12.0**
    - Component: Kubernetes State Metrics Exporter
    - File: `addons/monitoring/kube-state-metrics/v2.12.0/deployment.yaml`

---

## 3. MetLife DTR (dtr.metlife.com)

### Istio Components - Deployed Versions

#### Istio v1.13.1
- **docker.io/istio/proxyv2:1.13.1**
- **docker.io/istio/pilot:1.13.1**

#### Istio v1.17.2
- **dtr.metlife.com/istio/proxyv2:1.17.2**
- **dtr.metlife.com/istio/pilot:1.17.2**

#### Istio v1.17.5
- **dtr.metlife.com/istio/proxyv2:1.17.5**
- **dtr.metlife.com/istio/pilot:1.17.5**

#### Istio v1.17.6
- **docker.io/istio/proxyv2:1.17.6**
- **docker.io/istio/pilot:1.17.6**

#### Istio v1.19.0
- **dtr.metlife.com/istio/proxyv2:1.19.0**
- **dtr.metlife.com/istio/pilot:1.19.0**

#### Istio v1.20.5
- **dtr.metlife.com/istio/proxyv2:1.20.5**
- **dtr.metlife.com/istio/pilot:1.20.5**

#### Istio v1.21.2
- **dtr.metlife.com/istio/proxyv2:1.19.0** (Note: Uses v1.19.0)
- **dtr.metlife.com/istio/pilot:1.19.0** (Note: Uses v1.19.0)

#### Istio v1.22.5
- **dtr.metlife.com/istio/proxyv2:1.23.0** (Note: Uses v1.23.0)
- **dtr.metlife.com/istio/pilot:1.23.0** (Note: Uses v1.23.0)

---

## 4. JFrog Artifactory (jfrog-artifactory.metlife.com)

### Istio Images via Pipeline
- **Hub**: jfrog-artifactory.metlife.com/docker-metlife-base-images-virtual/istio
- **File**: `aks-config-pipeline/istio/install/istio.aks.yml`
- **Components**: Dynamically pulled based on Istio version during installation

---

## 5. Other Images (Not from ACR)

### BusyBox (Helper Container)
- **busybox:1.28**
  - Purpose: Init container for various setup tasks
  - Found in multiple Istio operator manifests

---

## Summary Statistics

### Total Image Categories:
1. **Istio Core Components**: 3 types × 8 versions × 2 variants = 48 images
2. **Monitoring Components**: 10 unique images with various versions
3. **DTR/JFrog Images**: Additional registry alternatives

### Deployment Environments:
- **ACR Primary**: acr14031e1dv01.azurecr.io
- **ACR Variable**: #{ACR_NAME}#.azurecr.io (environment-specific)
- **DTR**: dtr.metlife.com
- **JFrog**: jfrog-artifactory.metlife.com

### Key Scripts:
- **Image Push Script**: `install-prereqs.ps1`
- **Installation Pipeline**: `aks-config-pipeline/istio-installation.yml`

---

## Notes:
1. The `#{ACR_NAME}#` is a pipeline variable that gets substituted during deployment
2. Some Istio versions in manifests reference docker.io directly (v1.13.1, v1.17.6) while others use dtr.metlife.com
3. Distroless images are available for enhanced security with smaller attack surface
4. All images are maintained across multiple versions for backward compatibility

---

**Generated**: November 25, 2025
**Repository**: 14031_digital-soh-istio

