# Identity and Access for AI Systems

> **Audience:** Cloud architects, security engineers, identity engineers  
> **Azure Services:** Microsoft Entra ID, Azure Managed Identity, Azure Key Vault, Azure RBAC, Microsoft Entra Privileged Identity Management (PIM)

---

## Introduction

Identity is the foundational security control for AI systems on Azure. Unlike traditional applications where identity mistakes mainly affect data access, AI systems can amplify identity failures—a compromised identity with access to an LLM and its connected tools can cause significantly more damage than the same compromise in a traditional web application.

This document covers identity patterns for AI workloads including managed identity design, OAuth flows, RBAC scoping, and just-in-time access for AI administrators.

---

## Identity Threat Landscape for AI Systems

| Threat | Impact | Example |
|---|---|---|
| Compromised API key | Full access to LLM endpoint | Hardcoded Azure OpenAI key exposed in public repo |
| Overprivileged managed identity | Lateral movement via AI agent tools | Agent identity with Contributor role on entire subscription |
| Missing authentication on AI endpoints | Unauthorized model access, cost exposure | Azure OpenAI endpoint accessible without auth |
| Insider threat via admin access | Model poisoning, data exfiltration | Unrestricted admin access to training data and model artifacts |
| Token theft | Session hijacking | Bearer token stolen from browser, replayed to AI API |

---

## Managed Identity Design

Managed identities are the recommended approach for all service-to-service authentication in Azure AI workloads. They eliminate the need to manage credentials in application code.

### Identity Segmentation

Each component in an AI system should have its **own dedicated managed identity**, not a shared one. This limits blast radius and enables fine-grained access revocation.

| Component | Identity Type | Rationale |
|---|---|---|
| Application / Orchestrator | User-assigned managed identity | Can be pre-configured, attached to multiple compute instances, and independently managed |
| AI Agent | User-assigned managed identity | Per-agent identity enables per-agent RBAC and audit logging |
| Data ingestion pipeline | User-assigned managed identity | Separate from query identity; can have write access to storage without granting query identity write access |
| Embedding job (batch) | User-assigned managed identity | Short-lived batch jobs with scoped permissions |
| APIM instance | System-assigned managed identity | Used for APIM → Azure OpenAI authentication |

### Avoid System-Assigned for Shared Identities

Use **user-assigned** managed identities for production workloads:
- They exist independently of compute resources (survive resource recreation)
- They can be pre-authorized before compute is provisioned
- They support the principle of separation of duties

### RBAC Assignment Pattern

```
Orchestrator Identity
  ├── Cognitive Services OpenAI User → Azure OpenAI resource
  ├── Search Index Data Reader       → Azure AI Search index
  └── Key Vault Secrets User         → Azure Key Vault (config only)

Ingestion Pipeline Identity
  ├── Storage Blob Data Contributor  → Source document container
  ├── Cognitive Services OpenAI User → Azure OpenAI (embeddings only)
  └── Search Index Data Contributor  → Azure AI Search index
```

**Never assign:**
- `Owner` or `Contributor` at subscription or resource group level to AI service identities
- Roles that allow role assignment (`User Access Administrator`)
- Access to resources outside the AI system's defined scope

---

## Authentication Patterns

### Application → Azure OpenAI (Managed Identity)

```python
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI

credential = DefaultAzureCredential()
token = credential.get_token("https://cognitiveservices.azure.com/.default")

client = AzureOpenAI(
    azure_endpoint="https://<your-resource>.openai.azure.com/",
    azure_ad_token=token.token,
    api_version="2024-02-01"
)
```

**Never use API keys in application code.** If you must use an API key (legacy systems), store it in Azure Key Vault and retrieve it at runtime via managed identity.

### User → Application (Entra ID OAuth 2.0)

For user-facing AI applications:

1. Register the application in Microsoft Entra ID
2. Require users to authenticate via MSAL or Entra ID-integrated login
3. Validate the JWT access token in the API layer before processing any request
4. Use the user's identity context for audit logging and access control decisions

```
User → [Entra ID Login] → JWT Token → [Application API]
                                           │
                              Validates JWT (audience, issuer, expiry)
                              Extracts user identity for audit log
```

### Service-to-Service (Client Credentials Flow)

For automated pipelines and integrations that cannot use managed identity:

1. Register a **service principal** in Entra ID with a client certificate (not client secret)
2. Rotate certificates on a schedule using Key Vault certificate management
3. Apply **Conditional Access** policies to service principals accessing AI resources
4. Scope the service principal to minimum required roles

```
Pipeline → [Client Certificate Auth] → Entra ID → Access Token → Azure OpenAI
```

---

## Least Privilege Design

### AI Service RBAC Roles Reference

| Azure Role | Grants | Use Case |
|---|---|---|
| `Cognitive Services OpenAI User` | Inference API calls (completions, embeddings) | Application, orchestrator, APIM |
| `Cognitive Services OpenAI Contributor` | Inference + fine-tuning + model deployment | MLOps pipeline, model deployment automation |
| `Cognitive Services Contributor` | Full management of resource | Administrators only (use PIM) |
| `Search Index Data Reader` | Query an Azure AI Search index | Orchestrator, query pipeline |
| `Search Index Data Contributor` | Read and write index data | Ingestion pipeline |
| `Search Service Contributor` | Manage search service configuration | Administrators only (use PIM) |
| `Key Vault Secrets User` | Read secrets | Application runtime |
| `Key Vault Secrets Officer` | Create and update secrets | Administrators, automation pipelines |

### Scope RBAC at the Narrowest Level

| Level | When to Use |
|---|---|
| **Resource** | Default for most AI service roles (e.g., specific Azure OpenAI resource) |
| **Resource Group** | Only when the identity manages multiple related resources |
| **Subscription** | Avoid; use only for platform-level management identities |

---

## Privileged Access Management

### Azure PIM for AI Administrators

All administrative access to AI infrastructure should be managed through **Microsoft Entra Privileged Identity Management (PIM)**:

- Administrators are **eligible** for privileged roles, not permanently assigned
- Activation requires **justification and MFA**
- Access is **time-limited** (default: 1–8 hours; configure based on task type)
- All activations are **audit logged**

Privileged roles to protect with PIM:

- `Cognitive Services Contributor` on Azure OpenAI resources
- `Search Service Contributor` on Azure AI Search
- `Key Vault Administrator`
- `Storage Account Contributor` on training data storage
- Azure Machine Learning `Owner` or `Contributor`

### Break-Glass Accounts

Maintain emergency access accounts for AI infrastructure that bypass PIM, following the Microsoft guidance on [emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access). These accounts should:
- Be cloud-only (not federated)
- Use strong authentication without MFA dependency on external systems
- Have their usage monitored with alerts on any sign-in

---

## Conditional Access for AI APIs

Apply Entra ID Conditional Access policies to Azure OpenAI and other AI API access:

| Policy | Condition | Control |
|---|---|---|
| Require MFA | All interactive user access to AI management portals | MFA required |
| Compliant device | Users accessing AI applications with sensitive data | Require compliant or hybrid-joined device |
| Network location | Service principals accessing AI APIs | Allow only from known IP ranges / Azure services |
| Sign-in risk | High-risk sign-in detected | Block or require step-up authentication |

---

## API Key Management (When Keys Must Be Used)

Where API keys are required (third-party integrations, legacy systems), follow these controls:

1. **Store in Azure Key Vault** — never in code, environment variables, or config files
2. **Enable Key Vault soft delete and purge protection**
3. **Rotate keys on a defined schedule** — use Key Vault rotation policies with Azure Functions
4. **Alert on key access** — configure Key Vault diagnostics and alert on unexpected secret access
5. **Limit access to secrets** using Key Vault RBAC (`Key Vault Secrets User`)
6. **Use separate keys per environment** — never share production keys with dev/test

---

## Identity Design Checklist

- [ ] All service-to-service authentication uses managed identities (no API keys in code)
- [ ] Each AI system component has its own dedicated managed identity
- [ ] RBAC roles are scoped to the minimum required level and permission set
- [ ] No AI service identities have Owner/Contributor at subscription scope
- [ ] User authentication uses Entra ID OAuth 2.0 with JWT validation
- [ ] Privileged AI admin roles are managed via PIM (eligible, not permanent)
- [ ] Conditional Access policies applied to AI API access
- [ ] API keys (if used) are stored in Key Vault with rotation policies
- [ ] Key Vault audit logging enabled and alerts configured
- [ ] Break-glass accounts documented and monitored
- [ ] Regular access reviews scheduled (quarterly minimum) for all AI service identities

---

## Next Steps

- [AI Data Governance and Protection](05-data-governance.md)
- [Monitoring and Detection](07-monitoring-detection.md)
- [Secure Azure OpenAI Architecture](03-secure-architectures/secure-azure-openai.md)
