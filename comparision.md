# PCI OCI Network Architecture — Existing vs Future (V5) Summary

---

## Existing Architecture

- Single VCN per region — everything (app, DB, LB) lives in one flat network (`10.0.0.0/16` in DC, `10.20.0.0/16` in DR)
- No environment separation — PROD and NPROD share the same compartment labeled "Dev Compartment"
- Basic subnets — just "Production" and "Network-LB", no clear app/DB tier isolation
- Admin access via generic IPSec tunnel over Nextra Network
- No security services visible — no WAF, no key management, no vulnerability scanning, no Cloud Guard
- No NAT Gateway or Service Gateway — unclear how private resources reach Oracle services or internet outbound
- Several undocumented /32 host routes and mystery CIDRs (10.30.x, 10.10.x, 10.70.x) suggesting ad-hoc growth
- No centralized monitoring
- Generic naming — hard to identify what belongs where

---

## Future (V5) Architecture

- Hub-and-spoke model — dedicated HUB VCN (`10.50.0.0/16`) with separate spoke VCNs for PROD (`10.0.0.0/16`) and NPROD (`10.60.0.0/16`)
- Proper compartment isolation — PCI-DIGI-HUB, PCI-DIGI-PROD, PCI-DIGI-NPROD
- Purpose-built subnets — separate App, DB, LB, and Infra subnets, each public or private by design
- Security services added — WAF, Key Vault, Certificates, Vulnerability Scanning, Cloud Guard
- NAT Gateway + Service Gateway — private subnets get outbound internet and Oracle Services access without public exposure
- Pritunl VPN for admin access — replaces generic IPSec admin path
- ELK Monitoring — centralized logging
- Separate PROD and NPROD load balancers on dedicated public subnets in the HUB
- Structured naming convention (`PCI-DIGI-{ENV}-{FUNCTION}-{TYPE}-SN`)
- DR region scoped to production only — no NPROD spoke in Hyderabad

---

## Side-by-Side Comparison

| Aspect | Existing | Future (V5) |
|--------|----------|-------------|
| VCN Model | 1 VCN per region | Hub + Spoke (3 VCNs in DC, 2 in DR) |
| Compartments | Single "Dev" | HUB, PROD, NPROD (isolated) |
| Subnets | 2 generic | 8+ purpose-built (App/DB/LB/Infra, pub/pvt) |
| DC HUB CIDR | N/A (part of 10.0/16) | 10.50.0.0/16 |
| DC PROD CIDR | 10.0.0.0/16 | 10.0.0.0/16 (preserved) |
| DC NPROD CIDR | N/A | 10.60.0.0/16 (new) |
| DR HUB CIDR | N/A (part of 10.20/16) | 10.40.0.0/16 |
| DR PROD CIDR | 10.20.0.0/16 | 10.20.0.0/16 (preserved) |
| WAF | None | Yes |
| Key Management | None visible | OCI Key Vault |
| Vuln Scanning | None visible | OCI Vulnerability Scanning |
| Cloud Guard | None visible | Enabled |
| Monitoring | None visible | ELK stack |
| Admin Access | IPSec (generic) | Pritunl VPN |
| NAT Gateway | No | Yes |
| Service Gateway | No | Yes (Object Storage + Oracle Services) |
| Load Balancers | 1 shared | Separate PROD + NPROD LBs |
| Naming | Generic | Structured convention |
| DR NPROD | Referenced but unclear | Intentionally excluded |

---

## Bottom Line

Existing is a flat, single-VCN setup with minimal security controls. V5 is a proper enterprise-grade, PCI-aligned redesign with isolation, security layering, and operational tooling.
