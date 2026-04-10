# Enterprise AI Security Reference Architecture

> **Audience:** Cloud architects, CISOs, platform teams  
> **Purpose:** End-to-end secure reference architecture for enterprise AI workloads on Azure

---

## Introduction

This document synthesizes the guidance across this playbook into a complete enterprise AI security reference architecture. It is designed for organizations deploying AI at scale, including multiple AI use cases, multi-tenant environments, and regulated workloads.

Use this architecture as a starting point and adapt it to your organization's specific requirements, risk profile, and existing Azure landing zone.

---

## Architecture Overview

The reference architecture follows four security zones:

1. **Client Zone** — End users and consuming applications
2. **AI Gateway Zone** — Entry point for all AI traffic; policy enforcement
3. **AI Services Zone** — Core AI capabilities (inference, retrieval, orchestration)
4. **Data and Model Zone** — Sensitive data and model artifacts

---

## Full Architecture Diagram

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║                     ENTERPRISE AI SECURITY REFERENCE ARCHITECTURE            ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║                                                                               ║
║  ZONE 1: CLIENT                                                               ║
║  ┌─────────────────────────────────────────────────────────────────────────┐  ║
║  │  [Web App]  [Mobile App]  [Internal Tools]  [API Consumers]  [Agents]  │  ║
║  └────────────────────────────┬────────────────────────────────────────────┘  ║
║                               │ Entra ID OAuth 2.0                           ║
║  ZONE 2: AI GATEWAY                                                           ║
║  ┌─────────────────────────────────────────────────────────────────────────┐  ║
║  │  ┌──────────────────┐   ┌───────────────────┐   ┌──────────────────┐   │  ║
║  │  │  Azure Front Door│   │  Azure API         │   │  Azure AI Content│   │  ║
║  │  │  + WAF           │──▶│  Management        │──▶│  Safety          │   │  ║
║  │  │  (DDoS, geo-     │   │  (Auth, rate limit,│   │  (Prompt Shield, │   │  ║
║  │  │   filter, edge)  │   │   logging, routing)│   │   output filter) │   │  ║
║  │  └──────────────────┘   └───────────────────┘   └──────────────────┘   │  ║
║  └────────────────────────────┬────────────────────────────────────────────┘  ║
║                               │ Managed Identity (within VNet)               ║
║  ZONE 3: AI SERVICES                                                          ║
║  ┌─────────────────────────────────────────────────────────────────────────┐  ║
║  │  ┌────────────────┐  ┌────────────────┐  ┌───────────────────────────┐  │  ║
║  │  │  AI Orchestrator│  │  Azure OpenAI  │  │  Azure AI Search          │  │  ║
║  │  │  (Container App│  │  Service        │  │  (Vector Index,           │  │  ║
║  │  │  / AKS)        │  │  (Private EP)   │  │   Private EP, CMK,        │  │  ║
║  │  │                │─▶│  Managed Identity│  │   Security Trimming)     │  │  ║
║  │  │  Agent Runtime │  │  Auth           │  │                           │  │  ║
║  │  │  (Tool Router, │  └────────────────┘  └───────────────────────────┘  │  ║
║  │  │  Human-in-loop)│                                                      │  ║
║  │  └────────────────┘  ┌────────────────┐  ┌───────────────────────────┐  │  ║
║  │                       │  Azure Cache   │  │  Azure Logic Apps          │  │  ║
║  │                       │  for Redis     │  │  (Approval Workflows)      │  │  ║
║  │                       │  (Agent Memory)│  │                           │  │  ║
║  │                       └────────────────┘  └───────────────────────────┘  │  ║
║  └────────────────────────────┬────────────────────────────────────────────┘  ║
║                               │ Managed Identity (within VNet)               ║
║  ZONE 4: DATA AND MODEL                                                       ║
║  ┌─────────────────────────────────────────────────────────────────────────┐  ║
║  │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐  │  ║
║  │  │  Azure Blob  │  │  Azure SQL / │  │  Azure Key │  │  Azure ML    │  │  ║
║  │  │  Storage     │  │  Cosmos DB   │  │  Vault     │  │  (Model      │  │  ║
║  │  │  (Docs, CMK) │  │  (Structured │  │  (Secrets, │  │  Registry,  │  │  ║
║  │  │              │  │   data, CMK) │  │   CMK,     │  │  Training    │  │  ║
║  │  └──────────────┘  └──────────────┘  │   Rotation)│  │  Artifacts)  │  │  ║
║  │                                       └────────────┘  └──────────────┘  │  ║
║  │  ┌──────────────────────────────────────────────────────────────────┐   │  ║
║  │  │  Microsoft Purview (Data Map, Classification, DLP, Lineage)      │   │  ║
║  │  └──────────────────────────────────────────────────────────────────┘   │  ║
║  └─────────────────────────────────────────────────────────────────────────┘  ║
║                                                                               ║
║  CROSS-CUTTING: SECURITY AND OBSERVABILITY                                    ║
║  ┌─────────────────────────────────────────────────────────────────────────┐  ║
║  │  Azure Monitor · Log Analytics · Microsoft Sentinel · Defender for Cloud│  ║
║  │  Entra ID PIM · Azure Policy · Microsoft Purview · GitHub Adv. Security │  ║
║  └─────────────────────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

## Zone-by-Zone Security Controls

### Zone 1: Client Zone

| Control | Implementation |
|---|---|
| Authentication | Microsoft Entra ID with Conditional Access (MFA, device compliance) |
| Authorization | OAuth 2.0 with JWT validation; claims-based access control |
| AI disclosure | Users informed when interacting with AI-generated content |
| Client-side security | MSAL-based token acquisition; no credential storage in client |

### Zone 2: AI Gateway Zone

| Control | Implementation |
|---|---|
| DDoS protection | Azure Front Door with Azure DDoS Protection Standard |
| Web application firewall | Azure WAF with OWASP rule set |
| API authentication | APIM validates Entra ID JWT tokens on all requests |
| Rate limiting | Per-user, per-application TPM/RPM quotas in APIM |
| Prompt shield | Azure AI Content Safety prompt injection detection |
| Output moderation | Azure AI Content Safety output filtering |
| Request logging | APIM diagnostic logs to Log Analytics |
| Routing | APIM routes to appropriate AI service based on use case |

### Zone 3: AI Services Zone

| Control | Implementation |
|---|---|
| Network isolation | All services in VNet; private endpoints; no public access |
| Service authentication | Managed identity; Entra ID RBAC; no API keys |
| Orchestration security | Container App with dedicated managed identity; tool allowlist |
| Agent safety | Human-in-the-loop via Logic Apps; tool input validation |
| Vector search security | Security trimming; index partitioning; Entra ID RBAC |
| Memory security | Encrypted Redis cache; sanitization before context injection |

### Zone 4: Data and Model Zone

| Control | Implementation |
|---|---|
| Encryption at rest | CMK for all regulated data stores; Key Vault HSM |
| Data governance | Purview classification; sensitivity labels; DLP policies |
| Model integrity | Immutable storage; hash verification; signed artifacts |
| Secret management | All secrets in Key Vault with rotation policies |
| Data lineage | AML dataset versioning; Purview lineage |

---

## Network Architecture

### Virtual Network Design

```
VNet: 10.0.0.0/16
├── Subnet: gateway (10.0.1.0/24)
│   ├── Azure API Management
│   └── Azure Front Door origin (private link)
├── Subnet: app (10.0.2.0/24)
│   ├── Container Apps / AKS (AI Orchestrator)
│   └── Azure Logic Apps
├── Subnet: private-endpoints (10.0.3.0/24)
│   ├── Azure OpenAI private endpoint
│   ├── Azure AI Search private endpoint
│   ├── Azure Key Vault private endpoint
│   ├── Azure Storage private endpoints
│   └── Azure SQL private endpoint
└── Subnet: monitoring (10.0.4.0/24)
    └── Log Analytics workspace linked resources
```

### Private DNS Zones Required

| Service | Private DNS Zone |
|---|---|
| Azure OpenAI | `privatelink.openai.azure.com` |
| Azure AI Search | `privatelink.search.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure SQL Database | `privatelink.database.windows.net` |
| Azure Container Registry | `privatelink.azurecr.io` |

---

## Identity Architecture

```
Microsoft Entra ID Tenant
├── User identities (Entra ID with Conditional Access + MFA)
├── Application registrations (OAuth 2.0 for client apps)
└── Managed Identities
    ├── apim-identity          → Cognitive Services OpenAI User
    ├── orchestrator-identity  → Cognitive Services OpenAI User
    │                          → Search Index Data Reader
    │                          → Key Vault Secrets User
    ├── ingestion-identity     → Storage Blob Data Contributor (source containers)
    │                          → Cognitive Services OpenAI User (embeddings)
    │                          → Search Index Data Contributor
    └── agent-identity         → [Tool-specific scoped roles]
                               → Key Vault Secrets User
```

---

## Data Flow Summary

### User Query Flow (RAG + LLM)

```
1. User authenticates via Entra ID → receives JWT
2. Request arrives at Azure Front Door / WAF
3. APIM validates JWT, applies rate limiting
4. APIM calls Azure AI Content Safety (prompt shield)
5. If safe: APIM forwards to AI Orchestrator (Container App)
6. Orchestrator queries Azure AI Search (managed identity)
7. Security trimming applied → only authorized chunks returned
8. Orchestrator screens retrieved chunks for indirect injection
9. Orchestrator assembles prompt + context
10. Orchestrator calls Azure OpenAI (managed identity, private endpoint)
11. Azure OpenAI returns completion
12. Orchestrator calls AI Content Safety (output moderation)
13. Groundedness check applied
14. Response returned to user with source citations
15. All steps logged to Log Analytics
```

### AI Agent Action Flow

```
1. Trigger (user or automated) → authenticated via Entra ID
2. Agent orchestrator receives task
3. LLM plans steps using Azure OpenAI
4. For each planned action:
   a. Tool name validated against allowlist
   b. Parameters validated and sanitized
   c. Risk tier assessed
   d. If high/critical risk → Logic Apps approval workflow
   e. Human approves or rejects within SLA
   f. If approved → tool executed with agent managed identity
   g. Action logged to append-only audit log
5. Final result returned to trigger
6. Full session log written to Log Analytics
```

---

## Deployment Guidance

### Infrastructure as Code

Deploy this architecture using:
- **Azure Bicep** or **Terraform** for infrastructure provisioning
- **Azure Policy** assignments as code for compliance enforcement
- **GitHub Actions** or **Azure DevOps** for CI/CD pipelines

All infrastructure changes should go through:
1. Pull request review
2. Security scanning (IaC security scan — Checkov, Terrascan)
3. Azure deployment validation (`az deployment validate`)
4. Production deployment requires approval gate

### Landing Zone Integration

This architecture should be deployed within an [Azure Landing Zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/) structure:

- Use the **Platform Landing Zone** for shared services (Log Analytics, Sentinel, Purview, Key Vault HSM)
- Deploy AI workloads in an **Application Landing Zone** with spoke VNet peered to hub
- Use Azure Policy at the management group level to enforce baseline security controls across all AI workloads

---

## Compliance Coverage Summary

| Compliance Requirement | Architecture Control |
|---|---|
| Data residency | Regional deployment; no cross-region routing |
| Data encryption | CMK on all regulated stores |
| Access control | Entra ID RBAC; PIM for privileged access |
| Audit logging | Log Analytics; immutable audit logs |
| Network security | Private endpoints; VNet isolation; WAF |
| Vulnerability management | Defender for Cloud; container scanning |
| Incident response | Sentinel; defined runbooks |
| AI transparency | Content disclosures; model cards |
| Human oversight | Human-in-the-loop for consequential actions |
| Data governance | Microsoft Purview |

---

## Architecture Decision Records

| Decision | Choice | Rationale |
|---|---|---|
| AI gateway | Azure API Management | Centralized policy enforcement, observability, and routing |
| Content safety | Azure AI Content Safety | Native integration, prompt shield, output moderation |
| Secret storage | Azure Key Vault | Managed rotation, audit logging, HSM option |
| Vector search | Azure AI Search | Native Entra ID RBAC, security trimming, private endpoint |
| Authentication | Managed identity (service-to-service) | Eliminates credential management; audit-logged |
| Network isolation | Private endpoints for all AI services | No AI traffic traverses public internet |
| Agent approval | Azure Logic Apps | No-code approval workflows; audit trail built-in |

---

## Next Steps

1. **Start with identity** — Implement managed identities and disable API key authentication
2. **Add network isolation** — Deploy private endpoints for Azure OpenAI and AI Search
3. **Implement the gateway** — Deploy APIM with content safety integration
4. **Enable observability** — Connect all services to Log Analytics and activate Sentinel
5. **Implement data governance** — Register AI data stores in Microsoft Purview
6. **Conduct threat modeling** — Use the [threat modeling guide](01-threat-modeling.md) to identify gaps
7. **Red team** — Use PyRIT to test your AI systems before production deployment

---

## Related Documents

- [Threat Modeling AI Systems](01-threat-modeling.md)
- [OWASP LLM Top 10 Mitigations](02-owasp-llm-top10.md)
- [Secure Azure OpenAI Architecture](03-secure-architectures/secure-azure-openai.md)
- [Secure RAG Architecture](03-secure-architectures/secure-rag-architecture.md)
- [Secure AI Agent Architecture](03-secure-architectures/secure-ai-agent-architecture.md)
- [Identity and Access for AI Systems](04-identity-access.md)
- [AI Data Governance and Protection](05-data-governance.md)
- [Model Security](06-model-security.md)
- [Monitoring and Detection](07-monitoring-detection.md)
- [Responsible AI and Governance](08-responsible-ai.md)
