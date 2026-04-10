# Secure Azure OpenAI Architecture

> **Audience:** Cloud architects, platform engineers, security engineers  
> **Azure Services:** Azure OpenAI Service, Azure API Management, Azure Virtual Network, Azure Private DNS, Azure Key Vault, Microsoft Entra ID, Azure Monitor

---

## Introduction

Azure OpenAI Service provides access to OpenAI's GPT, Embeddings, and DALL-E models within the Azure trust boundary. Deploying it securely requires attention to network isolation, identity and access control, data protection, and observability.

This document provides a prescriptive secure architecture for Azure OpenAI deployments in enterprise environments.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Azure Subscription                           │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     Virtual Network                          │   │
│  │                                                              │   │
│  │  ┌─────────────┐    ┌──────────────┐    ┌────────────────┐  │   │
│  │  │  App Layer  │───▶│ Azure APIM   │───▶│ Private        │  │   │
│  │  │  (App Svc / │    │ (AI Gateway) │    │ Endpoint       │  │   │
│  │  │   AKS)      │    │              │    │ (Azure OpenAI) │  │   │
│  │  └─────────────┘    └──────────────┘    └───────┬────────┘  │   │
│  │         │                  │                    │            │   │
│  │  Managed Identity    APIM MI Auth        Azure OpenAI        │   │
│  │         │                  │             Service             │   │
│  │         ▼                  ▼                                  │   │
│  │  ┌─────────────┐    ┌──────────────┐                        │   │
│  │  │  Azure Key  │    │ Azure AI     │                        │   │
│  │  │  Vault      │    │ Content      │                        │   │
│  │  │  (secrets)  │    │ Safety       │                        │   │
│  │  └─────────────┘    └──────────────┘                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Observability: Azure Monitor · Log Analytics · Sentinel     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Security Design Principles

1. **No public endpoint exposure** — Azure OpenAI is accessed only through a private endpoint; public network access is disabled.
2. **No API key authentication in application code** — All authentication uses managed identities and Entra ID tokens.
3. **All traffic through a gateway** — Azure API Management acts as an AI gateway, enforcing policies before any request reaches Azure OpenAI.
4. **Defense in depth for inputs and outputs** — Azure AI Content Safety screens both prompts and completions.
5. **Full observability** — All API calls are logged to a central Log Analytics workspace.

---

## Network Architecture

### Private Endpoint

Deploy Azure OpenAI with a private endpoint inside the application virtual network:

```
Azure OpenAI (disable public access)
  └── Private Endpoint --> VNet Subnet --> Private DNS Zone (privatelink.openai.azure.com)
```

**Key settings:**
- Set `publicNetworkAccess: Disabled` on the Azure OpenAI resource
- Create a Private DNS Zone `privatelink.openai.azure.com` and link it to all consuming VNets
- Use a dedicated subnet for private endpoints with no other resources

### Network Security Groups

Apply NSG rules to the private endpoint subnet:
- **Inbound:** Allow only from app layer subnets and APIM subnet on HTTPS (443)
- **Outbound:** Deny all by default; allow only necessary egress

### Azure API Management (AI Gateway)

Deploy Azure API Management in the same VNet (internal mode) as the primary entry point for all LLM requests:

- Configure APIM to authenticate to Azure OpenAI using its **system-assigned managed identity** (assign `Cognitive Services OpenAI User` role)
- Apply APIM policies for:
  - Rate limiting (tokens per minute, requests per minute)
  - Input/output content safety checks (call Azure AI Content Safety API inline)
  - Request/response logging to Azure Monitor
  - JWT validation for client authentication
  - IP filtering for known client networks

---

## Identity and Access Control

| Component | Authentication Method | Azure Role |
|---|---|---|
| Application → APIM | Entra ID OAuth 2.0 (app registration) | APIM subscriber |
| APIM → Azure OpenAI | System-assigned managed identity | `Cognitive Services OpenAI User` |
| Application → Key Vault | User-assigned managed identity | `Key Vault Secrets User` |
| Administrators → Azure OpenAI | Entra ID + PIM just-in-time | `Cognitive Services Contributor` (time-limited) |

### Key Vault Integration

- Store the Azure OpenAI endpoint URL and any configuration in Azure Key Vault (not in app settings)
- **Never** store Azure OpenAI API keys in code, configuration files, or environment variables
- If API keys must be used (legacy scenarios), rotate them on a schedule using Key Vault rotation policies

---

## Content Safety

### Azure AI Content Safety Integration

Integrate Azure AI Content Safety inline in the APIM policy pipeline:

```xml
<!-- APIM inbound policy snippet -->
<send-request mode="new" response-variable-name="safety-check">
  <set-url>https://{content-safety-endpoint}/contentsafety/text:analyze</set-url>
  <set-method>POST</set-method>
  <set-body>{ "text": "@(context.Request.Body.As<string>())", "categories": ["Hate","Sexual","Violence","SelfHarm"] }</set-body>
</send-request>
<choose>
  <when condition="@(((IResponse)context.Variables["safety-check"]).StatusCode != 200)">
    <return-response><set-status code="400" reason="Content policy violation"/></return-response>
  </when>
</choose>
```

### Prompt Shield

Enable the **Prompt Shield** feature in Azure AI Content Safety to detect:
- Direct prompt injection attacks
- Indirect prompt injection from documents and external content

### Output Moderation

Apply output moderation to all completions before returning to clients. Filter or block responses that contain:
- Hate speech or harmful content
- PII (names, emails, phone numbers, credit card numbers)
- Sensitive organizational information patterns

---

## Data Protection

### Data at Rest

- Azure OpenAI does not persist prompts or completions by default
- If fine-tuning datasets are stored, use Azure Blob Storage with:
  - Customer-managed keys (CMK) via Azure Key Vault
  - Soft delete enabled
  - Immutability policies for training datasets

### Data in Transit

- All traffic uses TLS 1.2+ (enforced by Azure services)
- No prompt data should traverse the public internet; all calls are through private endpoints

### Data Residency

- Deploy the Azure OpenAI resource in the required Azure region for data residency compliance
- Disable cross-region routing if data must remain in a single geography

---

## Logging and Monitoring

### Diagnostic Settings

Enable Azure OpenAI diagnostic settings to send to Log Analytics:
- `Audit` logs — all management plane operations
- `RequestResponse` logs — all inference requests and responses (enable with caution due to PII risk; apply data masking)

### Key Metrics to Monitor

| Metric | Alert Threshold | Purpose |
|---|---|---|
| `TokensPerMinute` | > 80% of quota | Quota exhaustion warning |
| `FailedRequests` | > 5% error rate | Service degradation |
| `ContentFilteredRequests` | Spike above baseline | Potential attack or misuse |
| `Latency P95` | > 10 seconds | Performance degradation |

### Microsoft Sentinel Integration

Forward Log Analytics workspace to Microsoft Sentinel for:
- Correlation with other security signals
- Anomaly detection on query patterns (potential model extraction)
- Incident response automation via Sentinel playbooks

---

## Deployment Hardening Checklist

- [ ] Public network access disabled on Azure OpenAI resource
- [ ] Private endpoint deployed in application VNet
- [ ] Private DNS zone configured and linked
- [ ] APIM deployed in internal mode within the same VNet
- [ ] APIM authenticates to Azure OpenAI via managed identity (no API key)
- [ ] Rate limiting policies configured in APIM
- [ ] Azure AI Content Safety prompt shield enabled
- [ ] Output moderation enabled
- [ ] Diagnostic logs enabled and forwarded to Log Analytics
- [ ] Azure Monitor alerts configured for key metrics
- [ ] No API keys or connection strings in application code or config files
- [ ] Customer-managed keys configured for fine-tuning storage
- [ ] Azure Defender for AI (preview) or equivalent enabled
- [ ] Least-privilege RBAC applied to all identities

---

## Next Steps

- [Secure RAG Architecture](secure-rag-architecture.md)
- [Identity and Access for AI Systems](../04-identity-access.md)
- [Monitoring and Detection](../07-monitoring-detection.md)
