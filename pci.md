# PCI OCI Network — Handover/Takeover Questions

> **Legend:** `[OCI]` = OCI-specific | `[General]` = Cloud/platform agnostic

---

## Access & Credentials

1. `[OCI]` Who has tenancy admin access today? Can I get the list of all IAM users, groups, and policies?
2. `[OCI]` What are the OCI compartment-level permissions — who can do what and where?
3. `[General]` Where are the IPSec tunnel pre-shared keys stored? Is there a secrets manager or vault in use?
4. `[General]` What's the Pritunl VPN setup — where's the config, who manages user provisioning, and how are certs rotated?
5. `[General]` How is admin access audited? Are there any break-glass accounts?
6. `[General]` Are there any service accounts or API keys in use? Where are they documented?
7. `[OCI]` Is OCI Identity Federation configured with any external IdP (SAML, OIDC)? Who manages it?
8. `[General]` Is MFA enforced for all admin/console access?
9. `[General]` What's the password/key rotation policy? When was the last rotation done?
10. `[OCI]` Are there any OCI Identity Domains configured? Which domain type (Free, Oracle Apps, Premium)?

## Infrastructure as Code (IaC)

11. `[General]` Is this infrastructure managed via IaC (Terraform, OCI Resource Manager, etc.) or was it built manually via console?
12. `[General]` If IaC exists — where's the repo, what's the branching strategy, and what's the CI/CD pipeline for infra changes?
13. `[General]` If no IaC — which resources were created manually and are there any known drift issues?
14. `[OCI]` Are there any OCI Resource Manager stacks? Where are the state files stored?
15. `[General]` What Terraform version and provider versions are in use? Any version pinning?
16. `[General]` Are there any modules (custom or registry) being used? Where are they maintained?
17. `[General]` Is there a separate IaC pipeline for networking vs compute vs security, or is it monolithic?

## Networking — Existing Environment

18. `[OCI]` The existing route tables have several /32 host routes (10.30.5.2, 10.50.1.94, 10.50.1.131) — what are these hosts and why do they need specific routes?
19. `[General]` What's the 10.30.0.0/24 network in the DR route table? It's not documented anywhere.
20. `[General]` What's the 10.10.0.0/16 network (PCI-PROD-EOFFICE-DRG-Attachment)? Is eOffice a separate workload?
21. `[General]` What's the 10.70.0.0/16 network (PCI-Dev-NPROD-DC-DRG-ATTACHMENT) referenced in the DR RPC table?
22. `[OCI]` Are there any Security Lists or Network Security Groups (NSGs) configured? Can I get the full rule sets?
23. `[General]` What are the IPSec tunnel parameters — IKE version, encryption algo, DPD settings, BGP or static routing?
24. `[General]` How many IPSec tunnels are active and to which on-premises endpoints?
25. `[General]` Is there any network segmentation between App and DB tiers in the existing setup, or are they on the same subnet?
26. `[General]` Are there any third-party network virtual appliances (firewalls, IDS/IPS) deployed?
27. `[General]` What's the MTU configuration across the network — any jumbo frame settings?

## Networking — Future (V5) Environment

28. `[General]` What's the migration plan from existing to V5? Is it a parallel build or in-place migration?
29. `[General]` Is the V5 architecture already partially deployed or is it purely on paper?
30. `[General]` The DR region has no NPROD spoke VCN — is that intentional? Will NPROD ever need DR?
31. `[General]` How will traffic flow between PROD and NPROD spokes — is cross-spoke communication allowed or blocked by design?
32. `[OCI]` What are the NSG/Security List rules planned for the spoke VCNs?
33. `[General]` Is there a defined CIDR expansion plan if /24 subnets run out of IPs?
34. `[General]` What's the expected timeline for V5 migration? Are there any hard deadlines (audit, compliance)?
35. `[General]` Is there a rollback plan if V5 migration fails mid-way?
36. `[OCI]` Will the DRG be upgraded to DRGv2 (if not already) for the hub-and-spoke model?

## DNS & Name Resolution

37. `[OCI]` What DNS setup is in use — OCI DNS, custom resolvers, or on-premises DNS forwarding?
38. `[OCI]` Are there any private DNS zones configured?
39. `[General]` How do workloads in spoke VCNs resolve on-premises hostnames (and vice versa)?
40. `[General]` Are there any hardcoded IPs in application configs instead of DNS names?
41. `[General]` What's the DNS TTL policy? Any caching layers?

## Load Balancers

42. `[OCI]` What type of LBs are deployed — OCI LBaaS (flexible/dynamic), Network LB, or both?
43. `[General]` What are the backend sets, health check configs, and SSL termination settings?
44. `[General]` Are there any WAF policies already configured or is that part of V5 only?
45. `[General]` What are the current listener configurations (ports, protocols, routing rules)?
46. `[General]` What SSL/TLS certificates are in use — who issued them, when do they expire, and is renewal automated?
47. `[General]` Are there any rate limiting or DDoS protection rules configured?
48. `[General]` What's the session persistence (sticky session) configuration?

## Security & Compliance

49. `[General]` Is there a current PCI-DSS audit report or AOC (Attestation of Compliance)? What version of PCI-DSS?
50. `[General]` What's in scope for PCI — which subnets, which workloads, which data flows?
51. `[General]` Are there any compensating controls documented?
52. `[OCI]` Cloud Guard — is it enabled? What are the detector and responder recipes?
53. `[OCI]` Vulnerability Scanning — is it active on all compute instances? What's the scan schedule?
54. `[OCI]` Key Vault — what keys are stored, what's the rotation policy?
55. `[General]` Are there any encryption-at-rest and in-transit policies enforced at the tenancy level?
56. `[OCI]` Is there a logging/audit trail setup — OCI Audit, Service Logs, VCN Flow Logs?
57. `[General]` Is there a file integrity monitoring (FIM) solution in place? (PCI-DSS Req 11.5)
58. `[General]` Is there an intrusion detection/prevention system (IDS/IPS)? Where is it deployed?
59. `[General]` How is anti-malware/endpoint protection handled on compute instances?
60. `[General]` Are there any network ACLs or micro-segmentation policies beyond NSGs?
61. `[General]` Is there a penetration testing schedule? When was the last pentest and what were the findings?
62. `[General]` How is cardholder data identified and classified? Is there a data flow diagram?
63. `[General]` Are there any data loss prevention (DLP) controls in place?
64. `[OCI]` Are OCI Bastion sessions used for any access, or is it all through VPN?
65. `[General]` Is there a security incident response plan? Has it been tested?

## Monitoring & Alerting

66. `[General]` What monitoring is in place today — OCI Monitoring, custom dashboards, third-party tools?
67. `[General]` ELK stack in V5 — is it already deployed? What's the log ingestion pipeline (Filebeat, Fluentd, OCI Logging)?
68. `[General]` What alerts are configured and who receives them? Is there a PagerDuty/OpsGenie integration?
69. `[OCI]` Are VCN Flow Logs enabled? Where are they stored and for how long?
70. `[General]` What's the log retention policy? Does it meet PCI-DSS requirements (minimum 1 year, 3 months immediately available)?
71. `[General]` Are there any synthetic monitoring or uptime checks configured?
72. `[General]` Is there APM (Application Performance Monitoring) in place?
73. `[General]` How are infrastructure metrics (CPU, memory, disk, network) tracked and alerted on?
74. `[General]` Is there a centralized logging destination, or are logs scattered across instances?

## Compute & Workloads

75. `[OCI]` What are the compute shapes in use (VM, BM, Flex)? OS versions?
76. `[General]` How are instances patched — manual, OS Management, Ansible, etc.?
77. `[General]` Are there any auto-scaling configurations?
78. `[General]` What's running on the App Nodes and DB Nodes — specific software, versions, dependencies?
79. `[OCI]` Are there any container workloads (OKE) or is everything VM-based?
80. `[General]` Are golden images/AMIs maintained? Where and how are they built?
81. `[General]` What's the instance provisioning process — manual, cloud-init, Ansible, Chef, Puppet?
82. `[General]` Are there any cron jobs or scheduled tasks running on instances? Where are they documented?
83. `[General]` What's the OS hardening standard in use (CIS benchmarks, STIG)?
84. `[OCI]` Are boot volumes and block volumes encrypted? With OCI-managed keys or customer-managed?
85. `[General]` Are there any GPU or HPC instances in the environment?
86. `[General]` What's the instance lifecycle — how are decommissions handled?

## Storage & Object Storage

87. `[OCI]` Is OCI Object Storage used? What buckets exist, what's stored, and what are the retention policies?
88. `[OCI]` Are Object Storage buckets public or private? Are there any pre-authenticated requests (PARs) active?
89. `[OCI]` Is OCI File Storage (FSS) used for any shared file systems?
90. `[General]` What's the backup storage strategy — local snapshots, cross-region copies, or external?
91. `[General]` Are there any data lifecycle policies (auto-archive, auto-delete)?
92. `[OCI]` Is storage encryption using OCI Vault customer-managed keys or Oracle-managed?

## Database

93. `[OCI]` What database service is in use — OCI DBCS, Autonomous DB, Exadata, or self-managed on compute?
94. `[General]` What's the backup strategy — frequency, retention, cross-region replication?
95. `[OCI]` Is Data Guard configured between DC and DR for database replication?
96. `[General]` What's the RPO/RTO target?
97. `[General]` What database versions and patch levels are running?
98. `[General]` How is database access controlled — network-level, DB-level users, or both?
99. `[General]` Are there any database encryption settings (TDE, column-level encryption)?
100. `[General]` Is there a database audit trail enabled? Where are audit logs shipped?
101. `[General]` Are there any database jobs, replication streams, or ETL processes running?
102. `[General]` What's the database connection string/method used by applications — direct, connection pool, SCAN?

## DR & Business Continuity

103. `[General]` Is there a documented DR runbook? Has it been tested?
104. `[General]` What's the failover process — manual or automated?
105. `[General]` What's the current RPO and RTO, and does the V5 design change those targets?
106. `[General]` How is data replicated between DC Mumbai and DR Hyderabad?
107. `[OCI]` Are there any dependencies on services that don't have DR (e.g., single-region OCI services)?
108. `[General]` When was the last DR drill? What was the outcome?
109. `[General]` Is there a documented failback procedure (DR → DC)?
110. `[General]` Are DNS/traffic management entries configured for automatic failover (e.g., OCI Traffic Management, global LB)?
111. `[General]` What's the communication plan during a DR event — who gets notified, through what channel?

## CI/CD & Application Deployment

112. `[General]` What CI/CD tools are in use (Jenkins, GitLab CI, OCI DevOps, GitHub Actions)?
113. `[General]` Where are application artifacts stored (container registry, artifact repo, S3/Object Storage)?
114. `[General]` What's the deployment strategy — blue/green, rolling, canary, or manual?
115. `[General]` How are application configs and environment variables managed — config files, env vars, vault?
116. `[General]` Is there a separate pipeline for PROD vs NPROD deployments?
117. `[General]` What's the rollback process for a failed deployment?
118. `[General]` Are there any deployment dependencies on the network (e.g., pulling images from external registries)?

## Operational Runbooks & Processes

119. `[General]` Are there runbooks for common operations — instance restart, LB changes, route table updates, IPSec tunnel troubleshooting?
120. `[General]` What's the change management process — is there a CAB, change windows, approval flow?
121. `[General]` Who's the current on-call rotation and escalation path?
122. `[General]` Are there any known issues, tech debt items, or "things that break often"?
123. `[General]` Any scheduled maintenance windows or upcoming OCI maintenance events?
124. `[General]` What ticketing system is used (Jira, ServiceNow, etc.)? Are there open tickets being handed over?
125. `[General]` Are there any SLAs with internal teams or external customers tied to this infrastructure?
126. `[General]` What's the capacity planning process? Are there any upcoming scaling needs?

## Cost & Billing

127. `[General]` What's the current monthly spend? Any reserved instances or committed use discounts?
128. `[OCI]` Are there cost allocation tags on resources?
129. `[General]` Any resources that are running but shouldn't be (orphaned, forgotten test instances)?
130. `[OCI]` Are there any OCI budget alerts or cost anomaly detection configured?
131. `[General]` Is there a FinOps process or regular cost review cadence?
132. `[General]` Are there any third-party licenses tied to this infrastructure (Oracle DB, middleware, monitoring tools)?

## Vendor & Third-Party Dependencies

133. `[General]` Are there any third-party services or SaaS integrations connected to this environment?
134. `[General]` What vendor support contracts are active (OCI support tier, DB support, security tools)?
135. `[General]` Are there any managed service providers involved in day-to-day operations?
136. `[General]` What's the Nextra Network contract/SLA for the IPSec connectivity?
137. `[General]` Are there any software licenses expiring soon?

## Tagging, Limits & Governance

146. `[General]` Is there a resource tagging standard? Are tags enforced via policy?
147. `[OCI]` Have any OCI service limits been increased (compute, VCN, LB, DRG attachments)? Which ones?
148. `[OCI]` Are there OCI quota policies set at the compartment level?
149. `[OCI]` Are OCI Events and Notification topics configured for any automated responses?
150. `[General]` Is there a resource naming convention document? Does the existing environment follow it?

## Instance-Level Security

151. `[General]` Are there OS-level firewalls (iptables/firewalld/nftables) configured on instances? Where are the rules documented?
152. `[General]` Is NTP/time synchronization configured consistently across all instances? What's the source? (Critical for PCI audit log correlation)
153. `[General]` Are SSH keys rotated? Who has SSH access to production instances?
154. `[General]` Is there any host-based intrusion detection (OSSEC, Wazuh, etc.)?
155. `[General]` Are there any local user accounts on instances beyond the default opc/admin?

## Backup & Restore Validation

156. `[General]` Has a backup restore ever been tested? When was the last successful restore test?
157. `[General]` Are boot volume and block volume backups automated? What's the schedule and retention?
158. `[OCI]` Are backups cross-region copied for DR (boot volumes, block volumes, DB backups)?
159. `[General]` Is there a documented restore procedure with estimated RTO per component?

## Egress & Outbound Traffic

160. `[General]` What outbound (egress) traffic is allowed from private subnets? Is there an explicit allow-list?
161. `[General]` Do instances need to reach external endpoints (package repos, APIs, SaaS tools)? Through NAT or proxy?
162. `[General]` Is there any egress filtering or inspection (proxy, firewall) for outbound internet traffic?
163. `[General]` Are there any external API integrations that depend on whitelisted source IPs?

## Network Validation

164. `[General]` Can the outgoing team provide a network path validation (traceroute/MTR) between DC and DR?
165. `[General]` Has latency between DC Mumbai and DR Hyderabad been measured? What's the baseline?
166. `[General]` Are there any known network bottlenecks or bandwidth constraints on the IPSec tunnels?
167. `[OCI]` Is FastConnect used or planned alongside IPSec, or is IPSec the only on-premises link?

## Compliance Tooling

168. `[General]` Is there a CIS benchmark scanner or PCI compliance scanner running regularly (Qualys, Nessus, Prisma, etc.)?
169. `[General]` Are compliance scan reports available for the last 12 months?
170. `[General]` Is there an automated compliance-as-code framework (InSpec, Open Policy Agent, etc.)?
171. `[General]` Who is the QSA (Qualified Security Assessor) for PCI audits? When is the next audit?

## Documentation & Handover Artifacts

138. `[General]` Beyond these two diagrams, is there any other documentation — HLD, LLD, SOPs, network diagrams with IP details?
139. `[General]` Is there a CMDB or asset inventory?
140. `[General]` Where are passwords, keys, and secrets stored (OCI Vault, external tool, spreadsheet)?
141. `[OCI]` Can I get read-only console access to walk through the environment before full handover?
142. `[General]` Is there a knowledge base or wiki with tribal knowledge documented?
143. `[General]` Can we schedule a walkthrough session (screen share) of the live environment?
144. `[General]` Who are the key contacts on the outgoing team for post-handover questions (grace period)?
145. `[General]` Is there a formal handover sign-off document or checklist?

---

**Total: 171 questions**
