# Threat Modeling AI Systems

> **Audience:** Cloud architects, security engineers, AI/ML engineers  
> **Azure Services Referenced:** Azure OpenAI, Azure AI Search, Azure Machine Learning, Microsoft Threat Modeling Tool

---

## Introduction

Threat modeling for AI systems follows the same disciplined approach used for traditional software—but the attack surface is fundamentally different. AI systems introduce new data flows, external model dependencies, natural language interfaces, and non-deterministic behaviors that require dedicated analysis.

This document explains how to apply structured threat modeling to:
- Large Language Model (LLM) applications
- Retrieval-Augmented Generation (RAG) pipelines
- AI agent systems (autonomous, multi-step reasoning)

---

## Threat Modeling Methodology

This playbook recommends a combination of two frameworks adapted for AI systems:

| Framework | Purpose |
|---|---|
| **STRIDE** | Systematic identification of threat categories across a data flow diagram |
| **MITRE ATLAS** | AI-specific tactics, techniques, and procedures (TTPs) mapped to the adversarial ML lifecycle |

Use STRIDE for structured DFD-based analysis and MITRE ATLAS to enrich findings with known AI adversarial techniques.

---

## Step 1: Define the System and Its Boundaries

Before identifying threats, fully describe the system under review:

- **Assets:** What is being protected? (model weights, training data, inference outputs, API keys, user data)
- **Entry Points:** Where does untrusted data enter? (user prompts, API calls, document uploads, tool outputs)
- **Trust Boundaries:** Where does data cross between trust levels? (user → application → Azure OpenAI API → vector store)
- **Actors:** Who interacts with the system? (end users, API consumers, administrators, external tools/agents)

### Example System Description: RAG Chatbot

> A web application accepts user questions, retrieves relevant context from an Azure AI Search index, and passes both the question and context to Azure OpenAI to generate an answer. The application is hosted in Azure App Service behind Azure API Management.

---

## Step 2: Draw the Data Flow Diagram (DFD)

Create a Level 1 DFD that captures:

- **External entities** (users, external APIs, data sources)
- **Processes** (orchestration layer, embedding service, inference endpoint)
- **Data stores** (vector database, blob storage, SQL)
- **Data flows** with direction and data type
- **Trust boundaries** (network perimeter, identity boundary)

### DFD Example: RAG Pipeline

```
[User] --> (API Gateway) --> (Orchestrator / LangChain) --> [Azure OpenAI]
                                      |
                                      v
                              (Embedding Model) --> [Azure AI Search Index]
                                      |
                                      v
                              [Document Store / Blob Storage]
```

Trust boundaries exist between:
- Public internet → API Gateway
- Orchestrator → Azure OpenAI (managed identity boundary)
- Orchestrator → Azure AI Search (RBAC boundary)

---

## Step 3: Apply STRIDE

For each data flow and component, apply the six STRIDE threat categories:

| STRIDE Threat | AI-Specific Manifestation | Example |
|---|---|---|
| **S**poofing | Impersonation of trusted data sources or agents | Injecting malicious documents into a RAG corpus to impersonate authoritative content |
| **T**ampering | Modification of training data, prompts, or model outputs | Indirect prompt injection via a malicious web page retrieved during RAG |
| **R**epudiation | Lack of audit trail for model decisions | No logging of prompts/responses makes it impossible to investigate a data leak |
| **I**nformation Disclosure | Exfiltration of training data, PII, or system prompts | Model trained on sensitive data leaks it through carefully crafted queries |
| **D**enial of Service | Overwhelming inference endpoints | Token flooding attacks that exhaust quota and degrade availability |
| **E**levation of Privilege | Using AI to bypass authorization | Prompt injection that causes an AI agent to call an API it should not have access to |

---

## Step 4: Apply MITRE ATLAS

[MITRE ATLAS](https://atlas.mitre.org/) catalogs adversarial techniques against ML systems. Key tactics relevant to Azure AI workloads:

| ATLAS Tactic | Relevant Techniques | Azure Mitigation |
|---|---|---|
| **ML Attack Staging** | Acquiring model artifacts, poisoning training data | Supply chain controls, dataset integrity checks |
| **Reconnaissance** | Discover ML artifacts, API probing | Rate limiting, API Management policies |
| **Initial Access** | LLM prompt injection, malicious inputs | Input validation, Azure AI Content Safety |
| **Execution** | LLM plugin/tool abuse, jailbreaking | Least-privilege tool access, content filtering |
| **Exfiltration** | Model inversion, membership inference | Differential privacy, output filtering |
| **Impact** | Model evasion, denial of ML service | Anomaly detection, quota management |

---

## Step 5: Threat Register and Risk Rating

Document findings in a threat register. For each threat:

| Field | Description |
|---|---|
| **ID** | Unique identifier (e.g., T-001) |
| **Component** | Affected system component |
| **STRIDE Category** | S / T / R / I / D / E |
| **ATLAS Technique** | ATLAS TTP ID if applicable |
| **Threat Description** | What could an attacker do? |
| **Impact** | Business/technical impact if exploited |
| **Likelihood** | Probability of exploitation (High/Med/Low) |
| **Risk Rating** | Impact × Likelihood |
| **Mitigations** | Controls that reduce or eliminate the risk |
| **Residual Risk** | Risk remaining after mitigations |
| **Owner** | Team or person responsible |

### Sample Threat Register Entry

| Field | Value |
|---|---|
| ID | T-007 |
| Component | Orchestrator → Azure AI Search |
| STRIDE | Tampering |
| ATLAS Technique | AML.T0020 - Poison Training Data |
| Threat | An attacker with write access to the document corpus injects adversarial content that manipulates RAG retrieval results |
| Impact | High — users receive misleading or harmful information |
| Likelihood | Medium |
| Risk Rating | High |
| Mitigations | Restrict blob storage write access to service principals; enable document integrity checks; log all ingestion operations |
| Residual Risk | Low |
| Owner | Platform Engineering |

---

## Threat Modeling: LLM Applications

### Key Attack Surfaces

- **System prompt** — Can it be extracted or overridden via user input?
- **User input** — Is input sanitized before passing to the model?
- **Model outputs** — Are outputs rendered safely (no XSS via HTML injection)?
- **Plugins/tools** — Does the LLM have access to tools it does not need?
- **API keys** — Are credentials stored securely and rotated regularly?

### High-Priority Threats

1. **Prompt Injection** — Attacker embeds instructions in user input to override system prompt behavior
2. **System Prompt Leakage** — User extracts confidential system prompt through adversarial queries
3. **Insecure Output Handling** — Model-generated content is executed or rendered without sanitization
4. **Excessive Agency** — Model takes destructive actions via connected tools (e.g., deleting files, sending emails)

---

## Threat Modeling: RAG Pipelines

### Key Attack Surfaces

- **Document ingestion pipeline** — Can untrusted documents be injected into the knowledge base?
- **Vector store** — Is query isolation enforced? Can one tenant read another's vectors?
- **Retrieval results** — Are retrieved documents from trusted sources only?
- **Context window assembly** — Is there a risk of context window poisoning via retrieved content?

### High-Priority Threats

1. **Indirect Prompt Injection** — Malicious instructions embedded in a retrieved document alter model behavior
2. **Data Poisoning** — Adversarial documents are ingested and skew retrieval results
3. **Cross-Tenant Data Leakage** — Insufficient index partitioning exposes data across tenants
4. **Retrieval Manipulation** — Attacker crafts queries to retrieve sensitive chunks not intended for the user

---

## Threat Modeling: AI Agents

### Key Attack Surfaces

- **Tool/API access** — What can the agent call, and is it limited to what's needed?
- **Memory/state** — Is long-term memory sanitized before use?
- **Planning output** — Can an attacker inject malicious steps into a multi-step plan?
- **Human-in-the-loop controls** — Are high-impact actions gated by human approval?

### High-Priority Threats

1. **Privilege Escalation via Tool Abuse** — Agent is manipulated into calling a privileged API it has access to but should not use in context
2. **Prompt Injection via Environment** — Agent reads a web page or file containing injected instructions
3. **Runaway Agent** — Agent enters an infinite loop or takes unauthorized actions due to missing guardrails
4. **Memory Poisoning** — Long-term agent memory is corrupted with adversarial content that affects future runs

---

## Tooling and References

| Tool | Purpose |
|---|---|
| [Microsoft Threat Modeling Tool](https://aka.ms/tmt) | STRIDE-based DFD threat modeling |
| [MITRE ATLAS Navigator](https://atlas.mitre.org/navigator/) | Visualize and track ATLAS TTPs |
| [OWASP Threat Dragon](https://owasp.org/www-project-threat-dragon/) | Open-source DFD threat modeling |
| [PyRIT](https://github.com/Azure/PyRIT) | Azure Red Teaming toolkit for AI systems |

---

## Next Steps

- [OWASP LLM Top 10 Mitigations in Azure](02-owasp-llm-top10.md)
- [Secure Azure OpenAI Architecture](03-secure-architectures/secure-azure-openai.md)
