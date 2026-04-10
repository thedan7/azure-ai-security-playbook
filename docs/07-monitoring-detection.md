# Monitoring and Detection for AI Systems

> **Audience:** Security operations engineers, cloud architects, platform engineers  
> **Azure Services:** Azure Monitor, Log Analytics, Microsoft Sentinel, Microsoft Defender for Cloud, Azure API Management, Azure OpenAI

---

## Introduction

AI systems require monitoring strategies that go beyond traditional application telemetry. In addition to infrastructure health and application performance, security teams need visibility into:

- What prompts and queries users are sending
- What actions AI agents are taking
- Whether safety filters and access controls are working
- Whether adversarial activity (extraction, injection, abuse) is occurring

This document provides a monitoring architecture and detection strategy for Azure AI workloads.

---

## Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Monitoring Architecture                          │
│                                                                         │
│  [Azure OpenAI]   [APIM]   [App Service / AKS]   [AI Search]           │
│       │              │              │                  │                │
│       └──────────────┴──────────────┴──────────────────┘                │
│                                    │                                    │
│                         [Log Analytics Workspace]                       │
│                                    │                                    │
│              ┌─────────────────────┼─────────────────────┐             │
│              │                     │                     │             │
│    [Azure Monitor             [Microsoft               [Microsoft      │
│     Alerts &                   Sentinel               Defender for     │
│     Workbooks]                (SIEM/SOAR)]             Cloud]          │
│              │                     │                     │             │
│          Operations            Security               Posture &        │
│          Team                  Operations             Compliance       │
│                                 Center                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Log Collection Strategy

### Azure OpenAI Diagnostic Logs

Enable Azure OpenAI diagnostic settings to send to a central Log Analytics workspace:

| Log Category | Contains | Retention Recommendation |
|---|---|---|
| `Audit` | Management plane operations (key rotations, deployments, config changes) | 1 year |
| `RequestResponse` | Prompt and completion content (enable with PII controls — see below) | 90 days |
| `Trace` | Detailed execution traces | 30 days |

**Important:** `RequestResponse` logs can contain sensitive user data. Before enabling:
- Apply **log data masking** to redact PII fields
- Restrict Log Analytics workspace access using Azure RBAC
- Document the legal/compliance basis for retaining prompt data

### Azure API Management Logs

APIM provides a rich gateway log that includes:
- Caller IP address and identity
- Request and response body (configurable)
- Latency and response codes
- Policy execution trace

Enable APIM diagnostic settings to Log Analytics.

### Application-Layer Logging

The AI application/orchestrator should emit structured logs for:

```json
{
  "event": "llm_request",
  "session_id": "abc123",
  "user_id": "user@example.com",
  "request_id": "req-456",
  "timestamp": "2024-01-15T10:30:00Z",
  "input_tokens": 512,
  "output_tokens": 256,
  "model": "gpt-4o",
  "deployment": "prod-gpt4o",
  "content_safety_result": "pass",
  "groundedness_score": 0.92,
  "latency_ms": 1250,
  "tool_calls": []
}
```

Avoid logging raw prompt/response content in the application layer unless required for compliance.

### Agent Action Logging

For AI agent systems, log every action taken:

```json
{
  "event": "agent_tool_call",
  "agent_id": "research-agent-v2",
  "session_id": "session-789",
  "tool_name": "search_knowledge_base",
  "parameters": { "query": "Q3 revenue", "max_results": 5 },
  "result_count": 5,
  "approved_by": null,
  "approval_required": false,
  "timestamp": "2024-01-15T10:30:05Z",
  "duration_ms": 320
}
```

---

## Key Metrics and Alerting

### Azure Monitor Alert Rules

| Metric | Threshold | Severity | Purpose |
|---|---|---|---|
| `ContentFilteredRequests` per hour | > 3× baseline | High | Possible prompt injection attack |
| `FailedRequests` rate | > 5% of total requests | Medium | Service degradation or attack |
| `TokensPerMinute` | > 85% of quota | Medium | Quota exhaustion risk |
| `Latency P95` | > 10 seconds | Medium | Performance degradation |
| APIM `unauthorized` responses | > 10 per minute | High | Credential attack or misconfiguration |
| Agent tool calls per session | > configured limit | High | Potential runaway agent or abuse |

### Azure Monitor Workbooks

Create Azure Monitor Workbooks for the following AI-specific dashboards:

1. **AI Usage Dashboard** — Requests/day, token consumption, top users, model deployment utilization
2. **Security Events Dashboard** — Content filter hits, authentication failures, prompt injection detections
3. **Agent Activity Dashboard** — Tool calls by type, approval rates, session lengths
4. **RAG Quality Dashboard** — Groundedness scores, retrieval latency, index query patterns

---

## Microsoft Sentinel Detection Rules

### Prompt Injection Detection

```kusto
// Detect repeated content filter blocks from the same user/IP
AzureDiagnostics
| where ResourceType == "OPENAI"
| where tostring(properties_s) contains "content_filter_results"
| extend FilterResult = parse_json(properties_s)
| where FilterResult.content_filter_results.hate.filtered == true
    or FilterResult.content_filter_results.sexual.filtered == true
    or FilterResult.content_filter_results.violence.filtered == true
| summarize FilterCount = count() by CallerIPAddress, bin(TimeGenerated, 1h)
| where FilterCount > 10
| project TimeGenerated, CallerIPAddress, FilterCount
```

### Model Extraction Detection

```kusto
// High-volume unique queries from a single identity — potential extraction
AzureDiagnostics
| where ResourceType == "OPENAI"
| where OperationName == "ChatCompletions_Create"
| summarize
    RequestCount = count(),
    UniquePromptCount = dcount(requestBody_s),
    AvgTokens = avg(toint(properties_s))
  by CallerIPAddress, bin(TimeGenerated, 1h)
| where RequestCount > 300 and UniquePromptCount > 250
| project TimeGenerated, CallerIPAddress, RequestCount, UniquePromptCount
```

### Unauthorized Tool Call (Agent)

```kusto
// Agent attempted to call a tool not in the allowlist
AppEvents
| where Name == "agent_tool_call"
| extend ToolName = tostring(Properties["tool_name"])
| where ToolName !in ("search_knowledge_base", "get_customer_record", "send_notification")
| project TimeGenerated, AgentId = tostring(Properties["agent_id"]),
    SessionId = tostring(Properties["session_id"]),
    UnauthorizedTool = ToolName
```

### Unusual Geographic Access

```kusto
// Azure OpenAI access from unexpected geographic locations
SigninLogs
| where AppDisplayName contains "openai" or AppDisplayName contains "cognitive"
| where Location !in ("US", "GB", "DE")  // Replace with expected countries
| project TimeGenerated, UserPrincipalName, Location, IPAddress, ResultType
| where ResultType == 0  // Successful sign-ins only
```

### Privileged AI Admin Role Activation (PIM)

```kusto
// Monitor PIM activations for AI admin roles
AuditLogs
| where OperationName == "Add member to role completed (PIM activation)"
| extend RoleName = tostring(TargetResources[0].displayName)
| where RoleName in ("Cognitive Services Contributor", "Search Service Contributor")
| project TimeGenerated, InitiatedBy = tostring(InitiatedBy.user.userPrincipalName),
    Role = RoleName, Justification = tostring(AdditionalDetails[0].value)
```

---

## Microsoft Defender for Cloud

### Enable Defender Plans for AI Resources

| Defender Plan | Protects | Key Detections |
|---|---|---|
| **Defender for App Service** | Application hosting layer | Anomalous process execution, suspicious outbound connections |
| **Defender for Containers** | AKS / Container Apps | Container escape, image vulnerability exploitation |
| **Defender for Storage** | Training data, model artifacts | Unusual access patterns, malware upload |
| **Defender for Key Vault** | Secrets, CMK | Unusual secret access, potential exfiltration |
| **Microsoft Defender for AI (preview)** | Azure OpenAI workloads | Prompt injection, jailbreak attempts, policy violations |

### Security Posture Recommendations

Review and remediate Defender for Cloud recommendations relevant to AI workloads:

- Disable public network access on Azure OpenAI resources
- Enable diagnostic logging on all AI services
- Restrict API key access in favor of managed identity authentication
- Apply network security groups to AI service subnets

---

## Incident Response for AI Systems

### AI-Specific Incident Runbooks

Develop incident response runbooks for AI-specific scenarios:

#### Runbook: Prompt Injection Attack Detected

1. **Identify** — Which user/session triggered the detection? What was the injected content?
2. **Contain** — Block the source IP in APIM or Azure Front Door; revoke user session tokens if identity is confirmed
3. **Assess** — Did the injection succeed? Were any unauthorized actions taken? Were any data returned that should not have been?
4. **Eradicate** — Update content safety rules and APIM policies to block the attack pattern
5. **Recover** — Resume normal operations; confirm no persistent changes were made
6. **Post-incident** — Document the attack pattern; update Sentinel detection rules

#### Runbook: Suspected Model Extraction

1. **Identify** — Which identity/IP is performing extraction? How many queries?
2. **Contain** — Apply emergency rate limiting in APIM; block identified IP if non-employee
3. **Assess** — Estimate how many queries have been made; determine if enough queries were made to reconstruct meaningful behavior
4. **Notify** — Escalate to legal/IP team if significant extraction likely occurred
5. **Eradicate** — Revoke API credentials for the affected identity; update detection rules
6. **Post-incident** — Review whether extraction defenses need strengthening

#### Runbook: AI Agent Runaway or Abuse

1. **Identify** — Which agent, which session, what actions were taken?
2. **Contain** — Invoke agent kill switch (disable tool permissions or terminate the container)
3. **Assess** — Review all logged agent actions; identify any unauthorized writes, sends, or deletes
4. **Remediate** — Reverse any unintended actions (API compensating calls, data restoration)
5. **Root cause** — Was this a prompt injection? A logic error? An authorization failure?
6. **Prevent** — Update tool validation logic; add missing human approval gates

---

## Compliance and Audit Logging

| Requirement | Control |
|---|---|
| SOC 2 — Logical access | Log all authentication events; alert on anomalies |
| SOC 2 — Monitoring | Continuous monitoring with defined alert thresholds |
| GDPR — Data access audit | Log all AI queries that may return personal data |
| ISO 27001 — Incident management | Defined runbooks; incidents tracked in ticketing system |
| NIST AI RMF — Monitoring | Continuous evaluation of AI system performance and safety |

---

## Monitoring and Detection Checklist

- [ ] Azure OpenAI diagnostic settings enabled and forwarded to Log Analytics
- [ ] APIM diagnostic settings enabled and forwarded to Log Analytics
- [ ] Application-layer structured logging implemented
- [ ] Agent action logging implemented for all tool calls
- [ ] Azure Monitor alerts configured for key AI security metrics
- [ ] Azure Monitor Workbooks created for AI usage and security
- [ ] Microsoft Sentinel connected to Log Analytics workspace
- [ ] Sentinel detection rules deployed for prompt injection, extraction, and agent abuse
- [ ] Defender for AI, Storage, Containers, Key Vault plans enabled
- [ ] Incident response runbooks written and tested for AI-specific scenarios
- [ ] Log retention periods set per compliance requirements
- [ ] Log access restricted to authorized security and compliance teams

---

## Next Steps

- [Responsible AI and Governance](08-responsible-ai.md)
- [Enterprise AI Security Reference Architecture](09-reference-architecture.md)
- [OWASP LLM Top 10 Mitigations](02-owasp-llm-top10.md)
