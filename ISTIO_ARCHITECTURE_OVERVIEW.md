# Digital SOH Istio - Architecture Overview

## System Architecture

### High-Level Component View

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           Azure Cloud Environment                           │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    Azure Kubernetes Service (AKS)                     │ │
│  │                                                                       │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │                    istio-system Namespace                    │  │ │
│  │  │                                                              │  │ │
│  │  │  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │  │ │
│  │  │  │  Istiod        │  │  Istio Ingress │  │ Istio Egress │  │  │ │
│  │  │  │  (Control      │  │  Gateway       │  │ Gateway      │  │  │ │
│  │  │  │   Plane)       │  │                │  │              │  │  │ │
│  │  │  │  - Pilot       │  │  - LB: Internal│  │  - Control   │  │  │ │
│  │  │  │  - Citadel     │  │  - Static IP   │  │    Egress    │  │  │ │
│  │  │  │  - Galley      │  │  - HPA: 3-10   │  │  - HPA: 3-10 │  │  │ │
│  │  │  │  - HPA: 3-10   │  │                │  │              │  │  │ │
│  │  │  └────────────────┘  └────────────────┘  └──────────────┘  │  │ │
│  │  │                                                              │  │ │
│  │  │  ┌──────────────────────────────────────────────────────┐  │  │ │
│  │  │  │          Monitoring & Observability Tools           │  │  │ │
│  │  │  │  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌─────────┐ │  │  │ │
│  │  │  │  │Prometheus│ │ Grafana  │ │ Kiali  │ │ Jaeger  │ │  │  │ │
│  │  │  │  └──────────┘ └──────────┘ └────────┘ └─────────┘ │  │  │ │
│  │  │  └──────────────────────────────────────────────────────┘  │  │ │
│  │  └──────────────────────────────────────────────────────────────┘  │ │
│  │                                                                       │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │                Application Namespaces                        │  │ │
│  │  │  (istio-injection=enabled)                                   │  │ │
│  │  │                                                              │  │ │
│  │  │  ┌─────────────────────┐      ┌─────────────────────┐      │  │ │
│  │  │  │   Application Pod   │      │   Application Pod   │      │  │ │
│  │  │  │  ┌──────────────┐  │      │  ┌──────────────┐  │      │  │ │
│  │  │  │  │ App Container│  │      │  │ App Container│  │      │  │ │
│  │  │  │  └──────────────┘  │      │  └──────────────┘  │      │  │ │
│  │  │  │  ┌──────────────┐  │      │  ┌──────────────┐  │      │  │ │
│  │  │  │  │Envoy Sidecar │  │      │  │Envoy Sidecar │  │      │  │ │
│  │  │  │  │  (Proxy)     │  │      │  │  (Proxy)     │  │      │  │ │
│  │  │  │  └──────────────┘  │      │  └──────────────┘  │      │  │ │
│  │  │  └─────────────────────┘      └─────────────────────┘      │  │ │
│  │  └──────────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────┐              │
│  │         Azure Internal Load Balancer                     │              │
│  │         - Static IP per Environment                      │              │
│  │         - Routes to Istio Ingress Gateway                │              │
│  └──────────────────────────────────────────────────────────┘              │
└────────────────────────────────────────────────────────────────────────────┘
                             │                │
                             ▼                ▼
                    ┌─────────────────┐  ┌─────────────────┐
                    │  MetLife DTR    │  │ JFrog Artifactory│
                    │  dtr.metlife.com│  │  jfrog-artifact..│
                    │                 │  │                  │
                    │  Istio Images:  │  │  Base Images     │
                    │  - operator     │  │  - Pull Secrets  │
                    │  - pilot        │  │                  │
                    │  - proxyv2      │  │                  │
                    └─────────────────┘  └─────────────────┘
```

---

## Data Flow Architecture

### Inbound Traffic Flow

```
External Client
      │
      ▼
┌─────────────────────┐
│  Azure Internal     │
│  Load Balancer      │
│  (Static IP)        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Istio Ingress Gateway                  │
│  - TLS Termination                      │
│  - Virtual Service Routing              │
│  - Gateway Rules                        │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Application Pod                        │
│  ┌────────────────────────────────────┐ │
│  │ Envoy Sidecar (Port 15001)        │ │
│  │  - mTLS                           │ │
│  │  - Traffic Management             │ │
│  │  - Telemetry Collection          │ │
│  └──────────┬─────────────────────────┘ │
│             ▼                            │
│  ┌────────────────────────────────────┐ │
│  │ Application Container              │ │
│  │  - Business Logic                 │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Outbound Traffic Flow

```
┌─────────────────────────────────────────┐
│  Application Pod                        │
│  ┌────────────────────────────────────┐ │
│  │ Application Container              │ │
│  │  - Makes External API Call        │ │
│  └──────────┬─────────────────────────┘ │
│             ▼                            │
│  ┌────────────────────────────────────┐ │
│  │ Envoy Sidecar                     │ │
│  │  - Intercepts Outbound Traffic   │ │
│  │  - Applies Policies              │ │
│  │  - Telemetry Collection          │ │
│  └──────────┬─────────────────────────┘ │
└─────────────┼─────────────────────────────┘
              │
              ▼
   ┌──────────────────────┐
   │ Istio Egress Gateway │
   │  - Centralized Exit  │
   │  - TLS Origination   │
   │  - Access Control    │
   └──────────┬───────────┘
              │
              ▼
   ┌──────────────────────┐
   │  External Services   │
   │  - APIs              │
   │  - Databases         │
   │  - Third-party       │
   └──────────────────────┘
```

### Service-to-Service Communication

```
Service A Pod                    Service B Pod
┌────────────────┐              ┌────────────────┐
│  App Container │              │  App Container │
└───────┬────────┘              └────────┬───────┘
        │                                 ▲
        ▼                                 │
┌────────────────┐              ┌────────────────┐
│ Envoy Sidecar  │─────mTLS────>│ Envoy Sidecar  │
│  - Load Balance│              │  - AuthN/AuthZ │
│  - Retry Logic │              │  - Rate Limit  │
│  - Circuit Brk │              │  - Metrics     │
└────────────────┘              └────────────────┘
        │                                 │
        └─────────Telemetry Data─────────┘
                        │
                        ▼
              ┌──────────────────┐
              │   Prometheus     │
              │   (Metrics)      │
              └──────────────────┘
```

---

## Deployment Pipeline Architecture

### CI/CD Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Azure DevOps Pipeline                           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
          ┌──────────────────┐  ┌──────────────────┐
          │ Manual Trigger   │  │ Scheduled Run    │
          │ - Parameters     │  │ - Automated      │
          └────────┬─────────┘  └─────┬────────────┘
                   └────────┬──────────┘
                            ▼
              ┌──────────────────────────┐
              │  Stage 1: Pre-screening  │
              │  - Validate Parameters   │
              │  - Check Connectivity    │
              └───────────┬──────────────┘
                          ▼
              ┌──────────────────────────┐
              │  Stage 2: Tool Setup     │
              │  - Install kubectl       │
              │  - Install Helm          │
              │  - Download istioctl     │
              └───────────┬──────────────┘
                          ▼
              ┌──────────────────────────┐
              │  Stage 3: Authentication │
              │  - Azure Login           │
              │  - Get AKS Credentials   │
              │  - Set KUBECONFIG        │
              └───────────┬──────────────┘
                          ▼
              ┌──────────────────────────┐
              │  Stage 4: Namespace Setup│
              │  - Create istio-system   │
              │  - Create Image Secrets  │
              └───────────┬──────────────┘
                          ▼
          ┌───────────────┴────────────────┐
          ▼                                ▼
┌──────────────────────┐       ┌──────────────────────┐
│ Install Action       │       │ Uninstall Action     │
│ - Run install.sh     │       │ - Run uninstall.sh   │
│ - Apply PDBs         │       │ - Cleanup Resources  │
│ - Restart Deployments│       └──────────────────────┘
└──────────┬───────────┘
           ▼
┌──────────────────────────┐
│  Stage 5: Verification   │
│  - Check Pod Status      │
│  - Verify Gateway IP     │
│  - Run istioctl analyze  │
└──────────────────────────┘
```

### Installation Process Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     Install.ps1 Entry Point                  │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
              ┌────────────────────────┐
              │ Istio-DownloadVersion  │
              │ - Download istioctl    │
              │ - Generate Manifest    │
              └──────────┬─────────────┘
                         ▼
              ┌────────────────────────┐
              │ Istio-VersionInfo      │
              │ - Check Installed Ver  │
              │ - Determine Mode       │
              └──────────┬─────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────────┐
│   INSTALL   │  │   UPGRADE   │  │   REINSTALL     │
│   Mode      │  │   Mode      │  │   Mode          │
└──────┬──────┘  └──────┬──────┘  └────────┬────────┘
       │                │                   │
       │                │         ┌─────────▼─────────┐
       │                │         │ Uninstall Old Ver │
       │                │         └─────────┬─────────┘
       │                │                   │
       └────────────────┴───────────────────┘
                        │
                        ▼
          ┌──────────────────────────┐
          │ Istio-InstallClientVer   │
          │ - Apply Operator Config  │
          │ - Wait for Rollout       │
          │ - Verify Installation    │
          └──────────┬───────────────┘
                     ▼
          ┌──────────────────────────┐
          │ Post-Installation        │
          │ - Apply PDBs             │
          │ - Restart Deployments    │
          │ - DR Mode Check          │
          └──────────────────────────┘
```

---

## Network Architecture

### Namespace and Network Segmentation

```
┌─────────────────────────────────────────────────────────────────┐
│                        AKS Cluster Network                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  istio-system Namespace                                │    │
│  │  - Istio Control Plane                                 │    │
│  │  - Ingress/Egress Gateways                            │    │
│  │  - Monitoring Stack                                    │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Application Namespace 1 (istio-injection=enabled)     │    │
│  │  - Frontend Services                                   │    │
│  │  - Each Pod has Envoy Sidecar                         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Application Namespace 2 (istio-injection=enabled)     │    │
│  │  - Backend Services                                    │    │
│  │  - Each Pod has Envoy Sidecar                         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Application Namespace N (istio-injection=enabled)     │    │
│  │  - Microservices                                       │    │
│  │  - Each Pod has Envoy Sidecar                         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  System Namespaces (NO istio-injection)                │    │
│  │  - kube-system, kube-public, default                  │    │
│  │  - gatekeeper-system, calico-system                   │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

Network Policies Applied by Istio:
  ✓ mTLS between all services in mesh
  ✓ AuthN/AuthZ at service level
  ✓ Traffic routing via VirtualServices
  ✓ Egress control via DestinationRules
```

---

## Monitoring and Observability Architecture

### Telemetry Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Workloads                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │ Service A  │  │ Service B  │  │ Service C  │                │
│  │ + Envoy    │  │ + Envoy    │  │ + Envoy    │                │
│  └──┬──┬──┬───┘  └──┬──┬──┬───┘  └──┬──┬──┬───┘                │
└─────┼──┼──┼─────────┼──┼──┼─────────┼──┼──┼────────────────────┘
      │  │  │         │  │  │         │  │  │
      │  │  └─────────┼──┼──┼─────────┘  │  │
      │  │   Metrics  │  │  │  Traces    │  │  Logs
      │  └────────────┼──┼──┼─────────────┘  │
      │               │  │  │                 │
      ▼               ▼  ▼  ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Prometheus  │  │    Jaeger    │  │  Log Aggr.   │
│              │  │              │  │  (stdout)    │
│  - Metrics   │  │  - Traces    │  │  - Access    │
│  - Scraping  │  │  - Spans     │  │  - Error     │
│  - Storage   │  │  - Timing    │  │  - Audit     │
└──────┬───────┘  └──────┬───────┘  └──────────────┘
       │                 │
       └────────┬────────┘
                ▼
       ┌─────────────────┐
       │    Grafana      │◄─────── User Interface
       │  - Dashboards   │
       │  - Alerts       │
       │  - Analytics    │
       └─────────────────┘
                │
                └──────────┐
                           ▼
                  ┌─────────────────┐
                  │     Kiali       │◄─────── Mesh Visualization
                  │  - Topology     │
                  │  - Health       │
                  │  - Config       │
                  └─────────────────┘
```

### Monitoring Stack Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        istio-system                              │
│                                                                  │
│  ┌────────────────────┐      ┌────────────────────┐            │
│  │    Prometheus      │      │      Grafana       │            │
│  │                    │      │                    │            │
│  │  Port: 9090        │◄─────┤  Port: 3000        │            │
│  │                    │      │                    │            │
│  │  Scrapes:          │      │  Datasources:      │            │
│  │  - Istiod          │      │  - Prometheus      │            │
│  │  - Envoy Proxies   │      │  - Jaeger          │            │
│  │  - K8s API         │      │                    │            │
│  │  - Applications    │      │  Dashboards:       │            │
│  │                    │      │  - Mesh Overview   │            │
│  │  Retention: 15d    │      │  - Service Metrics │            │
│  └────────────────────┘      │  - Workload Metrics│            │
│                              │  - Performance     │            │
│  ┌────────────────────┐      └────────────────────┘            │
│  │      Kiali         │                                        │
│  │                    │      ┌────────────────────┐            │
│  │  Port: 20001       │      │      Jaeger        │            │
│  │                    │      │                    │            │
│  │  Features:         │      │  Port: 16686       │            │
│  │  - Graph View      │      │                    │            │
│  │  - Traffic Flow    │      │  Collector: 14268  │            │
│  │  - Config Check    │      │  Query: 16686      │            │
│  │  - Wizards         │      │  Agent: 5775       │            │
│  │                    │      │                    │            │
│  │  Backend:          │      │  Storage: Memory   │            │
│  │  - Prometheus      │      │  Traces: 50K       │            │
│  │  - Istio API       │      └────────────────────┘            │
│  └────────────────────┘                                        │
│                                                                  │
│  Access via Istio Gateway:                                      │
│  - /grafana/                                                    │
│  - /prometheus/                                                 │
│  - /kiali/                                                      │
│  - /jaeger/                                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## High Availability Architecture

### Control Plane HA

```
┌──────────────────────────────────────────────────────────┐
│              Istio Control Plane (Istiod)                │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │ Replica 1│  │ Replica 2│  │ Replica 3│  │  ...   │  │
│  │          │  │          │  │          │  │        │  │
│  │  Leader  │  │ Follower │  │ Follower │  │ Auto   │  │
│  │  Election│  │          │  │          │  │ Scale  │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  │
│                                                          │
│  Horizontal Pod Autoscaler:                             │
│  - Min Replicas: 3                                      │
│  - Max Replicas: 10                                     │
│  - Target CPU: 80%                                      │
│                                                          │
│  Pod Disruption Budget:                                 │
│  - Min Available: 2                                     │
│  - Prevents voluntary disruptions                       │
└──────────────────────────────────────────────────────────┘
```

### Gateway HA

```
┌──────────────────────────────────────────────────────────┐
│         Istio Ingress Gateway (Load Balanced)            │
│                                                          │
│  Azure Internal Load Balancer                           │
│  └─────────────┬────────────────────────┐               │
│                │                        │               │
│                ▼                        ▼               │
│  ┌──────────────────┐      ┌──────────────────┐        │
│  │  Gateway Pod 1   │      │  Gateway Pod 2   │        │
│  │  - Envoy Proxy   │      │  - Envoy Proxy   │  ...   │
│  │  - Routes        │      │  - Routes        │        │
│  └──────────────────┘      └──────────────────┘        │
│                                                          │
│  Horizontal Pod Autoscaler:                             │
│  - Min Replicas: 3                                      │
│  - Max Replicas: 10                                     │
│  - Target CPU: 80%                                      │
│  - Target Memory: 80%                                   │
│                                                          │
│  Pod Disruption Budget:                                 │
│  - Min Available: 2                                     │
│  - Max Unavailable: 1                                   │
└──────────────────────────────────────────────────────────┘
```

---

## Multi-Environment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MetLife Azure Subscriptions                   │
└─────────────────────────────────────────────────────────────────┘
         │                    │                   │
    ┌────▼─────┐         ┌────▼─────┐       ┌────▼─────┐
    │   Dev    │         │   Int    │       │   QA     │
    │ Cluster  │         │ Cluster  │       │ Cluster  │
    └──────────┘         └──────────┘       └──────────┘
    Istio 1.x.x          Istio 1.x.x        Istio 1.x.x
    IP: .147.60          IP: .151.28        IP: .20.188
         │                    │                   │
         └────────────────────┼───────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
         ┌─────────┐                     ┌─────────┐
         │  Prod   │                     │   DR    │
         │ Cluster │                     │ Cluster │
         └─────────┘                     └─────────┘
         Istio 1.x.x                     Istio 1.x.x
         IP: .20.221                     IP: (standby)
         Active                          Gateways: OFF

Configuration Flow:
  Dev → Int → QA → Prod
                    ↓
                   DR (Synchronized)
```

---

## Security Architecture

### mTLS Communication

```
┌──────────────────────────────────────────────────────────┐
│                 Istio mTLS Architecture                   │
│                                                          │
│  Service A                          Service B           │
│  ┌────────────┐                    ┌────────────┐       │
│  │   Pod A    │                    │   Pod B    │       │
│  │            │                    │            │       │
│  │ ┌────────┐ │                    │ ┌────────┐ │       │
│  │ │  App   │ │                    │ │  App   │ │       │
│  │ └───┬────┘ │                    │ └───▲────┘ │       │
│  │     │      │                    │     │      │       │
│  │ ┌───▼────────┐  Encrypted    ┌─────┴──────┐ │       │
│  │ │   Envoy    │───────mTLS───>│   Envoy    │ │       │
│  │ │            │<──────────────│            │ │       │
│  │ │  - Client  │                │  - Server  │ │       │
│  │ │    Cert    │                │    Cert    │ │       │
│  │ └────────────┘                └────────────┘ │       │
│  └────────────┘                    └────────────┘       │
│         │                                  │             │
│         └──────────────┬───────────────────┘             │
│                        ▼                                 │
│              ┌──────────────────┐                        │
│              │  Istiod (Citadel)│                        │
│              │                  │                        │
│              │  - CA Authority  │                        │
│              │  - Issue Certs   │                        │
│              │  - Rotate Certs  │                        │
│              └──────────────────┘                        │
│                                                          │
│  Certificate Rotation: Every 24 hours                   │
│  TLS Version: 1.3                                       │
│  Cipher Suites: Strong encryption only                 │
└──────────────────────────────────────────────────────────┘
```

### Authentication & Authorization

```
External Request
      │
      ▼
┌─────────────────────┐
│ Ingress Gateway     │
│ - JWT Validation    │
│ - RequestAuth       │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Authorization Policy│
│ - RBAC Rules        │
│ - Service Account   │
│ - Namespace Rules   │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Service Envoy       │
│ - mTLS Verification │
│ - Identity Check    │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Application Service │
└─────────────────────┘
```

---

## Resource Requirements

### Control Plane Resources

| Component | Min CPU | Max CPU | Min Memory | Max Memory | Replicas |
|-----------|---------|---------|------------|------------|----------|
| Istiod | 500m | 1000m | 2048Mi | 2560Mi | 3-10 (HPA) |
| Ingress Gateway | 100m | 1000m | 128Mi | 1024Mi | 3-10 (HPA) |
| Egress Gateway | 100m | 1000m | 128Mi | 1024Mi | 3-10 (HPA) |

### Sidecar Resources (Per Pod)

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Envoy Proxy | 100m | 2000m | 128Mi | 1024Mi |

### Cluster Sizing Recommendations

**Small Environment (Dev/Int):**
- Nodes: 3-5
- Node Size: Standard_D4s_v3 (4 vCPU, 16GB RAM)

**Medium Environment (QA):**
- Nodes: 5-10
- Node Size: Standard_D8s_v3 (8 vCPU, 32GB RAM)

**Large Environment (Prod):**
- Nodes: 10-20
- Node Size: Standard_D16s_v3 (16 vCPU, 64GB RAM)

---

## Disaster Recovery Architecture

### DR Cluster Configuration

```
┌────────────────────────────────────────────────────────┐
│              Production Cluster (Active)                │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │  Istio Control Plane: Running                │    │
│  │  Ingress Gateway: Active (3+ replicas)       │    │
│  │  Egress Gateway: Active (3+ replicas)        │    │
│  │  Applications: Serving Traffic               │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────┬─────────────────────────────────────┘
                   │
                   │ Configuration Sync
                   │ (Manual/Automated)
                   ▼
┌────────────────────────────────────────────────────────┐
│            Disaster Recovery Cluster (Standby)         │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │  Istio Control Plane: Running                │    │
│  │  Ingress Gateway: STOPPED (0 replicas)       │    │
│  │  Egress Gateway: STOPPED (0 replicas)        │    │
│  │  Applications: Deployed but idle             │    │
│  └──────────────────────────────────────────────┘    │
│                                                        │
│  Failover Steps:                                      │
│  1. Scale ingress/egress gateways to 3+ replicas     │
│  2. Update DNS/Load Balancer to DR IP               │
│  3. Verify traffic flow                              │
│  4. Monitor health and performance                   │
└────────────────────────────────────────────────────────┘
```

---

## Image Registry Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Public Docker Hub (Source)                    │
│              docker.io/istio/*                             │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       │ Pull Images
                       │ (During Pre-requisites)
                       ▼
┌────────────────────────────────────────────────────────────┐
│              MetLife DTR                                   │
│              dtr.metlife.com/istio/*                       │
│              - operator:1.x.x                              │
│              - pilot:1.x.x                                 │
│              - proxyv2:1.x.x                               │
│              - *-distroless variants                       │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       │ Pull During Deployment
                       ▼
┌────────────────────────────────────────────────────────────┐
│              AKS Cluster Nodes                             │
│              - Image Pull Secrets: jfrogauth               │
│              - Cached locally on nodes                     │
└────────────────────────────────────────────────────────────┘
                       │
                       │ Also Uses
                       ▼
┌────────────────────────────────────────────────────────────┐
│              JFrog Artifactory                             │
│              jfrog-artifactory.metlife.com                 │
│              - Base images                                 │
│              - Application images                          │
└────────────────────────────────────────────────────────────┘
```

---

This architecture overview provides a comprehensive visual understanding of how the Digital SOH Istio system is structured, how components interact, and how data flows through the system.

