# SAP HR Transformation — Knowledge Library

A practitioner's reference library for SAP HR Transformation programmes, covering the end-to-end **Hire-to-Retire (H2R)** business process. The library is organized into three pillars: **Strategy & Governance**, **SAP Services** (cloud platform, integration, identity, and operations), and **SAP HR Transformation** (on-premise ECC/S4 HCM core configuration). Each section contains architecture decisions, configuration guides, and step-by-step screenshots built from real project experience.

---

## What This Library Covers

This repository is a living knowledge base for SAP integration architects, BTP engineers, and HCM functional consultants. It spans both ends of an H2R landscape: the cloud-side strategy, governance, and integration layer (BTP, Integration Suite, IAS/IPS) and the on-premise core — building and configuring SAP ECC/S4 HCM from the ground up (Enterprise Structure, Personnel Structure, Organizational Management, Payroll, and Time Management).

---

## Repository Structure

```
├── 01 Strategy/
│   ├── Integration Principles & Methodology
│   ├── Cloud Integration Strategy & Governance
│   ├── BTP Infrastructure Management
│   └── SSO End-to-End Process
│
├── 02 SAP Services/
│   ├── SAP BTP — Account Model & Infrastructure
│   ├── SAP IAS & IPS — Identity & SSO
│   ├── SAP Integration Suite — iFlow Development
│   ├── SAP Cloud ALM — Integration Monitoring
│   ├── SAP Cloud Transport Management (cTMS)
│   └── SAP BTP Audit Log & SIEM Integration
│
└── 03 SAP HR Transformation/
    ├── HR Data Migration/
    │   ├── Phase 1 Organization Set up — Stages 0–10
    │   │   (Enterprise Structure, Number Ranges, Payroll Area & Control Record,
    │   │    PINCH & Admin Groups, Pay Scale Structure, Features, Organizational
    │   │    Management, PA-OM Integration, Personnel Administration, Employee
    │   │    Hiring, Time Management)
    │   └── Phase 2 SF Integration
    │       (Migration, Replication, Employee Migration Scenarios)
```

---

## 01 Strategy

The strategy layer defines **what** to build and **why** — before a single iFlow is written or a subaccount created.

### Integration Principles & Methodology

Governing principles applied across all H2R integration scenarios:

- **Seamless Integrations** — design integrations that support effortless business process execution without friction
- **Reusability** — abstract integration patterns as accelerators for future scenarios; favour modular iFlow design to lower Total Cost of Ownership
- **API-First** — prioritise OData and REST APIs over IDocs or file-based transfers wherever possible
- **Modernisation** — enforce current security standards: OAuth 2.0 for all SuccessFactors communication, TLS 1.2+ everywhere, and no Basic Auth in production
- **Holistic Integration** — cover all integration domains: Process, Data, Event-based, and AI-driven
- **Best Practices** — follow SAP Integration Solution Advisory Methodology (ISA-M) as the design framework

### Cloud Integration Strategy & Governance

Governance is not optional in an H2R landscape — employee data is subject to GDPR and data residency requirements. The governance framework operates across four dimensions:

| Dimension | Scope |
|---|---|
| **Landscape Strategy** | BTP account topology: Global Account → Directories → Subaccounts by workstream and environment |
| **Security Strategy** | Authentication (OAuth 2.0, SAML 2.0, Client Certificates), API Management as a mandatory gateway for inbound traffic, PII masking in message logs |
| **Onboarding Strategy** | Documented team onboarding process; RBAC role collections for all BTP services; no ad-hoc access grants |
| **Cost Management** | Consumption-based CPEA model; entitlement buffers maintained at Global Account level; per-subaccount service catalogues |

Key governance rules enforced across all H2R BTP subaccounts:

- Every Global Account must have a minimum of **2 Administrators** (prevents admin lockout)
- Every service subscription must have a minimum of **2 Role Collections** assigned
- A Global Services Catalogue and per-subaccount Local Services Catalogue must be maintained
- All user access is granted via a documented User Onboarding process — no exceptions
- Naming conventions enforced for subaccounts, subdomains, iFlows, security credentials, and CF Spaces

### BTP Infrastructure Management

Infrastructure management covers the operational lifecycle of the BTP platform:

- **BTP Structuring** — Subaccount topology design, entitlement and quota management, and IaaS tooling (Terraform for infrastructure-as-code)
- **DevOps & Release Management** — cTMS for controlled transport of Integration Suite artifacts across DEV → QA → PRD; GitHub for version control and backup of iFlow content packages
- **Operations** — Cloud ALM for real-time monitoring of integration errors and proactive alerting
- **Audit Logging** — All BTP audit logs routed from the SAP BTP Audit Log Service to enterprise SIEM systems (Splunk, Microsoft Sentinel)
- **Access Control** — IAS as the single, centralised proxy Identity Provider for all BTP subaccounts; Azure AD as the upstream corporate directory

### SSO End-to-End Process

The SSO flow in a modern H2R landscape follows this chain:

```
Employee → Corporate IdP (Azure AD)
         → SAP IAS (Proxy)
         → SAP Application (SuccessFactors, BTP, Build Work Zone)
```

IAS sits at the centre — it acts as a **proxy** between the corporate identity (Azure AD/ADFS) and all SAP applications. This means employees authenticate once with their corporate credentials and access all SAP HR systems without separate logins. The SSO design covers SAML 2.0 federation, OIDC-based flows, and the OAuth 2.0 SAML Bearer Assertion pattern used for SuccessFactors API access.

---

## 02 SAP Services

Each service document covers architecture, design decisions, procurement/licensing, step-by-step configuration, and common pitfalls.

### SAP Business Technology Platform (BTP)

**File:** `02 SAP Services/00 SAP_BTP_Hire_to_Retire.md`

Covers the foundational BTP infrastructure for H2R:

- Global Account structure and Directory hierarchy
- Subaccount design: minimum DEV, QA, PRD separation aligned to the SuccessFactors data centre region
- Entitlement management: Integration Suite plans (Standard vs Enterprise Edition), Connectivity, Destination, XSUAA, IAS, Alert Notification
- Cloud Connector: installation, system mapping, RFC and HTTP resource access control, HA Master-Shadow configuration, and certificate management
- Destination configuration for SuccessFactors OData (OAuth2SAMLBearerAssertion), on-premise SAP ERP HCM (RFC via Cloud Connector), and SFTP
- Trust Configuration: step-by-step IAS trust establishment, SAML assertion structure, shadow user creation, and role collection mapping via IAS groups
- Security layers: network (TLS, SCC outbound-only tunnel), application (RBAC, MFA), and data (PGP encryption, message log PII controls)
- Common pitfalls with resolutions: immutable subaccount subdomains, entitlement quota exhaustion, admin lockout, dual-IdP risk

### SAP Identity Authentication Service (IAS) & Identity Provisioning Service (IPS)

**File:** `02 SAP Services/01 SAP_IAS_IPS_Hire_to_Retire.md`

**Config Details:** `02 SAP Services/01.1 IAS Config.md`

Covers the full identity lifecycle for H2R:

- IAS architecture: proxy IdP model, application registration for SuccessFactors and BTP, risk-based authentication, MFA enforcement
- IPS source and target system configuration: SF Employee Central as source, IAS and on-premise LDAP as targets
- User provisioning flow: New Hire event in SF EC → IPS job → IAS user account → BTP shadow user → role collection assignment
- Transformation rules: attribute mapping, source transformations, and special (conditional) transformations
- SSO protocol support: SAML 2.0 for SuccessFactors, OIDC for BTP and Build Work Zone, Kerberos for on-premise SSO
- Authentication types: OAuth 2.0 SAML Bearer Assertion, Client Certificate, OIDC
- Azure AD ↔ IAS federation: configuring IAS as a proxy in front of Azure AD for SuccessFactors SSO
- IPS properties reference and detailed step-by-step provisioning job configuration

### SAP Integration Suite — Hire-to-Retire

**File:** `02 SAP Services/02 SAP_Integration_Suite_Hire_to_Retire.md`

Covers the full iFlow development lifecycle for H2R replication:

- Architecture: Integration Suite as the middleware hub between SuccessFactors and on-premise SAP ERP HCM
- Standard vs custom integrations: SAP-delivered content packages (EC to ERP replication, Onboarding, Payroll) vs custom-built iFlows
- SuccessFactors Adapter configuration: Compound Employee API, Intelligent Services (event-driven), OData V2/V4
- Authentication: OAuth 2.0 SAML Bearer Assertion for all SF connections (no Basic Auth in production)
- Certificates and security materials: keystore management, credential store, OAuth client configuration
- Groovy scripting patterns and XSLT mapping for data transformation
- Error and exception handling: dead letter queues, alert notification integration, retry strategies
- Modularity and reusability: local integration process design, externalised parameters, content package governance
- Monitoring: message processing logs, MPL retention policies, PII-safe logging practices

### SAP Cloud ALM — Integration Monitoring

**File:** `02 SAP Services/03 SAP_Cloud_ALM_Hire_to_Retire.md`

Covers Cloud ALM as the operations hub for H2R monitoring:

- Procurement: Cloud ALM is included at no additional cost with any CPEA or BTPEA contract
- Landscape registration: connecting Integration Suite, SuccessFactors, and on-premise ERP systems to Cloud ALM
- Integration & Exception Monitoring configuration: scenario setup, health indicator thresholds, alert rules
- Real-time monitoring of H2R integration scenarios: employee hire, transfer, termination flows
- Alerting: email and Microsoft Teams notifications for integration failures
- End-to-end scenario monitoring across system boundaries (SF → Integration Suite → ERP HCM)
- Operations runbook: daily health check procedures, incident response steps

### SAP Cloud Transport Management (cTMS)

**File:** `02 SAP Services/04 SAP_Transport_Management_Hire_to_Retire.md`

**Config Details:** `02 SAP Services/04.1 TMS Config.md`

Covers controlled promotion of Integration Suite artifacts across landscapes:

- Architecture: cTMS as a central transport orchestration layer between DEV, QA, and PRD Integration Suite tenants
- Transport nodes and transport routes configuration
- Integration Suite export configuration: connecting the Content Agent Service to cTMS
- End-to-end transport workflow: exporting iFlow packages from DEV → transport request in cTMS → import to QA → approval → import to PRD
- Externalised parameters: managing environment-specific values (URLs, credentials) across transport targets without re-configuration
- CI/CD integration: using cTMS APIs in a GitHub Actions pipeline for automated promotion
- Governance: mandatory transport approval gates for PRD imports

### SAP BTP Audit Log & SIEM Integration

**File:** `02 SAP Services/05 SAP_BTP_Audit_Management_SIEM.md`

Covers the audit log architecture and SIEM routing for compliance:

- SAP BTP Audit Log Service: what events are captured (configuration changes, security events, data access)
- Service plans and entitlements: `standard` vs `premium` plan
- Audit Log Viewer: browsing audit logs in the BTP Cockpit
- Audit Log Retrieval API: OAuth-secured API for programmatic log retrieval
- SIEM integration patterns: push (Integration Suite iFlow forwards logs) vs pull (SIEM agent polls the API)
- Step-by-step Splunk integration: HTTP Event Collector (HEC) configuration, index and sourcetype setup, dashboard queries
- Security and compliance considerations: log retention periods, PII handling in audit events, SOC 2 alignment

---

## 03 SAP HR Transformation

While Strategy and SAP Services cover the cloud-side integration and platform layer, this pillar covers the **on-premise core** — building and configuring SAP ECC / S/4HANA HCM from a blank system through to a working Hire-to-Retire lifecycle, and then extending that core into a hybrid landscape with SAP SuccessFactors. Content is organized into two streams: a phased, stage-by-stage configuration build (`HR Data Migration`) and a topic-by-topic reference library (`SAP HCM`) covering structures, SuccessFactors integration patterns, and migration/replication artifacts.

### HR Data Migration — Phased Configuration Build

**Folder:** `03 SAP HR Transformation/HR Data Migration/`

A stage-by-stage build of a working HCM system, where each stage depends on the one before it:

- **Phase 1 — Organization Set Up (Stages 0–10):** Enterprise Structure, Number Ranges, Payroll Area & Control Record, PINCH & Admin Groups, Pay Scale Structure, Features (TARIF & LGMST), Organizational Management, PA-OM Integration, Personnel Administration, Employee Hiring, Time Management
- **Phase 2 — SF Integration:** Migration, Replication, and Employee Migration Scenarios — extending the on-premise build into a hybrid landscape with SAP SuccessFactors Employee Central

---

## Technology Stack

| Layer | Technology |
|---|---|
| HR Cloud | SAP SuccessFactors Employee Central, Onboarding 2.0, Recruiting |
| Integration Middleware | SAP Integration Suite (Cloud Integration + API Management) |
| Identity & SSO | SAP IAS (Identity Authentication), SAP IPS (Identity Provisioning) |
| Platform | SAP Business Technology Platform (BTP), Cloud Foundry |
| On-Premise HR Core | SAP ERP HCM (PA/OM/Payroll/Time), SAP S/4HANA HCM (H4S4 compatibility scope) |
| Corporate Directory | Microsoft Azure AD / Microsoft Entra ID |
| DevOps | SAP Cloud Transport Management (cTMS), GitHub |
| Monitoring | SAP Cloud ALM |
| Security & Audit | SAP BTP Audit Log Service, Splunk, Microsoft Sentinel |
| Connectivity | SAP Cloud Connector |

---

## Key Design Decisions

These are the non-obvious decisions that shape the entire H2R landscape and are documented with rationale in each product file:

**IAS must replace the default SAP ID Service in all production subaccounts.** The default `accounts.sap.com` IdP does not support corporate SSO, MFA enforcement, or new hire activation email workflows.

**API Management is mandatory for all inbound traffic.** Direct calls to Integration Suite without an API Management gateway bypass rate limiting, threat detection, and centralised token validation.

**Cloud Connector HA (Master + Shadow) is required for production.** H2R data replication is business-critical — a single Cloud Connector instance is a single point of failure.

**OAuth 2.0 SAML Bearer Assertion is the required authentication type for all SuccessFactors API connections.** Basic Authentication is deprecated by SAP and must not be used in any production integration.

**All BTP audit logs must be routed to the enterprise SIEM.** BTP subaccount access, configuration changes, and data read events constitute security-relevant audit evidence required for GDPR and SOC 2 compliance.

---

## SAP Reference Links

| Topic | Link |
|---|---|
| SAP BTP Documentation | [help.sap.com/docs/btp](https://help.sap.com/docs/btp) |
| SAP Integration Suite | [help.sap.com/docs/sap-integration-suite](https://help.sap.com/docs/sap-integration-suite) |
| SAP Identity Authentication | [help.sap.com/docs/identity-authentication](https://help.sap.com/docs/identity-authentication) |
| SAP Identity Provisioning | [help.sap.com/docs/identity-provisioning](https://help.sap.com/docs/identity-provisioning) |
| SAP Cloud ALM | [help.sap.com/docs/cloud-alm](https://help.sap.com/docs/cloud-alm) |
| SAP Cloud Transport Management | [help.sap.com/docs/cloud-transport-management](https://help.sap.com/docs/cloud-transport-management) |
| SAP BTP Audit Log Service | [help.sap.com — Audit Log Retrieval API](https://help.sap.com/docs/btp/sap-business-technology-platform/audit-log-retrieval-api-usage-for-the-cloud-foundry-environment) |
| SAP Architecture Center | [architecture.learning.sap.com](https://architecture.learning.sap.com) |
| SAP Enterprise Architecture Framework | [SAP EA Framework Guide](https://help.sap.com/docs/SAP_ENTERPRISE_ARCHITECTURE_FRAMEWORK/60bc20e6e0a24426a817705bcb415220/f4d6d16c2a844c26a28157251ea288cb.html) |
| SuccessFactors Implementation Design Principles | [SAP Community — SF IDP](https://pages.community.sap.com/topics/successfactors/implementation-design-principles) |
| SAP Community Blogs | [community.sap.com](https://community.sap.com/t5/all-sap-community-blogs/ct-p/all-blogs) |
| SAP Developers | [developers.sap.com](https://developers.sap.com) |
| SAP Learning Hub | [learning.sap.com](https://learning.sap.com) |

---

## About This Library

Built from hands-on experience delivering SAP HR Transformation programmes — spanning both the cloud integration layer and the on-premise HCM core. The content reflects real implementation decisions, common pitfalls encountered in the field, and the governance patterns needed to operate a production H2R landscape safely and sustainably.

Contributions, corrections, and additional SAP product documentation are welcome via pull request.

---

*Scope: Hire-to-Retire | Platform: SAP BTP Cloud Foundry | Last Updated: June 2026*
