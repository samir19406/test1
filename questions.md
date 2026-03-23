# Pre-Proposal Questionnaire — PCI OCI Infrastructure

> These questions must be answered before preparing any future-state proposals.
> Two proposals will be prepared: (1) OCI Improvement and (2) AWS Migration Alternative.

---

## 1. Current Utilisation & Capacity

### Compute
1. How many compute instances are running across DC and DR? List per region.
2. What are the instance shapes (vCPU, RAM, storage) for each — App Nodes, DB Nodes, LB, VPN, monitoring?
3. What is the average and peak CPU utilisation per instance over the last 3–6 months?
4. What is the average and peak memory utilisation per instance over the last 3–6 months?
5. Are any instances consistently over-provisioned (running below 20% utilisation)?
6. Are any instances consistently under-provisioned (hitting 80%+ regularly)?
7. What is the total vCPU and RAM allocated vs actually consumed?

### Storage
8. What is the total block volume storage provisioned vs actually used?
9. What is the boot volume size per instance?
10. Is Object Storage in use? How much data is stored and what's the growth rate (monthly/quarterly)?
11. Are there any file storage (FSS) mounts? Size and usage?
12. What is the storage IOPS requirement — are current volumes meeting performance needs?

### Network
13. What is the average and peak network throughput on the IPSec tunnels (Mbps/Gbps)?
14. What is the bandwidth utilisation on the load balancers?
15. What is the latency between DC Mumbai and DR Hyderabad (measured)?
16. How much data transfers between DC and DR monthly (for cost estimation)?
17. What is the internet egress volume per month (GB)?

### Database
18. What database engine and version is in use (Oracle DB, MySQL, PostgreSQL, etc.)?
19. What is the database size (data + indexes) per environment?
20. What is the database growth rate (monthly/quarterly)?
21. What are the peak concurrent connections and queries per second?
22. What is the database CPU and memory utilisation trend?

### Load Balancer
23. What is the average requests per second hitting the load balancer?
24. What is the peak requests per second?
25. What is the average response time at the LB level?
26. How many backend servers are in each backend set?

---

## 2. Application & Workload Details

27. What applications are running? List each with a brief description.
28. What is the technology stack per application (language, framework, runtime, middleware)?
29. Are applications stateful or stateless?
30. What are the application dependencies — internal services, external APIs, third-party SaaS?
31. What are the application ports and protocols in use?
32. Is there any message queue, caching layer, or middleware (Redis, Kafka, RabbitMQ, etc.)?
33. How are application deployments done today — manual, CI/CD, scripts?
34. What is the deployment frequency — daily, weekly, monthly?
35. Are there any batch jobs or scheduled processes? What time do they run and how long do they take?
36. What is the expected user base — concurrent users, total users, geographic distribution?

---

## 3. Data Classification & Compliance

37. What type of data is processed — cardholder data, PII, government-sensitive data?
38. What is the PCI-DSS scope — which systems, subnets, and data flows are in scope?
39. What PCI-DSS version is the current compliance based on?
40. Are there any additional compliance requirements beyond PCI-DSS (ISMS, ISO 27001, government-specific mandates like MeitY guidelines)?
41. Is there a data residency requirement — must data stay within India?
42. Is there a data classification policy? What are the classification levels?
43. Are there any data sovereignty or data localisation regulations that apply?
44. Who is the current QSA (Qualified Security Assessor)? When is the next audit?
45. Are there any existing audit findings or non-conformities that need to be addressed?

---

## 4. Availability & Performance Requirements

46. What is the required uptime SLA — 99.9%, 99.95%, 99.99%?
47. What is the acceptable RPO (Recovery Point Objective)?
48. What is the acceptable RTO (Recovery Time Objective)?
49. Is active-active DR required, or is active-passive acceptable?
50. What is the acceptable maximum latency for end-user transactions?
51. Are there any seasonal or event-driven traffic spikes? When and how much?
52. Is auto-scaling a requirement or is fixed capacity acceptable?

---

## 5. Security Requirements

53. What are the mandatory security controls expected (WAF, IDS/IPS, DLP, SIEM, FIM)?
54. Is there a requirement for a dedicated SIEM solution? If yes, any preference (Splunk, ELK, QRadar, OCI/AWS native)?
55. Is end-to-end encryption (in-transit + at-rest) mandatory?
56. Is there a requirement for HSM (Hardware Security Module) for key management?
57. What is the log retention requirement (PCI minimum is 1 year, 3 months immediately accessible)?
58. Is there a penetration testing requirement and frequency?
59. Is there a vulnerability scanning requirement and frequency?
60. Are there specific government security standards to follow (e.g., MeitY Cloud Guidelines, CERT-In directives)?
61. Is there a requirement for network-level micro-segmentation?
62. Is multi-factor authentication (MFA) mandatory for all access?

---

## 6. Connectivity & Network Requirements

63. What on-premises locations need connectivity to the cloud environment?
64. What is the required bandwidth for on-premises to cloud connectivity?
65. Is dedicated connectivity (OCI FastConnect / AWS Direct Connect) required, or is IPSec VPN sufficient?
66. Are there any other cloud environments that need to be connected (multi-cloud)?
67. What is the current internet bandwidth requirement for end users?
68. Is there a requirement for a CDN (Content Delivery Network)?
69. Are there any IP whitelisting requirements with external parties?
70. Is IPv6 required or planned?

---

## 7. DR & Business Continuity Requirements

71. Is DR mandatory? (Assuming yes given current DC/DR setup)
72. Must the DR site be in a different geographic region within India?
73. Is there a preference for DR region (Hyderabad, or open to others)?
74. Is automated failover required, or is manual failover acceptable?
75. What is the maximum acceptable data loss during a disaster (RPO)?
76. What is the maximum acceptable downtime during a disaster (RTO)?
77. How frequently should DR drills be conducted?
78. Is there a requirement for a third site (e.g., pilot light or backup)?

---

## 8. Operational & Management Requirements

79. What is the expected team size for managing the infrastructure?
80. Is 24x7 monitoring required?
81. Is there a preference for managed services vs self-managed?
82. What ticketing/ITSM tool is in use (ServiceNow, Jira, etc.)?
83. Is there a requirement for infrastructure-as-code (Terraform, CloudFormation, etc.)?
84. What is the change management process — CAB, approval matrix?
85. Are there any specific SLA reporting requirements for the government body?
86. Is there a requirement for a dedicated operations dashboard?

---

## 9. Budget & Commercial

87. What is the current monthly/annual infrastructure spend?
88. Is there a defined budget for the improved/new infrastructure?
89. Is there a preference for CAPEX vs OPEX model?
90. Are there any existing OCI commitments (Universal Credits, annual contracts)?
91. Is cost optimisation a primary driver, or is it secondary to security/compliance?
92. Is there a government procurement process that affects cloud vendor selection (GeM, empanelment)?
93. Are both OCI and AWS empanelled/approved for this government body?

---

## 10. Migration & Timeline

94. What is the expected timeline for the infrastructure improvement/migration?
95. Is there a hard deadline (audit date, contract renewal, compliance deadline)?
96. Is a phased approach acceptable, or does it need to be a big-bang migration?
97. What is the acceptable downtime window for migration?
98. Are there any blackout periods where changes cannot be made?
99. Is there an existing migration team, or does this need to be staffed?

---

## 11. Current Challenges & Pain Points

> Understanding what's broken or painful today is critical — it defines the priority for both proposals.

### Infrastructure & Network
106. What are the top 3–5 infrastructure challenges you face today?
107. Are there any recurring outages or incidents? What's the frequency and root cause?
108. Are there any performance bottlenecks — slow application response, DB query delays, network latency?
109. Is the current IPSec tunnel reliable, or are there frequent disconnections/flapping?
110. Are there any capacity issues — running out of IPs, hitting OCI service limits, storage full?
111. Is the current single-VCN flat network causing any security or operational issues?
112. Are there any routing issues or traffic blackholes in the current setup?

### Security & Compliance
113. Are there any open audit findings or non-conformities from the last PCI-DSS assessment?
114. What security gaps are you most concerned about today?
115. Has there been any security incident or breach in the past 12 months?
116. Is the lack of WAF, IDS/IPS, or SIEM causing compliance concerns?
117. Is there any difficulty in producing audit evidence or compliance reports with the current setup?
118. Are there any unencrypted data flows or storage that need to be addressed?

### Operations & Management
119. Is the lack of IaC causing operational pain — manual changes, configuration drift, slow provisioning?
120. Are deployments slow or error-prone? What's the average deployment time?
121. Is there a lack of monitoring/alerting that has led to missed incidents?
122. Are there any "single person dependency" risks — knowledge that only one person holds?
123. Is patching and OS updates a challenge? Are instances running outdated/unpatched software?
124. Is there any difficulty in troubleshooting issues due to lack of centralized logging?

### DR & Availability
125. Has DR ever been tested? Did it work?
126. Is the current DR setup giving confidence, or is it "there on paper but untested"?
127. Is there any data replication lag between DC and DR that's a concern?
128. Has there been a situation where DR was needed but couldn't be activated?

### Cost & Efficiency
129. Is the current infrastructure cost-efficient, or is there suspected overspend?
130. Are there idle or underutilised resources that could be right-sized or decommissioned?
131. Is there visibility into cost breakdown per environment/application, or is it a single bill with no clarity?
132. Are there any unexpected cost spikes? What caused them?

### Team & Process
133. Is the current team adequately skilled for the platform (OCI), or is there a skills gap?
134. Is the change management process too slow or too informal?
135. Are there any vendor support issues — slow response, unresolved tickets?
136. What would you fix first if you had unlimited budget and time?

---

## 12. Future Growth & Scalability

100. What is the expected growth in users over the next 1–3 years?
101. Are there new applications or workloads planned?
102. Is there a plan to onboard additional environments (UAT, staging, sandbox)?
103. Is containerisation (Kubernetes) on the roadmap?
104. Is there a plan for API gateway or microservices architecture?
105. Are there any plans for AI/ML workloads or big data processing?

---

> **Note:** Answers to these questions will directly feed into both proposals (OCI Improvement + AWS Alternative). Incomplete answers will result in assumptions that may need to be revised later.
>
> **Total: 136 questions**
