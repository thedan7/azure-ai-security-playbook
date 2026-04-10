# Enterprise AI Security Reference Architecture

Full Mermaid diagram of the enterprise AI security reference architecture.

## High-Level Zone Architecture

```mermaid
flowchart TB
    subgraph ClientZone["Zone 1: Client"]
        WebApp["Web Application"]
        MobileApp["Mobile App"]
        APIConsumer["API Consumer"]
        AgentTrigger["Agent Trigger"]
    end

    subgraph GatewayZone["Zone 2: AI Gateway"]
        AFD["Azure Front Door\n+ WAF\n(DDoS, Geo Filter)"]
        APIM["Azure API Management\n(Auth, Rate Limit,\nLogging, Routing)"]
        ContentSafety["Azure AI Content Safety\n(Prompt Shield,\nOutput Filter)"]
    end

    subgraph AIServicesZone["Zone 3: AI Services"]
        Orchestrator["AI Orchestrator\n(Container App / AKS)"]
        AzureOAI["Azure OpenAI\n(Private Endpoint)"]
        AISearch["Azure AI Search\n(Private Endpoint,\nSecurity Trimming)"]
        AgentRuntime["Agent Runtime\n(Tool Router)"]
        LogicApps["Azure Logic Apps\n(Approval Workflows)"]
        Redis["Azure Cache for Redis\n(Agent Memory)"]
    end

    subgraph DataZone["Zone 4: Data and Model"]
        BlobStorage["Azure Blob Storage\n(Documents, CMK)"]
        KeyVault["Azure Key Vault\n(Secrets, CMK, Rotation)"]
        AzureML["Azure Machine Learning\n(Model Registry)"]
        Purview["Microsoft Purview\n(Classification, DLP, Lineage)"]
        SQL["Azure SQL / Cosmos DB\n(Structured Data, CMK)"]
    end

    subgraph Security["Cross-Cutting Security and Observability"]
        Monitor["Azure Monitor\n+ Log Analytics"]
        Sentinel["Microsoft Sentinel\n(SIEM / SOAR)"]
        Defender["Microsoft Defender for Cloud"]
        EntraID["Microsoft Entra ID\n+ PIM"]
    end

    ClientZone --> AFD
    AFD --> APIM
    APIM --> ContentSafety
    ContentSafety --> Orchestrator
    Orchestrator --> AzureOAI
    Orchestrator --> AISearch
    Orchestrator --> AgentRuntime
    AgentRuntime --> LogicApps
    AgentRuntime --> Redis
    AISearch --> BlobStorage
    Orchestrator --> KeyVault
    AzureOAI --> KeyVault
    AzureML --> BlobStorage
    Purview --> BlobStorage
    Purview --> SQL

    GatewayZone --> Monitor
    AIServicesZone --> Monitor
    DataZone --> Monitor
    Monitor --> Sentinel
    Monitor --> Defender
```

## Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant EntraID as Microsoft Entra ID
    participant APIM as Azure API Management
    participant Orchestrator as AI Orchestrator
    participant OpenAI as Azure OpenAI

    User->>EntraID: Authenticate (username + MFA)
    EntraID-->>User: JWT Access Token
    User->>APIM: Request + JWT Token
    APIM->>EntraID: Validate JWT
    EntraID-->>APIM: Token valid; claims returned
    APIM->>Orchestrator: Forwarded request (user identity in header)
    Note over Orchestrator: Uses managed identity (no API key)
    Orchestrator->>OpenAI: Request + Managed Identity Token
    OpenAI-->>Orchestrator: Completion response
    Orchestrator-->>User: Response + citations
```

## Content Safety Flow

```mermaid
flowchart LR
    UserPrompt["User Prompt"] --> PromptShield["Prompt Shield\n(Azure AI Content Safety)"]
    PromptShield -- "Injection detected" --> Block["Block Request\nReturn 400"]
    PromptShield -- "Safe" --> LLM["Azure OpenAI\n(Inference)"]
    LLM --> OutputFilter["Output Moderation\n(Azure AI Content Safety)"]
    OutputFilter -- "Policy violation" --> Redact["Redact / Block\nReturn safe response"]
    OutputFilter -- "Safe" --> GroundCheck["Groundedness Check"]
    GroundCheck --> Response["Response + Citations"]
```
