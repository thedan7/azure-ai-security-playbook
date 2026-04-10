# Azure AI Security Playbook

> A community-driven technical security reference for architects and engineers building AI systems on Microsoft Azure.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## Overview

The **Azure AI Security Playbook** provides practical, prescriptive security guidance for enterprise teams deploying AI workloads on Microsoft Azure. It is designed for cloud architects, security engineers, and platform teams who need actionable patterns and controls—not just principles.

The playbook covers the full lifecycle of AI security: from threat modeling and identity design, through data governance and model protection, to monitoring and responsible AI governance.

---

## Who Is This For?

| Role | How to Use This Playbook |
|---|---|
| **Cloud Architects** | Use the reference architectures and secure design patterns |
| **Security Engineers** | Map controls to OWASP LLM Top 10 and Azure-native detections |
| **Platform / MLOps Teams** | Apply identity, data governance, and monitoring guidance |
| **Compliance Officers** | Reference the responsible AI and governance section |

---

## Contents

| # | Section | Description |
|---|---|---|
| 1 | [Threat Modeling AI Systems](docs/01-threat-modeling.md) | How to threat model LLM apps, RAG pipelines, and AI agents |
| 2 | [OWASP LLM Top 10 Mitigations](docs/02-owasp-llm-top10.md) | Azure controls mapped to each OWASP LLM Top 10 risk |
| 3 | Secure AI Architectures | |
| | &nbsp;&nbsp;[Secure Azure OpenAI Architecture](docs/03-secure-architectures/secure-azure-openai.md) | Network isolation, access controls, and monitoring for Azure OpenAI |
| | &nbsp;&nbsp;[Secure RAG Architecture](docs/03-secure-architectures/secure-rag-architecture.md) | End-to-end security for Retrieval-Augmented Generation pipelines |
| | &nbsp;&nbsp;[Secure AI Agent Architecture](docs/03-secure-architectures/secure-ai-agent-architecture.md) | Security patterns for autonomous and agentic AI systems |
| 4 | [Identity and Access for AI Systems](docs/04-identity-access.md) | Entra ID, managed identities, and least-privilege design |
| 5 | [AI Data Governance and Protection](docs/05-data-governance.md) | Data classification, leakage prevention, and Microsoft Purview |
| 6 | [Model Security](docs/06-model-security.md) | Model poisoning, supply chain risks, and extraction attacks |
| 7 | [Monitoring and Detection](docs/07-monitoring-detection.md) | Azure Monitor, Sentinel, and Defender for Cloud for AI workloads |
| 8 | [Responsible AI and Governance](docs/08-responsible-ai.md) | Governance frameworks, fairness, and accountability |
| 9 | [Enterprise AI Security Reference Architecture](docs/09-reference-architecture.md) | Complete secure enterprise AI architecture |

---

## Quick Start

1. **New to AI security?** Start with [Threat Modeling AI Systems](docs/01-threat-modeling.md) to build a mental model of the attack surface.
2. **Addressing compliance gaps?** Jump to [OWASP LLM Top 10 Mitigations](docs/02-owasp-llm-top10.md) for control mappings.
3. **Designing a new system?** Choose the relevant architecture guide under [Secure AI Architectures](docs/03-secure-architectures/).
4. **Want the full picture?** Read the [Enterprise AI Security Reference Architecture](docs/09-reference-architecture.md).

---

## Architecture Diagrams

Architecture diagrams for this playbook are maintained in the [`diagrams/`](diagrams/) folder as text-based (Mermaid / ASCII) diagrams to support version control and community contributions.

---

## Contributing

We welcome contributions from the community! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to submit new content, fix errors, or improve existing sections.

---

## License

This project is licensed under the [MIT License](LICENSE). Content is provided as-is for educational and reference purposes.

---

## Acknowledgements

This playbook references and builds upon:

- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Microsoft Azure Security Benchmarks](https://learn.microsoft.com/en-us/security/benchmark/azure/)
- [Microsoft Responsible AI Standard](https://www.microsoft.com/en-us/ai/responsible-ai)
- [NIST AI Risk Management Framework](https://www.nist.gov/system/files/documents/2023/01/26/AI%20RMF%201.0.pdf)
- [MITRE ATLAS](https://atlas.mitre.org/)
