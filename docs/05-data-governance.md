# AI Data Governance and Protection

> **Audience:** Cloud architects, data engineers, security engineers, compliance officers  
> **Azure Services:** Microsoft Purview, Azure Blob Storage, Azure SQL Database, Azure Key Vault, Azure Policy, Microsoft Defender for Cloud, Azure Monitor

---

## Introduction

AI systems process, transform, and generate data at scale. This creates unique governance challenges: training datasets may contain sensitive information, RAG pipelines ingest enterprise knowledge bases, and LLMs can surface data that was never intended to be accessible to a given user.

This document covers data classification, leakage prevention, governance frameworks, and Azure-native controls for protecting data in AI workloads.

---

## AI Data Leakage Risk Taxonomy

| Leakage Type | Description | Example |
|---|---|---|
| **Training data memorization** | Model memorizes and can regurgitate training data | Fine-tuned model reveals PII from training dataset |
| **RAG over-retrieval** | Retrieval system returns documents the user should not see | HR chatbot returns salary data from another employee |
| **Prompt/response logging exposure** | Logged prompts/responses contain sensitive data accessible to unauthorized parties | Support team accesses logs containing user health data |
| **System prompt disclosure** | System prompt containing business logic or confidential instructions is leaked | User extracts detailed pricing rules from system prompt |
| **Cross-tenant leakage** | Multi-tenant system exposes one tenant's data to another | RAG vector store not partitioned per tenant |
| **Output inference** | User infers sensitive data from model responses | Attacker determines if a target person is in a database via membership inference |

---

## Data Classification Framework

Before deploying AI systems, classify all data that will be used in training, fine-tuning, RAG, or prompt assembly.

### Classification Tiers

| Level | Definition | Examples | AI Usage Controls |
|---|---|---|---|
| **Public** | Approved for public release | Marketing content, public documentation | Unrestricted use in AI systems |
| **Internal** | For internal business use | Process documentation, internal policies | OK for internal AI systems; not external |
| **Confidential** | Sensitive business data | Financial data, contracts, strategic plans | Restricted RAG access; audit logging required |
| **Highly Confidential** | Regulated or highest-sensitivity data | PII, PHI, PCI data, credentials | Excluded from RAG by default; special approval required |

### Microsoft Purview for AI Data Classification

Use **Microsoft Purview** to:

1. **Discover** sensitive data across Azure storage, databases, and data lakes
2. **Classify** data using built-in classifiers (PII, PHI, PCI, financial data)
3. **Apply sensitivity labels** that follow data through AI pipelines
4. **Monitor** data access and movement with Purview audit logs
5. **Enforce** data handling policies via DLP rules

#### Purview Integration with AI Data Stores

| Data Store | Purview Integration |
|---|---|
| Azure Blob Storage | Register as a Purview data source; enable auto-scan on ingestion |
| Azure AI Search | Register via Purview data map; classify indexed content |
| Azure SQL Database | Register and scan for sensitive columns |
| Azure Data Lake Storage | Register with scheduled scans; apply column-level classification |

---

## Data Protection Controls

### Encryption

| Data State | Control | Implementation |
|---|---|---|
| **At rest — storage** | AES-256 encryption | Enabled by default; use CMK for regulated workloads |
| **At rest — search index** | Azure AI Search encryption | Enable CMK with Azure Key Vault |
| **At rest — ML model artifacts** | AML workspace encryption | Enable CMK on AML workspace storage account |
| **In transit** | TLS 1.2+ | Enforced by all Azure AI services; verify no TLS 1.0/1.1 in network policies |
| **In use** | Confidential computing | Azure confidential VMs for sensitive ML training jobs |

### Customer-Managed Keys (CMK)

For workloads subject to regulatory requirements (HIPAA, PCI-DSS, GDPR):

- Enable CMK on Azure OpenAI fine-tuning storage
- Enable CMK on Azure AI Search indexes
- Enable CMK on Azure Machine Learning workspaces
- Store CMK in Azure Key Vault with hardware security module (HSM) backing
- Configure Key Vault access policies to only allow the resource's managed identity

### Azure Key Vault Best Practices

- Enable **purge protection** to prevent accidental or malicious key deletion
- Enable **soft delete** with a 90-day retention period
- Use **Key Vault Managed HSM** for FIPS 140-2 Level 3 key protection
- Enable **Key Vault diagnostic logging** to detect unauthorized key access
- Implement key rotation using **Key Vault rotation policies** with Azure Function triggers

---

## Data Loss Prevention for AI Systems

### Microsoft Purview DLP for AI

Configure Purview DLP policies to:
- **Block upload** of highly confidential files to AI data ingestion endpoints
- **Alert** when files classified above a threshold are accessed by AI service identities
- **Audit** all data movement involving AI pipelines

### Application-Layer DLP

Implement DLP at the application layer for prompt and response pipelines:

```
User Prompt → [PII Detection / Redaction] → Sanitized Prompt → Azure OpenAI
                      │
              Detect: Names, SSNs, emails,
              credit card numbers, phone
              numbers, health identifiers
```

Use Azure AI Content Safety or custom NER (Named Entity Recognition) models to:
1. **Detect PII** in user prompts before sending to the model
2. **Redact or pseudonymize** PII before including in context
3. **Filter PII from responses** before returning to users
4. **Log detected PII events** for compliance reporting (without logging the actual PII)

### Prompt and Response Logging Controls

| Log Type | Data Risk | Control |
|---|---|---|
| Full prompt logs | May contain user PII, sensitive queries | Apply PII redaction before logging; restrict log access |
| Full response logs | May contain sensitive or hallucinated data | Apply content screening before logging |
| Metadata-only logs | Low risk | Preferred for operational monitoring |
| Audit logs (user identity + action) | Identity data | Retain per compliance requirements; restrict access |

**Recommendation:** Log metadata and flagged events by default. Implement full prompt/response logging only when required for compliance, with appropriate access controls and data retention limits.

---

## Data Residency and Sovereignty

| Requirement | Azure Control |
|---|---|
| Data must stay in a specific country | Deploy all AI resources in the required Azure region; disable geo-redundancy |
| Cross-border transfer restrictions | Use Azure Private Link to prevent data from traversing public internet |
| Sovereign cloud requirements | Use Azure Government, Azure China, or Azure operated by 21Vianet as appropriate |
| Contractual data residency | Use Azure Policy to enforce resource deployment to approved regions |

### Azure Policy for Data Residency

```json
// Example: Deny deployment of Azure OpenAI outside approved regions
{
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "equals": "Microsoft.CognitiveServices/accounts" },
        { "field": "location", "notIn": ["eastus", "westeurope"] }
      ]
    },
    "then": { "effect": "Deny" }
  }
}
```

---

## Training Data Governance

### Data Lifecycle for AI Training

```
Data Collection → Classification → Quality Review → Approval Gate → Training → Audit Trail
      │               │                 │                │              │           │
  Document        Purview scan      Human review    Signed-off     Versioned   Immutable
  source and      for PII/PHI       for accuracy    by data        dataset     lineage
  lineage         and sensitivity   and bias        steward        in AML      record
```

### Controls at Each Stage

1. **Data Collection:** Record provenance (source system, collection date, collection method)
2. **Classification:** Auto-classify using Purview; manual review for borderline cases
3. **Quality Review:** Statistical checks for anomalies (distribution shifts, injection attempts)
4. **Approval Gate:** Data steward sign-off required before data enters training pipeline
5. **Training:** Use versioned, immutable datasets; store checksums
6. **Audit Trail:** Maintain lineage from raw data to trained model in AML

### Right to Erasure (GDPR Article 17) for AI

Handling data erasure requests when data has been used in AI training is technically complex:

- **Preferred:** Exclude PII from training data entirely using Purview-based filtering
- **If PII is in training data:** Maintain a record of which training runs used which data; be prepared to retrain without the erased data upon request
- **For RAG systems:** Implement a document purge API that removes documents and their vector embeddings from the index

---

## Data Governance Roles and Responsibilities

| Role | Responsibilities |
|---|---|
| **Data Owner** | Approves data use in AI systems; accountable for classification accuracy |
| **Data Steward** | Manages day-to-day governance; approves training datasets |
| **AI Engineer** | Implements technical controls; ensures pipelines respect classification labels |
| **Security Engineer** | Monitors for leakage events; configures DLP policies |
| **Compliance Officer** | Maps regulatory requirements to governance controls; reviews audit logs |

---

## Compliance Mapping

| Regulation | Key AI Data Requirement | Azure Control |
|---|---|---|
| **GDPR** | Lawful basis for training data use; right to erasure | Purview data map, consent management, data lineage |
| **HIPAA** | PHI must not be used in AI without Business Associate Agreement | Purview PHI classifiers, Azure HIPAA BAA, CMK |
| **PCI DSS** | Cardholder data must not appear in AI training or logs | Purview PCI classifiers, DLP, log redaction |
| **ISO 27001** | Information classification and handling | Purview sensitivity labels, Azure Policy |
| **NIST AI RMF** | Data quality, provenance, and bias documentation | AML dataset versioning, Purview lineage |

---

## Data Governance Checklist

- [ ] All data stores used by AI systems registered in Microsoft Purview
- [ ] Sensitivity labels applied to all data used in AI pipelines
- [ ] Highly confidential data excluded from general-purpose RAG indexes
- [ ] CMK encryption enabled for all regulated AI data stores
- [ ] Key Vault purge protection and soft delete enabled
- [ ] PII detection and redaction applied to prompts and responses
- [ ] Full prompt/response logs restricted to authorized personnel
- [ ] Azure Policy enforces approved regions for AI resource deployment
- [ ] Training data lineage recorded in Azure Machine Learning
- [ ] Data erasure procedure documented and tested
- [ ] Compliance mapping documented and reviewed annually
- [ ] Data governance roles assigned and acknowledged

---

## Next Steps

- [Model Security](06-model-security.md)
- [Monitoring and Detection](07-monitoring-detection.md)
- [Responsible AI and Governance](08-responsible-ai.md)
