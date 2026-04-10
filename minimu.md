---
title: Minimum Requirements to Start Testing (NPROD)
---
flowchart TB

    subgraph MINIMUM["🟢 MINIMUM TO START TESTING"]
        direction TB

        subgraph NETWORK["1️⃣ Network Foundation"]
            HUB_VCN["HUB VCN\n10.50.0.0/16"]
            NPROD_VCN["NPROD VCN\n10.60.0.0/16"]
            DRG["DRG\n(Hub ↔ NPROD peering)"]
            IGW["Internet Gateway\n(HUB)"]
            NAT["NAT Gateway\n(NPROD outbound)"]
            SGW["Service Gateway\n(Oracle Services)"]
        end

        subgraph SUBNETS["2️⃣ Subnets (4 minimum)"]
            LB_SN["HUB NPROD LB Public SN\n10.50.3.0/24"]
            INFRA_SN["HUB Infra Public SN\n10.50.0.0/24"]
            APP_SN["NPROD App Private SN\n10.60.0.0/24"]
            DB_SN["NPROD DB Private SN\n10.60.1.0/24"]
        end

        subgraph COMPUTE["3️⃣ Compute & Data"]
            LB["NPROD Load Balancer"]
            APP["App Node\n(min 1 instance)"]
            DB["DB Node\n(min 1 instance)"]
        end

        subgraph ACCESS["4️⃣ Access"]
            VPN["Pritunl VPN\n(admin access)"]
            NSG["Network Security Groups\n(LB → App → DB rules)"]
            RT["Route Tables\n(HUB ↔ NPROD)"]
        end

        subgraph SECURITY["5️⃣ Security (PCI minimum)"]
            WAF["WAF\n(in front of LB)"]
            CERTS["TLS Certificates"]
        end
    end

    subgraph OPTIONAL["⚪ CAN WAIT — Not Needed for Day 1 Testing"]
        direction TB
        ELK["ELK Monitoring"]
        KV["Key Vault"]
        VS["Vulnerability Scanning"]
        CG["Cloud Guard"]
        DR["DR Region (Hyderabad)"]
        PROD["PROD Spoke VCN"]
        IPSEC["IPSec / Nextra Tunnel"]
    end

    %% Flow
    User((Tester)) --> WAF
    WAF --> LB
    LB --> APP
    APP --> DB
    VPN -.-> APP
    VPN -.-> DB
    APP -.-> NAT
    APP -.-> SGW

    %% Network links
    HUB_VCN --- DRG
    DRG --- NPROD_VCN
    IGW --- LB_SN
    LB_SN --- LB
    INFRA_SN --- VPN
    APP_SN --- APP
    DB_SN --- DB

    %% Styles
    style MINIMUM fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style NETWORK fill:#e3f2fd,stroke:#1565c0
    style SUBNETS fill:#e3f2fd,stroke:#1565c0
    style COMPUTE fill:#fff3e0,stroke:#e65100
    style ACCESS fill:#f3e5f5,stroke:#6a1b9a
    style SECURITY fill:#fce4ec,stroke:#c62828
    style OPTIONAL fill:#f5f5f5,stroke:#9e9e9e,stroke-dasharray: 5 5
    style ELK stroke-dasharray: 5 5
    style KV stroke-dasharray: 5 5
    style VS stroke-dasharray: 5 5
    style CG stroke-dasharray: 5 5
    style DR stroke-dasharray: 5 5
    style PROD stroke-dasharray: 5 5
    style IPSEC stroke-dasharray: 5 5
