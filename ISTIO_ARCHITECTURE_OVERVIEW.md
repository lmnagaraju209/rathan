# Digital SOH Istio Project - Complete Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Project Structure](#project-structure)
4. [Key Components](#key-components)
5. [Supported Istio Versions](#supported-istio-versions)
6. [Installation Process](#installation-process)
7. [Deployment Scenarios](#deployment-scenarios)
8. [Monitoring Stack](#monitoring-stack)
9. [CI/CD Pipeline](#cicd-pipeline)
10. [Cluster Management](#cluster-management)
11. [Configuration Files](#configuration-files)
12. [Prerequisites](#prerequisites)
13. [Usage Examples](#usage-examples)
14. [Troubleshooting](#troubleshooting)

---

## Project Overview

### What is this Project?

This project is an **Istio Service Mesh Deployment and Management System** for **Azure Kubernetes Service (AKS)** clusters. It provides automated installation, upgrade, and management of Istio service mesh infrastructure along with a comprehensive monitoring stack.

### Purpose

The project serves to:
- **Deploy Istio service mesh** on AKS clusters in multiple environments (Dev, Int, QA, Prod, DR)
- **Manage Istio lifecycle** including installation, upgrades, and rollbacks
- **Automate monitoring tools** deployment (Prometheus, Grafana, Kiali, Jaeger)
- **Handle container image management** for MetLife's private Docker registry (DTR)
- **Provide cluster management utilities** for starting/stopping clusters

### Target Audience

This project is designed for:
- **DevOps Engineers** managing AKS infrastructure
- **Platform Engineers** responsible for service mesh operations
- **SREs** monitoring and maintaining Kubernetes clusters
- **Application Teams** deploying services on the platform

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Azure Kubernetes Service                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Istio Service Mesh Layer                │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │  │
│  │  │  Pilot     │  │  Ingress   │  │  Egress    │    │  │
│  │  │ (Control   │  │  Gateway   │  │  Gateway   │    │  │
│  │  │  Plane)    │  └────────────┘  └────────────┘    │  │
│  │  └────────────┘                                      │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Monitoring & Observability Stack           │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │  │
│  │  │Prometheus│ │ Grafana  │ │  Kiali   │ │ Jaeger │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          Application Workloads with Sidecars         │  │
│  │  ┌───────────────┐  ┌───────────────┐              │  │
│  │  │ App Pod + Envoy│  │ App Pod + Envoy│              │  │
│  │  └───────────────┘  └───────────────┘              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │                        │
         ▼                        ▼
   Load Balancer          Private Registry
   (Internal Azure LB)    (dtr.metlife.com / JFrog)
```

### Key Architectural Decisions

1. **Internal Load Balancer**: Uses Azure internal load balancer for ingress traffic
2. **Private Registry**: All images pulled from MetLife DTR or JFrog Artifactory
3. **Multi-Environment Support**: Supports dev, int, qa, uat, prod, and dr environments
4. **Operator-Based Deployment**: Uses Istio Operator for declarative configuration
5. **Pod Disruption Budgets**: Ensures high availability during cluster operations

---

## Project Structure

```
14031_digital-soh-istio/
│
├── istio/                              # Istio version-specific configurations
│   ├── v1.13.1/
│   ├── v1.17.2/
│   ├── v1.17.5/
│   ├── v1.17.6/
│   ├── v1.19.0/
│   ├── v1.20.5/
│   ├── v1.21.2/
│   ├── v1.22.5/
│   │   └── istio-operator-manifest.yaml   # Generated manifest
│   ├── istio-operator-values.yaml         # Base Istio configuration
│   └── howto-generate-manifest.ps1        # Manifest generation script
│
├── aks-config-pipeline/                # Azure DevOps pipeline configs
│   ├── istio-installation.yml          # Main pipeline definition
│   ├── variables.yml                   # Pipeline variables
│   ├── istio/
│   │   ├── install/
│   │   │   ├── install.sh              # Linux installation script
│   │   │   ├── download-linux.sh       # Istio CLI download
│   │   │   ├── uninstall-istio.sh      # Uninstall script
│   │   │   └── istio.aks.yml           # AKS-specific Istio config
│   │   └── pod-disruption-budget/
│   │       └── istio-pdb.yml           # PDB for high availability
│   └── taskgroup/
│       └── taskgroup-akscredential.yaml # AKS authentication
│
├── addons/                             # Monitoring and observability tools
│   ├── monitoring/
│   │   ├── prometheus/                 # Metrics collection
│   │   │   ├── prometheus-v2.41.0.yml
│   │   │   └── prometheus-v15.9.0.yml
│   │   ├── grafana/                    # Dashboards
│   │   │   ├── grafana-v9.0.1.yml
│   │   │   ├── grafana-v10.4.0.yml
│   │   │   └── dashboard/
│   │   │       └── kubernetes-pod-overview.json
│   │   ├── kiali/                      # Service mesh visualization
│   │   │   ├── kiali-v1.63.1.yml
│   │   │   └── kiali-v1.83.0.yml
│   │   ├── jaeger/                     # Distributed tracing
│   │   │   ├── jaeger-v1.35.yml
│   │   │   └── jaeger-v1.48.yml
│   │   ├── dashboard/
│   │   │   └── kubernetes-dashboard-v2.7.0.yaml
│   │   ├── kube-state-metrics/         # Kubernetes metrics exporter
│   │   │   └── v2.12.0/
│   │   ├── gateway/                    # Istio gateway configs
│   │   │   ├── gateway-routes.yaml
│   │   │   └── gateway-tls-secret.yaml
│   │   ├── install.ps1                 # Monitoring installation
│   │   └── install-secrets.ps1
│   └── cluster-roles/
│       └── service-discovery-client.yaml
│
├── aksUpgrade/                         # AKS upgrade utilities
│   ├── clean-up-pdb.ps1               # Clean up PDBs before upgrade
│   └── clean-up-pods.ps1              # Clean up pods
│
├── cluster-mgmt/                       # Cluster lifecycle management
│   ├── cluster-mgmt-common.ps1        # Common functions
│   ├── start-cluster.ps1              # Start AKS cluster
│   └── stop-cluster.ps1               # Stop AKS cluster
│
├── install.ps1                         # Main installation orchestrator (Windows)
├── install-common.ps1                  # Common PowerShell functions
├── install-prereqs.ps1                 # Pre-installation tasks
├── install-postreqs.ps1                # Post-installation tasks
├── pipeline-prereqs.ps1                # Pipeline-specific prerequisites
└── README.md                           # Quick start guide
```

---

## Key Components

### 1. **Istio Control Plane**
- **Pilot (Istiod)**: Service discovery, traffic management, and configuration
- **Configuration**: 
  - Min replicas: 3, Max replicas: 10
  - CPU: 500m-1000m, Memory: 2048Mi-2560Mi
  - Horizontal Pod Autoscaling enabled

### 2. **Istio Gateways**

#### Ingress Gateway
- Handles incoming traffic to the mesh
- Configured with Azure internal load balancer
- Static IP assignment per environment
- Min replicas: 3, Max replicas: 10
- CPU: 100m-1000m, Memory: 128Mi-1024Mi

#### Egress Gateway
- Controls outbound traffic from the mesh
- Same resource configuration as ingress
- Can be disabled in DR environments

### 3. **Traffic Policy**
- **Mode**: `ALLOW_ANY` (configurable to `REGISTRY_ONLY`)
- Enables access to external services
- Can be restricted for security compliance

### 4. **Monitoring Stack Components**

| Tool | Version | Purpose |
|------|---------|---------|
| **Prometheus** | v2.41.0, v15.9.0 | Metrics collection and storage |
| **Grafana** | v9.0.1, v10.4.0 | Dashboard visualization |
| **Kiali** | v1.63.1, v1.83.0 | Service mesh topology and health |
| **Jaeger** | v1.35, v1.48 | Distributed tracing |
| **Kube-state-metrics** | v2.12.0 | Kubernetes object metrics |
| **Kubernetes Dashboard** | v2.7.0 | Cluster resource visualization |

---

## Supported Istio Versions

The project supports the following Istio versions:
- **1.13.1** (Legacy)
- **1.17.2**
- **1.17.5**
- **1.17.6**
- **1.19.0**
- **1.20.5**
- **1.21.2**
- **1.22.5** (Latest)

### Version Upgrade Policy

The system automatically determines the deployment mode:

1. **INSTALL**: Fresh installation (no existing Istio)
2. **UPGRADE**: Incremental upgrade (same major version, 0-1 minor version difference)
   - Example: 1.17.x → 1.17.y or 1.17.x → 1.18.x
3. **REINSTALL**: Breaking change upgrade (requires uninstall and reinstall)
   - Example: 1.13.x → 1.17.x

---

## Installation Process

### Installation Workflow

```
┌─────────────────────────────────────────────────────────────┐
│  1. Pull Docker Images from docker.io                       │
│     - istio/operator                                         │
│     - istio/pilot                                            │
│     - istio/proxyv2                                         │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Push Images to MetLife DTR/JFrog                        │
│     - dtr.metlife.com/istio/operator:version                │
│     - Regular and distroless variants                       │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Download Istio CLI (istioctl)                           │
│     - Extract binaries                                       │
│     - Set permissions                                        │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Generate Istio Manifest                                 │
│     - Use istio-operator-values.yaml                        │
│     - Create version-specific manifest                      │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Pre-Installation Checks                                 │
│     - istioctl x precheck                                   │
│     - Validate cluster connectivity                         │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Deploy Istio (Install/Upgrade/Reinstall)               │
│     - Create istio-system namespace                         │
│     - Apply operator configuration                          │
│     - Wait for rollout completion                           │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Verify Installation                                     │
│     - istioctl verify-install                               │
│     - Check LoadBalancer IP assignment                      │
│     - Validate gateway status                               │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Apply Pod Disruption Budgets                            │
│     - Ensure high availability                              │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  9. Restart Application Deployments                         │
│     - Inject Envoy sidecars                                 │
│     - Exclude system namespaces                             │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  10. Post-Installation Checks                               │
│      - istioctl analyze                                     │
│      - Verify proxy status                                  │
└─────────────────────────────────────────────────────────────┘
```

### PowerShell Installation Functions

#### 1. **Istio-DownloadVersion**
Downloads Istio CLI for a specific version and generates manifest

```powershell
Istio-DownloadVersion -version "1.22.5"
```

#### 2. **Istio-VersionInfo**
Checks installed Istio version and determines upgrade mode

```powershell
Istio-VersionInfo -version "1.22.5" -path "./istio/v1.22.5/istioctl_v1.22.5"
```

#### 3. **Istio-InstallClientVersion**
Fresh installation of Istio

```powershell
Istio-InstallClientVersion -version "1.22.5" -ipaddr "10.213.147.60" -path "./istio/v1.22.5/istioctl_v1.22.5" -values "./istio/v1.22.5/istio-operator-values.yaml"
```

#### 4. **Istio-UpgradeRemoteVersion**
Incremental upgrade of existing Istio

```powershell
Istio-UpgradeRemoteVersion -version "1.22.5" -ipaddr "10.213.147.60" -path "./istio/v1.22.5/istioctl_v1.22.5" -values "./istio/v1.22.5/istio-operator-values.yaml"
```

#### 5. **Istio-UninstallRemoteVersion**
Uninstall existing Istio installation

```powershell
Istio-UninstallRemoteVersion -version "1.17.6" -path "./istio/v1.17.6/istioctl_v1.17.6" -manifest "./istio/v1.17.6/istio-operator-manifest.yaml"
```

---

## Deployment Scenarios

### Scenario 1: Fresh Installation (Dev Environment)

**Command:**
```powershell
.\install.ps1 -version "1.22.5" -ipaddr "10.213.147.60" -helm-value-prefix "dev"
```

**What Happens:**
1. No existing Istio detected → INSTALL mode
2. Downloads Istio 1.22.5 binaries
3. Deploys control plane and gateways
4. Assigns load balancer IP: 10.213.147.60
5. Restarts all application deployments

### Scenario 2: Incremental Upgrade (QA: 1.21.2 → 1.22.5)

**Command:**
```powershell
.\install.ps1 -version "1.22.5" -ipaddr "10.213.20.188" -helm-value-prefix "qa"
```

**What Happens:**
1. Detects existing 1.21.2 → UPGRADE mode
2. Downloads Istio 1.22.5 binaries
3. Performs in-place upgrade
4. Preserves existing configurations
5. Rolling restart of control plane

### Scenario 3: Breaking Upgrade (Int: 1.13.1 → 1.22.5)

**Command:**
```powershell
.\install.ps1 -version "1.22.5" -ipaddr "10.213.151.28" -helm-value-prefix "int"
```

**What Happens:**
1. Detects version jump → REINSTALL mode
2. Downloads old version (1.13.1) binaries
3. Uninstalls 1.13.1 using old istioctl
4. Cleans up resources
5. Installs fresh 1.22.5
6. Restarts all deployments

### Scenario 4: Disaster Recovery Environment

**Command:**
```powershell
.\install.ps1 -version "1.22.5" -ipaddr "10.213.20.221" -helm-value-prefix "dr"
```

**Special Handling:**
1. Installs Istio normally
2. **Stops ingress/egress gateways** (replica count = 0)
3. DR cluster remains in standby mode
4. Traffic doesn't flow through gateways

### Scenario 5: Dry Run (Testing)

**Command:**
```powershell
.\install.ps1 -version "1.22.5" -ipaddr "10.213.147.60" -helm-value-prefix "dev" -dry-run
```

**What Happens:**
1. Simulates all operations
2. No actual changes to cluster
3. Useful for validation

---

## Monitoring Stack

### Deployment

**Monitoring is deployed via:**
```powershell
.\addons\monitoring\install.ps1
```

### Components Deployed

#### 1. **Prometheus**
- **Purpose**: Scrapes and stores metrics
- **Namespace**: `istio-system`
- **Metrics Sources**:
  - Istio control plane
  - Envoy proxies
  - Kubernetes API server
  - Application pods

#### 2. **Grafana**
- **Purpose**: Visualization dashboards
- **Pre-configured Dashboards**:
  - Istio mesh overview
  - Istio service metrics
  - Istio workload metrics
  - Kubernetes pod overview
- **Access**: Via Istio ingress gateway

#### 3. **Kiali**
- **Purpose**: Service mesh observability
- **Features**:
  - Service graph visualization
  - Traffic flow analysis
  - Configuration validation
  - Health checks
- **Integration**: Uses Prometheus as data source

#### 4. **Jaeger**
- **Purpose**: Distributed tracing
- **Trace Collection**:
  - Envoy sidecars send traces
  - End-to-end request tracking
  - Latency analysis
  - Error detection

#### 5. **Kubernetes Dashboard**
- **Purpose**: Cluster resource management
- **Access URL**: `https://<monitoring-host>/kubernetes-dashboard/`

### Access Configuration

Monitoring tools are exposed via Istio Gateway with the following routes:
- Kubernetes Dashboard: `/kubernetes-dashboard/`
- Prometheus (if enabled): `/prometheus/`
- Grafana (if enabled): `/grafana/`
- Kiali (if enabled): `/kiali/`
- Jaeger (if enabled): `/jaeger/`

---

## CI/CD Pipeline

### Azure DevOps Pipeline

**File**: `aks-config-pipeline/istio-installation.yml`

### Pipeline Parameters

| Parameter | Type | Values | Description |
|-----------|------|--------|-------------|
| **action** | string | install, uninstall | Operation to perform |
| **aks** | string | Cluster names | Target AKS cluster |
| **ingress** | string | IP addresses | Ingress load balancer IP |
| **rg** | string | Resource groups | Azure resource group |
| **tag** | string | Version numbers | Istio version to deploy |
| **environment** | string | dev, int, qa, prod | Target environment |
| **AZ_SERVICE_CONNECTION** | string | Service principals | Azure authentication |

### Pipeline Stages

#### Stage 1: Pre-screening
- Display all parameters
- List directory contents
- Validate configuration

#### Stage 2: Tool Installation
- Install kubectl (v1.28.3)
- Install Helm (v3.14.0)
- Download Istio CLI for specified version

#### Stage 3: Authentication
- Connect to Azure using service principal
- Get AKS credentials
- Set KUBECONFIG

#### Stage 4: Namespace & Secret Setup
- Create/recreate `istio-system` namespace
- Create JFrog image pull secrets in all namespaces
- Credentials from JFROG_BASE_IMAGE_PULL variable group

#### Stage 5: Istio Installation/Uninstallation
**For Install:**
- Execute `install.sh` with ingress IP
- Apply pod disruption budgets
- Restart deployments in application namespaces

**For Uninstall:**
- Execute `uninstall_istio.sh`
- Clean up resources

### Example Pipeline Execution

**Deploying to Dev:**
```yaml
Parameters:
  action: install
  aks: AZAMERES01KUB14031NP01
  ingress: 10.213.147.60
  rg: RG_14031_DigitalSOH_DEV
  tag: 1.22.5
  environment: dev
  AZ_SERVICE_CONNECTION: SPAZDO14031NP01
```

---

## Cluster Management

### Starting a Cluster

**Script**: `cluster-mgmt/start-cluster.ps1`

**Purpose**: Start a stopped AKS cluster

**Usage:**
```powershell
.\cluster-mgmt\start-cluster.ps1 -ClusterName "AZAMERES01KUB14031NP01" -ResourceGroup "RG_14031_DigitalSOH_DEV"
```

### Stopping a Cluster

**Script**: `cluster-mgmt/stop-cluster.ps1`

**Purpose**: Stop a running AKS cluster (cost savings)

**Usage:**
```powershell
.\cluster-mgmt\stop-cluster.ps1 -ClusterName "AZAMERES01KUB14031NP01" -ResourceGroup "RG_14031_DigitalSOH_DEV"
```

### AKS Upgrade Utilities

#### Clean Up Pod Disruption Budgets

**Script**: `aksUpgrade/clean-up-pdb.ps1`

**Purpose**: Remove PDBs before AKS upgrade to prevent blocking

**Why**: PDBs can prevent node draining during cluster upgrades

#### Clean Up Pods

**Script**: `aksUpgrade/clean-up-pods.ps1`

**Purpose**: Force cleanup of stuck pods

---

## Configuration Files

### 1. **istio-operator-values.yaml**

Main configuration for Istio deployment

**Key Settings:**

```yaml
hub: dtr.metlife.com/istio        # Private registry
profile: default                    # Istio profile

meshConfig:
  accessLogFile: /dev/stdout        # Enable access logs
  enableTracing: true               # Distributed tracing
  outboundTrafficPolicy:
    mode: ALLOW_ANY                 # Allow external access

components:
  pilot:
    k8s:
      resources:
        requests: { cpu: 500m, memory: 2048Mi }
        limits: { cpu: 1000m, memory: 2560Mi }
      hpaSpec:
        minReplicas: 3
        maxReplicas: 10
  
  ingressGateways:
    - name: istio-ingressgateway
      k8s:
        service:
          type: LoadBalancer
          loadBalancerIP: #{AKS_INGRESS_IPADDR}#  # Replaced at runtime
        serviceAnnotations:
          service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

### 2. **istio.aks.yml**

AKS-specific Istio configuration used in Linux installation

**Features:**
- Template placeholders: `@ISTIO_INGRESS_IP`
- Replaced by `sed` during installation
- Image pull secret configuration

### 3. **istio-pdb.yml**

Pod Disruption Budget for high availability

**Prevents:**
- Too many pods going down during:
  - Node upgrades
  - Cluster maintenance
  - Voluntary disruptions

---

## Prerequisites

### System Requirements

#### For Windows/PowerShell Scripts:
- PowerShell 5.1 or later
- Azure CLI installed
- kubectl CLI installed
- jq (JSON processor)
- curl or Invoke-WebRequest
- tar utility

#### For Linux/Bash Scripts:
- Bash 4.0+
- kubectl CLI
- curl
- sed
- nc (netcat) for health checks

### Azure Requirements

1. **Service Principal** with permissions:
   - AKS cluster admin access
   - Azure resource read/write

2. **AKS Cluster**:
   - Kubernetes version: 1.24+
   - Node pools with sufficient resources
   - Internal load balancer support

3. **Networking**:
   - Pre-allocated static IPs for ingress
   - Private network connectivity

### Container Registry Access

1. **MetLife DTR** (`dtr.metlife.com`):
   - Read access for pulling images
   - Write access for pushing images

2. **JFrog Artifactory** (`jfrog-artifactory.metlife.com`):
   - Credentials stored in Azure DevOps variable group
   - Image pull secrets created automatically

### Permissions Required

- **Namespace**: Create, delete, list
- **Deployments**: Create, update, delete, rollout restart
- **Services**: Create, update, delete, list
- **Secrets**: Create, list
- **ConfigMaps**: Create, update, list
- **CustomResourceDefinitions**: Create, list
- **ClusterRoles/ClusterRoleBindings**: Create, list

---

## Usage Examples

### Example 1: Install Istio on Dev Cluster

```powershell
# Connect to Azure
az login
az account set --subscription "Digital-SOH-Dev"

# Get AKS credentials
az aks get-credentials --name AZAMERES01KUB14031NP01 --resource-group RG_14031_DigitalSOH_DEV

# Install Istio
.\install.ps1 -version "1.22.5" -ipaddr "10.213.147.60" -helm-value-prefix "dev"
```

### Example 2: Upgrade Production Cluster

```powershell
# Dry run first
.\install.ps1 -version "1.22.5" -ipaddr "10.213.20.221" -helm-value-prefix "prod" -dry-run

# Execute actual upgrade
.\install.ps1 -version "1.22.5" -ipaddr "10.213.20.221" -helm-value-prefix "prod"
```

### Example 3: Deploy Monitoring Stack

```powershell
# Set environment variables
$env:MONITORING_HOST_NAME = "monitoring.digital.metlife.com"

# Install monitoring tools
.\addons\monitoring\install.ps1
```

### Example 4: Generate Istio Manifest

```powershell
# Generate manifest for version 1.22.5
.\istio\howto-generate-manifest.ps1 -version "1.22.5"

# Commit to repository
git add istio/v1.22.5/istio-operator-manifest.yaml
git commit -m "Add Istio 1.22.5 manifest"
```

### Example 5: Pipeline-Based Deployment

Trigger via Azure DevOps UI:
1. Go to Pipelines → SOH-istio-deployment
2. Click "Run pipeline"
3. Select parameters:
   - Action: install
   - AKS: AZAMERES01KUB14031QA01
   - Ingress: 10.213.20.188
   - RG: RG_14031_DigitalSOH_QA
   - Tag: 1.22.5
   - Environment: qa
4. Click "Run"

---

## Troubleshooting

### Common Issues

#### Issue 1: Image Pull Failures

**Symptoms:**
```
ImagePullBackOff on istio-pilot pod
```

**Solutions:**
1. Check image pull secrets:
   ```bash
   kubectl get secrets -n istio-system | grep jfrogauth
   ```

2. Recreate secret:
   ```bash
   kubectl create secret docker-registry jfrogauth \
     --docker-server=jfrog-artifactory.metlife.com \
     --docker-username=<user> \
     --docker-password=<pass> \
     --namespace=istio-system
   ```

#### Issue 2: Load Balancer IP Not Assigned

**Symptoms:**
```
Ingress gateway service stuck in Pending state
```

**Solutions:**
1. Check IP availability in Azure:
   ```bash
   az network vnet subnet show --vnet-name <vnet> --name <subnet> --resource-group <rg>
   ```

2. Verify service annotation:
   ```bash
   kubectl get svc istio-ingressgateway -n istio-system -o yaml | grep loadBalancerIP
   ```

#### Issue 3: Upgrade Fails

**Symptoms:**
```
Error: cannot upgrade from 1.13.1 to 1.22.5
```

**Solutions:**
1. Use REINSTALL mode (automatically detected)
2. Or manually uninstall and reinstall:
   ```powershell
   .\install.ps1 -version "1.13.1" -ipaddr "<ip>" -helm-value-prefix "dev" -dry-run
   # Then install new version
   .\install.ps1 -version "1.22.5" -ipaddr "<ip>" -helm-value-prefix "dev"
   ```

#### Issue 4: Pods Not Getting Sidecars

**Symptoms:**
```
Application pods don't have istio-proxy container
```

**Solutions:**
1. Enable sidecar injection on namespace:
   ```bash
   kubectl label namespace <namespace> istio-injection=enabled
   ```

2. Restart deployments:
   ```bash
   kubectl rollout restart deployment -n <namespace>
   ```

3. Check injection:
   ```bash
   istioctl x check-inject -n <namespace> deployment/<deployment-name>
   ```

#### Issue 5: Control Plane Not Ready

**Symptoms:**
```
istio-pilot pods in CrashLoopBackOff
```

**Solutions:**
1. Check logs:
   ```bash
   kubectl logs -n istio-system deployment/istiod
   ```

2. Verify resources:
   ```bash
   kubectl describe pod -n istio-system -l app=istiod
   ```

3. Check node resources:
   ```bash
   kubectl top nodes
   ```

### Diagnostic Commands

#### Check Istio Version
```bash
istioctl version -o json
```

#### Pre-Installation Check
```bash
istioctl x precheck
```

#### Analyze Configuration
```bash
istioctl analyze -n <namespace>
```

#### Check Proxy Status
```bash
istioctl proxy-status
```

#### Get Bug Report
```bash
istioctl bug-report --duration 10m -o yaml
```

#### Check Mesh Configuration
```bash
kubectl -n istio-system get istiooperator -o yaml
```

### Logging and Debugging

#### Enable Debug Logging
Add to `istio-operator-values.yaml`:
```yaml
values:
  global:
    logging:
      level: "default:debug"
```

#### Access Logs
```bash
# Control plane logs
kubectl logs -n istio-system deployment/istiod

# Ingress gateway logs
kubectl logs -n istio-system deployment/istio-ingressgateway

# Application sidecar logs
kubectl logs <pod-name> -c istio-proxy -n <namespace>
```

---

## Environment-Specific Details

### Cluster Mapping

| Environment | Cluster Name | Resource Group | Ingress IP | Service Connection |
|-------------|--------------|----------------|------------|-------------------|
| **Dev** | AZAMERES01KUB14031NP01 | RG_14031_DigitalSOH_DEV | 10.213.147.60 | SPAZDO14031NP01 |
| **Int** | AZAMERES01KUB14031NP02 | RG_14031_DigitalSOH_INT | 10.213.151.28 | SPAZDO14031NP01 |
| **QA** | AZAMERES01KUB14031QA01 | RG_14031_DigitalSOH_QA | 10.213.20.188 | SPAZDO14031QA01 |
| **Prod** | AZAMERES01KUB14031PR01 | RG_14031_DigitalSOH_PROD | 10.213.20.221 | SPAZDO14031PR01 |

---

## Best Practices

### 1. Version Management
- Always test in dev/int before prod
- Use dry-run mode for validation
- Keep manifests in version control
- Document version upgrade paths

### 2. High Availability
- Maintain PDBs for critical components
- Use HPA for control plane and gateways
- Monitor resource utilization
- Plan for node failures

### 3. Security
- Use private container registries
- Enable mTLS between services
- Restrict egress traffic (REGISTRY_ONLY mode)
- Rotate credentials regularly

### 4. Monitoring
- Set up alerting in Prometheus
- Monitor control plane health
- Track sidecar injection rate
- Review access logs regularly

### 5. Disaster Recovery
- Keep DR cluster in sync
- Test failover procedures
- Document rollback steps
- Maintain backup configurations

---

## Additional Resources

### Official Documentation
- Istio Documentation: https://istio.io/docs
- Azure AKS Documentation: https://docs.microsoft.com/azure/aks
- Kubernetes Documentation: https://kubernetes.io/docs

### Internal Resources
- MetLife DevOps Portal: (internal link)
- AKS Cluster Runbooks: (internal link)
- Incident Response Procedures: (internal link)

### Validation Commands Reference

```bash
# Check Istio installation
kubectl -n istio-system get IstioOperator -o yaml

# Get Istio version details
istioctl version -o json

# Describe default revision
istioctl x revision describe default -o yaml

# Pre-flight check
istioctl x precheck

# Check specific namespace
istioctl x precheck -n <namespace>

# Generate bug report
istioctl bug-report --duration 10m -o yaml

# Analyze namespace configuration
istioctl analyze -n <namespace> -o yaml

# Check sidecar injection
istioctl x check-inject -n <namespace> deployment/<deployment-name>
istioctl x check-inject -n <namespace> -l app=<app-label>

# Proxy status
istioctl proxy-status -n <namespace>
istioctl proxy-status -n <namespace> deployment/<deployment-name>
istioctl proxy-status -n <namespace> -l app=<app-label>
```

---

## Summary

This project provides a **comprehensive, production-ready solution** for deploying and managing Istio service mesh on Azure Kubernetes Service. It includes:

✅ **Automated Installation**: PowerShell and Bash scripts for multiple scenarios
✅ **Version Management**: Support for multiple Istio versions with intelligent upgrade logic
✅ **CI/CD Integration**: Azure DevOps pipeline for automated deployments
✅ **Monitoring Stack**: Pre-configured Prometheus, Grafana, Kiali, and Jaeger
✅ **High Availability**: Pod Disruption Budgets and autoscaling
✅ **Multi-Environment**: Support for dev, int, qa, uat, prod, and DR environments
✅ **Enterprise Features**: Private registry integration, internal load balancers
✅ **Operational Tools**: Cluster management, upgrade utilities, diagnostic commands

The project is designed for **enterprise-grade deployments** with emphasis on **reliability, security, and operational excellence**.

---

**Document Version**: 1.0
**Last Updated**: November 25, 2025
**Maintained By**: Digital SOH Platform Team

