# Model Security

> **Audience:** AI/ML engineers, security engineers, cloud architects  
> **Azure Services:** Azure Machine Learning, Azure Container Registry, Azure Artifacts, Microsoft Defender for Containers, Azure Key Vault, GitHub Advanced Security

---

## Introduction

AI models are software artifacts that carry unique security risks not present in traditional software. They can be poisoned during training, extracted through repeated inference, stolen via supply chain attacks, or manipulated to behave incorrectly on specific inputs. This document covers the key model security risks and the Azure controls that address them.

---

## Model Security Risk Overview

| Risk | Description | Impact |
|---|---|---|
| **Training data poisoning** | Malicious data introduced during training creates backdoors or degrades performance | Model behaves incorrectly on specific trigger inputs; safety guardrails bypassed |
| **Model poisoning (fine-tune attacks)** | Adversarial fine-tuning introduces malicious behaviors | Backdoored model deployed to production |
| **Model extraction** | Systematic API queries reconstruct model weights or behavior | Intellectual property theft; enables offline adversarial testing |
| **Model inversion** | Query patterns extract training data from the model | PII or confidential data from training set exposed |
| **Adversarial evasion** | Crafted inputs cause misclassification or unexpected model behavior | Safety filters bypassed; content moderation defeated |
| **Supply chain compromise** | Malicious code or weights introduced via third-party models or libraries | Backdoor in production model; data exfiltration |
| **Model artifact tampering** | Saved model weights modified after training | Behavior changed without detection |

---

## Training Data Poisoning

### Attack Description

An attacker with write access to training data sources introduces adversarial examples that:
- Create backdoors (trigger inputs that cause specific misbehavior)
- Degrade overall model performance
- Introduce biases that cause discriminatory outputs
- Bypass safety fine-tuning

### Mitigations

#### Data Source Controls
- Enforce strict write access controls on all training data stores (Azure RBAC, minimum `Storage Blob Data Contributor` to approved identities only)
- Enable **Azure Blob Storage immutability policies** on finalized training datasets — once a dataset is approved and locked, no modifications are possible
- Maintain **versioned copies** of all training datasets in Azure Machine Learning; never overwrite

#### Data Quality Scanning
- Implement automated statistical checks before training:
  - Distribution shift detection between dataset versions
  - Outlier detection on label distributions
  - Near-duplicate detection (potential amplification attacks)
- Use **Azure Databricks** or AML pipelines to run data quality checks as a gate before training jobs start

#### Human Review Gates
- Require data steward sign-off before any new dataset version enters training
- Maintain a signed audit trail (AML experiment logging) linking dataset versions to training runs

---

## Supply Chain Security

### Attack Description

Attackers compromise a dependency in the AI supply chain:
- Malicious Python packages on PyPI mimicking legitimate ML libraries
- Compromised model weights distributed via public hubs (Hugging Face, etc.)
- Backdoored container images used as training environments

### Mitigations

#### Package Management
- Use **Azure Artifacts private feeds** as the sole package source in production environments; mirror approved packages from public PyPI
- Enforce Artifacts feed usage via `pip.conf` or `pyproject.toml` in all training and serving environments
- Enable **GitHub Advanced Security Dependabot** for automated vulnerability alerts on Python dependencies
- Pin all package versions in `requirements.txt` or `pyproject.toml`; do not use unpinned ranges in production

#### Container Image Security
- Store all training and serving container images in **Azure Container Registry (ACR)**
- Enable **Microsoft Defender for Container Registries** to scan images for CVEs on push and on schedule
- Sign all production images using **Azure Key Vault + Notary** signing
- Enforce **Azure Policy** to require ACR images for all AKS/ACI deployments (deny images from public registries)

```yaml
# Example: Azure Container Registry security settings
properties:
  adminUserEnabled: false           # Disable admin user; use managed identity
  publicNetworkAccess: Disabled     # Private endpoint only
  networkRuleSet:
    defaultAction: Deny
  policies:
    quarantinePolicy:
      status: enabled               # Quarantine images pending vulnerability scan
    retentionPolicy:
      days: 30
      status: enabled
```

#### Foundation Model Sourcing
- Use the **Azure AI Foundry model catalog** as the approved source for foundation models
- Verify model checksums/hashes against official publisher values before use
- Document the provenance of all models in use (source, version, hash) in a model registry

---

## Model Extraction Defense

### Attack Description

An adversary makes large numbers of carefully crafted queries to the model API to reconstruct its behavior (a "model extraction" or "model stealing" attack), creating a local copy they can use for:
- Offline adversarial testing (finding inputs that bypass safety filters)
- Intellectual property theft
- Evading detection by using the extracted model instead of the production API

### Mitigations

| Control | Implementation |
|---|---|
| **Rate limiting** | Apply per-user and per-application token and request quotas in Azure API Management |
| **Anomaly detection** | Use Microsoft Sentinel to detect systematic high-volume query patterns characteristic of extraction |
| **Authentication enforcement** | Require Entra ID tokens for all API access; no anonymous queries |
| **Output perturbation** | For proprietary models, introduce controlled noise in outputs (reduces extraction fidelity) |
| **Private endpoints** | Deploy Azure OpenAI behind private endpoints to prevent access from outside authorized networks |
| **Audit logging** | Log all inference requests with caller identity for post-incident investigation |

#### Sentinel Detection Rule (KQL)

```kusto
// Detect potential model extraction: high query volume from single identity
AzureDiagnostics
| where ResourceType == "OPENAI"
| where OperationName == "ChatCompletions_Create"
| summarize RequestCount = count(), UniquePrompts = dcount(requestBody_s) by CallerIPAddress, bin(TimeGenerated, 1h)
| where RequestCount > 500 and UniquePrompts > 400
| project TimeGenerated, CallerIPAddress, RequestCount, UniquePrompts
```

---

## Model Inversion and Membership Inference

### Attack Description

- **Model inversion:** Adversary queries the model to reconstruct training data (especially effective when the model was fine-tuned on small, sensitive datasets)
- **Membership inference:** Adversary determines whether a specific record was in the training dataset

### Mitigations

- **Data minimization at training time:** Fine-tune only on the minimum data required; do not include PII unless strictly necessary
- **Differential privacy:** Apply differential privacy during fine-tuning (available in Azure ML via `smartnoise-sdk` or custom DP-SGD implementations) to limit information leakage about individual training examples
- **Output restriction:** Limit the precision of model outputs (e.g., return top-3 categories instead of full probability distributions)
- **Monitoring:** Alert on queries that appear designed to probe model knowledge about specific individuals

---

## Model Artifact Integrity

### Attack Description

Model weights, checkpoints, or configuration files stored in Azure are tampered with post-training, introducing malicious behavior without triggering any CI/CD checks.

### Mitigations

#### Artifact Signing and Verification
- Generate a cryptographic hash (SHA-256) of all model artifacts after training
- Store hashes in **Azure Key Vault** or as a signed attestation
- Verify hashes before loading any model artifact in the inference serving pipeline
- Use **Azure Machine Learning model registration** with versioning and immutability settings

```python
import hashlib

def compute_model_hash(model_path: str) -> str:
    sha256 = hashlib.sha256()
    with open(model_path, "rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            sha256.update(chunk)
    return sha256.hexdigest()

# At serving time
expected_hash = get_approved_hash_from_keyvault(model_name, model_version)
actual_hash = compute_model_hash(loaded_model_path)
if actual_hash != expected_hash:
    raise SecurityError("Model artifact integrity check failed — hash mismatch")
```

#### Storage Controls
- Store production model artifacts in Azure Blob Storage with **immutability policies** (WORM — Write Once Read Many)
- Restrict write access to model artifact storage to the training pipeline identity only
- Enable **Azure Storage versioning** to preserve history and detect unauthorized modifications
- Enable **Azure Defender for Storage** to alert on unexpected access or modifications

---

## Model Registry and Governance

Use **Azure Machine Learning model registry** to track the full lifecycle of all models:

| Stage | Control |
|---|---|
| **Registration** | Record training dataset version, training run ID, evaluation metrics, and approver |
| **Staging** | Run automated security and quality evaluation before promotion to production |
| **Production** | Require approval workflow; sign off by AI security reviewer |
| **Deprecation** | Document reason; retain artifacts for audit; revoke serving permissions |

### Model Card Requirements

For each production model, maintain a model card documenting:
- Intended use and out-of-scope uses
- Training data sources and known limitations
- Fairness evaluation results
- Security evaluation results (red team findings)
- Data handling and privacy considerations

---

## Adversarial Robustness Testing

Before deploying a model to production, conduct adversarial robustness testing:

| Test Type | Tool / Approach | Purpose |
|---|---|---|
| **Red teaming** | [PyRIT](https://github.com/Azure/PyRIT) (Azure Python Risk Identification Toolkit) | Automated adversarial probing of LLMs |
| **Prompt injection testing** | Manual + automated test cases | Verify injection mitigations hold |
| **Jailbreak testing** | Curated jailbreak prompt dataset | Verify safety training is effective |
| **Data extraction probing** | Membership inference attacks | Verify training data is not memorized |
| **Content filter bypass** | Adversarial rephrasing, encoding tricks | Verify content safety filters are robust |

Document all findings, remediations, and residual risk in the model security assessment.

---

## Model Security Checklist

- [ ] Training datasets locked with Azure Blob immutability policies after approval
- [ ] Statistical data quality checks run before each training job
- [ ] Production Python packages sourced from Azure Artifacts private feed
- [ ] Container images scanned by Defender for Container Registries
- [ ] Production images signed and signature enforced by Azure Policy
- [ ] Foundation models sourced from Azure AI Foundry catalog with hash verification
- [ ] Model artifacts stored with immutability policies; write access restricted
- [ ] Model artifact hash verification implemented in serving pipeline
- [ ] Model registry entries include dataset version, training run, and approver
- [ ] Rate limiting applied to inference API to defend against extraction
- [ ] Sentinel detection rules configured for extraction patterns
- [ ] Red team testing performed before production deployment
- [ ] Model cards created and maintained for all production models

---

## Next Steps

- [Monitoring and Detection](07-monitoring-detection.md)
- [AI Data Governance and Protection](05-data-governance.md)
- [Threat Modeling AI Systems](01-threat-modeling.md)
