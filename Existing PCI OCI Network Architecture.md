# Existing PCI OCI Network Architecture

## 1. Overview

This document describes the current-state PCI-compliant network architecture deployed on Oracle Cloud Infrastructure (OCI). The environment spans two OCI regions in a primary/DR configuration, using a single-VCN-per-region model with DRG-based connectivity.

---

## 2. Regions

| Region | Role | Location |
|--------|------|----------|
| OCI DC Mumbai | Primary (DC) | Mumbai |
| OCI DR Hyderabad | Disaster Recovery (DR) | Hyderabad |

Both regions reside within a single **PCI Tenancy**.

---

## 3. DC Mumbai Region

### 3.1 VCN & Subnets

- **Compartment:** Dev Compartment
- **Availability Domain:** AD 1
- **VCN:** `dev` — `10.0.0.0/16`

| Subnet | CIDR | Purpose |
|--------|------|---------|
| Production subnet | 10.0.0.0/24 | App and DB workloads |
| Private Network-LB | 10.0.1.0/24 | Load Balancer |

### 3.2 Gateways

| Gateway | Purpose |
|---------|---------|
| Internet Gateway | Public internet access for end users |
| Dynamic Routing Gateway (DRG) | On-premises and cross-region connectivity |
| Remote Peering Gateway | DC-to-DR inter-region peering |

### 3.3 DRG Attachments

- `PCI-HUB-DC-DRG Attachment` — Hub routing
- `DRG Attachment for IPSec Tunnels` — On-premises connectivity via Nextra Network

### 3.4 Components

- App Nodes
- DB Nodes
- Load Balancer

### 3.5 Connectivity

- **End User Access** — via Internet Gateway
- **Admin Access** — via IPSec Tunnel over Nextra Network
- **Inter-region** — DC Remote Peering Connection to DR Hyderabad

### 3.6 Dev Route Table

| Destination | Target | Description |
|-------------|--------|-------------|
| 0.0.0.0/0 | Internet Gateway | Towards Internet |
| 10.14.153.0/24 | Dynamic Routing Gateway | Towards NXTRA Network |
| 10.20.0.0/16 | Dynamic Routing Gateway | DR region |
| 10.30.5.2/32 | Dynamic Routing Gateway | — |
| 10.50.0.0/16 | Dynamic Routing Gateway | — |
| 10.50.1.94/32 | Dynamic Routing Gateway | Collector Server |
| 10.50.1.131/32 | Dynamic Routing Gateway | — |

### 3.7 RPC (HUB) DC Route Table

| Destination | Next Hop | Description |
|-------------|----------|-------------|
| 10.0.0.0/16 | Virtual Cloud Network | DC dynamic gateway |
| 10.10.0.0/16 | Virtual Cloud Network | PCI-PROD-EOFFICE-DRG-Attachment |
| 10.50.0.0/16 | Virtual Cloud Network | PCI-HUB-DRG-Attachment |

---

## 4. DR Hyderabad Region

### 4.1 VCN & Subnets

- **Compartment:** Dev Compartment
- **Availability Domain:** AD 1
- **VCN:** `dev` — `10.20.0.0/16`

| Subnet | CIDR | Purpose |
|--------|------|---------|
| Production subnet | 10.20.0.0/24 | App and DB workloads |

### 4.2 Gateways

| Gateway | Purpose |
|---------|---------|
| Internet Gateway | Public internet access |
| Dynamic Routing Gateway (DRG) | On-premises and cross-region connectivity |
| Remote Peering Gateway | DR-to-DC inter-region peering |

### 4.3 DRG Attachments

- `PCI-HUB-DR-DRG Attachment` — Hub routing
- `DRG Attachment for IPSec Tunnels` — On-premises connectivity

### 4.4 Components

- App Nodes
- DB Nodes
- Load Balancer

### 4.5 HUB Route Table

| Destination | Target | Description |
|-------------|--------|-------------|
| 0.0.0.0/0 | Internet Gateway | Towards Internet |
| 10.14.153.0/24 | Dynamic Routing Gateway | Towards NXTRA Network |
| 10.0.0.0/16 | Dynamic Routing Gateway | DC region |
| 10.30.5.2/32 | Dynamic Routing Gateway | — |
| 10.50.0.0/16 | Dynamic Routing Gateway | — |
| 10.30.0.0/24 | Dynamic Routing Gateway | — |
| 10.50.1.131/32 | Dynamic Routing Gateway | — |

### 4.6 RPC DC Route Table

| Destination | Next Hop | Description |
|-------------|----------|-------------|
| 10.60.0.0/16 | Virtual Cloud Network | PCI-Dev-HUB-DC-DRG-ATTACHMENT |
| 10.0.0.0/16 | Virtual Cloud Network | PCI-Dev-PROD-DC-DRG-ATTACHMENT |
| 10.70.0.0/16 | Virtual Cloud Network | PCI-Dev-NPROD-DC-DRG-ATTACHMENT |

---

## 5. IP Address Allocation

| Network | CIDR | Region | Purpose |
|---------|------|--------|---------|
| Dev VCN (DC) | 10.0.0.0/16 | DC Mumbai | Primary workloads |
| Dev VCN (DR) | 10.20.0.0/16 | DR Hyderabad | DR workloads |
| Nextra Network | 10.14.153.0/24 | On-premises | Corporate/admin network |

---

## 6. Connectivity Diagram

```
  End Users ──► Internet Gateway ──► Load Balancer ──► App Nodes ──► DB Nodes
                                                          │
  Admin ──► IPSec Tunnel (Nextra) ──► DRG ────────────────┘
                                       │
                              Remote Peering Connection
                                       │
                                       ▼
                              DR Hyderabad Region
```

---

## 7. Limitations & Gaps

- Single VCN per region — no environment isolation (PROD/NPROD share the same network)
- Generic compartment naming ("Dev Compartment")
- No visible security services (WAF, vulnerability scanning, key management)
- No Service Gateway or NAT Gateway
- No centralized monitoring infrastructure
- Several /32 host routes in the route table suggest ad-hoc routing additions
- Load balancer on a private subnet — public access path unclear
